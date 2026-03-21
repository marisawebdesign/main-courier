# C-04: Cart & Checkout

> **Spec Version:** 1.0
> **Status:** Complete — Ready for Development
> **Dependencies:** C-01 (authenticated session, customer record), C-03 (cart items from catalog or shopping list), C-06 (batch context and cutoff timing), Stripe API, PostgreSQL
> **Relates to:** C-03 (items flow into cart), C-06 (batch cutoff drives early auth and auto-checkout), C-07 (Delivery Tracking — order status after checkout), D-02 (Driver submits receipt → triggers capture), S-01 (Store Dashboard — receives order, confirms fulfillment)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md. Payment data is NEVER stored locally — Stripe holds all card data. The API only stores Stripe customer IDs, PaymentIntent IDs, and PaymentMethod references. No card numbers ever touch the server.

---

## Overview

C-04 covers the cart review page, checkout flow, payment processing, delivery options, and post-fulfillment capture. Every order uses Stripe's auth/capture model — a hold is placed on the customer's card, and the actual charge is captured after fulfillment. No immediate charges.

The system supports two payment methods: card (via Stripe, default) and cash (for verified customers only, admin-approved). All deliveries are contact (face-to-face handoff) by default, with no-contact delivery unlockable after the customer builds a reliability track record.

### Design Principles

1. **Auth/capture everywhere.** No immediate charges for card orders. Every order authorizes a hold and captures later. Buffer percentages vary by order type.
2. **Early auth for group orders.** Batch route participants' cards are authorized at the batch cutoff — not on delivery day. A card decline discovered on delivery day is too late.
3. **Minimal payment surface.** Stripe handles all card data via Stripe Elements. The API stores only Stripe IDs.
4. **Cash is a privilege.** Cash payment requires internal verification by admin. Not available by default.
5. **Contact delivery is default.** Every delivery requires face-to-face handoff until the customer earns no-contact access through a reliability track record.
6. **Idempotent everything.** Every Stripe API call uses an idempotency key. Network failures cannot create duplicate charges.
7. **Immediate feedback, manual fallback.** Payment errors surface instantly. Edge cases are flagged for manual admin review — no background retry queues.

---

## Payment Model

### Three Auth Buffer Tiers

| Order Type | Buffer | Rationale | Config Key |
|---|---|---|---|
| Partnered store (solo order) | 5% | Prices are from the catalog. Buffer covers minor stale pricing, weight-based adjustments. | `partnered_auth_buffer_pct` |
| Batch group order (unpartnered) | 10% | Prices are customer estimates. Auth happens early (at cutoff) so there's time to handle failures before delivery day. | `batch_auth_buffer_pct` |
| On-demand solo (unpartnered) | 20% | Prices are customer estimates. Auth happens at checkout with no time buffer before shopping. | `ondemand_auth_buffer_pct` |

All buffer percentages are stored in `system_config` and adjustable through admin.

### Auth/Capture Flow

```
Checkout (or batch cutoff for group orders)
  │
  ▼
Stripe PaymentIntent created
  capture_method: 'manual'
  amount: order_total × (1 + buffer_pct)
  │
  ▼
Auth hold placed on customer's card
Order created with status: 'authorized'
  │
  ▼
[Driver shops or store fulfills]
  │
  ▼
Driver/store submits actual total
  │
  ├── Final ≤ authorized → Capture final amount, release difference
  │
  └── Final > authorized (rare) → Capture full auth, flag for admin review
```

### Early Auth for Group/Batch Orders

For unpartnered store batch routes, the auth timing is different from solo orders:

**Solo orders (partnered or on-demand):** Auth happens at checkout when the customer taps "Place Order."

**Batch group orders:** Auth happens at the batch cutoff, NOT when the customer submits their list. The flow is:

```
Customer submits shopping list → batch_participants.status = 'list_submitted'
  (no payment yet — just a commitment)

Batch cutoff arrives (e.g., Wed 8pm)
  │
  ▼
System runs batch confirmation job:
  For each participant with status 'list_submitted':
    1. Calculate fees + auth hold amount
    2. Attempt Stripe auth on saved card
    3. If auth succeeds:
       - Create order record (status = 'authorized')
       - Update participant status to 'checked_out'
       - SMS: "Your Walmart order is confirmed. Hold: $XX.XX"
    4. If auth fails:
       - SMS: "Card declined for your Walmart order. Fix by [deadline] 
         or you'll be removed from the route."
       - Give 2-hour grace window
       - If still failing after grace: remove from batch, notify

All cards must be authorized before the driver starts shopping.
```

**Why this matters:** If you auth at list submission time (days before delivery), the auth could expire or the card status could change by delivery day. If you auth on delivery day, a decline means the driver has already started shopping. The cutoff is the sweet spot — it's close enough to delivery that the auth won't expire, but early enough to handle failures before the driver starts.

**The 2-hour grace window:** When a card declines at the batch cutoff, the customer gets 2 hours to add a different card or add funds. If they fix it, the system re-attempts the auth. If they don't, they're removed from the batch, the consolidated shopping list is regenerated without their items, and the batch fee is recalculated for remaining participants.

### Cash Payment

Cash payment is available only to customers who have been internally verified by admin. It is not available by default and cannot be self-selected.

**Verification process:**

1. Customer requests cash payment ability (from their Account & Settings page, or by contacting your partner directly)
2. Admin reviews the customer: do they have a history of successful orders? Are they a known community member? Does your partner vouch for them?
3. Admin toggles `cash_verified = true` on the customer record through the admin dashboard
4. Customer now sees "Cash" as a payment option at checkout

**Cash order flow:**

```
Checkout with cash selected
  │
  ▼
No Stripe auth — order created with payment_method = 'cash'
Order status: 'pending_cash'
  │
  ▼
Driver delivers order
Customer pays driver in cash
  │
  ▼
Driver confirms cash collected in their app
  Enters amount received
  │
  ▼
Order status: 'cash_collected'
System records the cash amount on the order
```

**Cash order protections:**

- Cash orders are limited to a configurable maximum amount (`cash_order_max`, default $100). Larger orders require card payment.
- Cash customers can be un-verified at any time by admin if they have issues (no-shows, short payments, etc.)
- The driver app shows a prominent indicator that an order is cash: "💵 CASH — Collect $XX.XX"
- Cash orders still have all fees calculated (delivery, service, shopping). The driver collects the full total.
- No tips on cash orders (too complex to handle cash tips in the system — tips are a card-only feature)
- Cash orders cannot use auto-checkout (there's no card to charge)

**Reconciliation:** Cash collected by the driver is tracked per-delivery in the system. Your partner reconciles cash at the end of each delivery day. The admin dashboard shows a cash reconciliation view: total cash orders, expected cash collection, actual cash collected (from driver input), and any discrepancies.

---

## Delivery Options

### Contact Delivery (Default)

All deliveries are contact by default. The driver must hand the order to the customer (or a household member) at the door. The driver confirms delivery in their app, which includes:

- Marking the order as `delivered`
- The customer's name and address are shown to verify
- An optional delivery photo (for the driver's records, not required)

If the customer is not home, the driver calls/texts the customer's phone number (from the order record). If no response within 10 minutes, the driver follows the undeliverable protocol:

```
1. Attempt phone call
2. Wait 5 minutes
3. Attempt text message
4. Wait 5 minutes
5. If still no response:
   - Mark order as 'delivery_attempted'
   - Return items to vehicle
   - Admin notified for follow-up
   - Rescheduling handled manually by admin
```

### No-Contact Delivery (Earned)

No-contact delivery lets the driver leave the order at the door (or a specified safe spot) without waiting for the customer. This is a convenience feature that must be earned through a reliability track record.

**Unlock criteria:**

A customer unlocks no-contact delivery when ALL of the following are true:
- `successful_contact_deliveries >= no_contact_unlock_threshold` (configurable, default: 5)
- No missed deliveries (customer wasn't home) in their history
- No disputes or chargebacks on any order
- Account age >= 30 days

**How it works once unlocked:**

At checkout, the customer sees a delivery method toggle:

```
── Delivery Method ─────────────────────
○ 🤝 Contact — hand to me at the door
○ 📦 No-contact — leave at my door
  Drop-off spot (optional):
  [e.g., "side porch, under the bench"]
```

If they select no-contact, they can specify a drop-off location. The driver:
1. Places the order at the specified spot (or door if no spot specified)
2. Takes a delivery photo (required for no-contact)
3. Sends an SMS: "Your order has been delivered! [photo link]"
4. Marks order as `delivered` with the photo attached

**Losing no-contact access:**

If a customer reports a missing or damaged no-contact delivery, OR if they file a dispute/chargeback, no-contact access is revoked and they revert to contact-only. Admin can also manually revoke. The customer would need to rebuild their track record to regain it.

**Tracking delivery reliability:**

The customer record tracks:
```sql
successful_contact_deliveries  INTEGER DEFAULT 0
failed_deliveries              INTEGER DEFAULT 0  -- not home, dispute, etc.
no_contact_unlocked            BOOLEAN DEFAULT false
no_contact_unlocked_at         TIMESTAMPTZ
no_contact_revoked             BOOLEAN DEFAULT false
```

These fields are updated after each delivery by the order completion flow.

---

## Cart Page

### Cart Page — Partnered Store

```
┌─────────────────────────────────────────┐
│  ← Back to [Store Name]                │
│                                         │
│  Your Order — Calais IGA                │
│  Wed delivery · Mar 25                  │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ [img] Sourdough Bread           │   │
│  │       $4.50          [ - 1 + ]  │   │
│  ├─────────────────────────────────┤   │
│  │ [img] Apple Pie                 │   │
│  │       $12.00         [ - 2 + ]  │   │
│  ├─────────────────────────────────┤   │
│  │ [img] Honey 16oz               │   │
│  │       $8.00          [ - 1 + ]  │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Subtotal:                     $36.50   │
│  Delivery fee:                  $4.99   │
│  Service fee (5%):              $1.83   │
│  ────────────────────────────────────   │
│  Total:                        $43.32   │
│                                         │
│  A hold of $45.49 will be placed on     │
│  your card. You're charged the actual   │
│  amount after fulfillment. Any          │
│  difference is released to your card.   │
│                                         │
│  [Checkout →]                           │
│                                         │
│  [Clear Cart]                           │
└─────────────────────────────────────────┘
```

### Cart Page — Unpartnered Store (Batch Route)

```
┌─────────────────────────────────────────┐
│  ← Back to Shopping List                │
│                                         │
│  Your Order — Walmart                   │
│  Thu batch route · Mar 26               │
│  ⏰ Cutoff: Wed 8pm (1d 6h left)       │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 2% Milk, 1 gallon          x1   │   │
│  │ Sub: Store brand ok    ~$3.50   │   │
│  ├─────────────────────────────────┤   │
│  │ Cheerios, family size       x1   │   │
│  │ Sub: Similar item ok   ~$5.99   │   │
│  ├─────────────────────────────────┤   │
│  │ Ground beef, 1 lb           x2   │   │
│  │ Sub: None             ~$9.98    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Est. items:                   ~$19.47  │
│  Shopping fee:                  $3.00   │
│  Delivery fee:                  $3.99   │
│  Service fee (5%):             ~$0.97   │
│  ────────────────────────────────────   │
│  Est. total:                  ~$27.43   │
│                                         │
│  Your card will be authorized at the    │
│  batch cutoff (Wed 8pm) for ~$30.17     │
│  to cover price differences. Charged    │
│  the actual amount after shopping.      │
│                                         │
│  [Checkout →]                           │
│                                         │
│  [Edit Shopping List]                   │
└─────────────────────────────────────────┘
```

Note: for batch orders, the auth hold messaging says "authorized at the batch cutoff" — not immediately. The customer understands their card won't be charged right now, but will be held at cutoff time.

### Cart Item Rows

**Partnered store items:** Product photo, name, unit price, quantity stepper. Tapping the row returns to product detail. Decrementing to 0 removes the item.

**Unpartnered store items:** Description text (no photo), quantity, substitution abbreviation, estimated price with `~`. Not editable from cart — "Edit Shopping List" returns to C-03.

### Fee Calculation

All fees are calculated server-side by `GET /api/cart`. The frontend never computes fees.

| Fee | Calculation | Source |
|---|---|---|
| Subtotal (partnered) | Sum of `unit_price × quantity` | `products` table |
| Est. items (unpartnered) | Sum of `estimated_price × quantity` | Customer estimates |
| Shopping fee | `base_shopping_fee / participant_count`, floored at `min_shopping_fee` | `shopping_fee_config` + `weekly_batches` |
| Delivery fee | Tiered by order value and delivery type | `fee_tiers` |
| Service fee | `service_fee_pct × subtotal`, capped at `service_fee_cap` | `service_fee_config` |
| Tip | Customer-selected | Checkout input |
| Auth buffer | `total × buffer_pct` | `system_config` |

### Empty Cart

```
Your cart is empty.
Browse stores near you to start an order.
[← Back to Home]
```

---

## Checkout Page

### Layout

```
┌─────────────────────────────────────────┐
│  ← Back to Cart                         │
│                                         │
│  Checkout — [Store Name]                │
│                                         │
│  ── Payment ──────────────────────────  │
│                                         │
│  ○ 💳 Visa ending 4242        [Change]  │
│  ○ 💵 Cash                              │
│    (only if cash_verified = true)       │
│                                         │
│  (or if no saved card:)                 │
│  [Card number                        ]  │
│  [MM/YY]  [CVC]  [ZIP]                 │
│  ☐ Save this card for future orders     │
│                                         │
│  ── Delivery ─────────────────────────  │
│                                         │
│  📍 42 Route 1, Weston, ME     [Change] │
│                                         │
│  ○ 🚚 Door delivery                     │
│  ○ 📦 Hub pickup — $2.00 less          │
│    (if hubs available)                  │
│                                         │
│  ── Delivery Method ─────────────────  │
│  (only if no_contact_unlocked = true)   │
│                                         │
│  ○ 🤝 Contact — hand to me             │
│  ○ 📦 No-contact — leave at door       │
│    Drop-off spot: [________________]   │
│                                         │
│  ── Tip Your Driver ─────────────────  │
│  (card payment only)                    │
│                                         │
│  [$0]  [$3]  [$5]  [$10]  [Custom]     │
│                                         │
│  ── Auto-Checkout ────────────────────  │
│  (batch orders, card payment only)      │
│                                         │
│  ☐ Auto-checkout before batch cutoff    │
│    If I forget, charge this card        │
│    before the deadline.                 │
│                                         │
│  ── Order Summary ────────────────────  │
│                                         │
│  Subtotal:                     $36.50   │
│  Delivery fee:                  $4.99   │
│  Service fee:                   $1.83   │
│  Tip:                           $5.00   │
│  ────────────────────────────────────   │
│  Total:                        $48.32   │
│  Auth hold:                    $50.74   │
│                                         │
│  [Place Order — Hold $50.74 →]         │
│  (or for cash: [Place Order — $48.32]) │
│                                         │
└─────────────────────────────────────────┘
```

### Payment Section

**Card payment (default):**

If the customer has a saved Stripe PaymentMethod, it's pre-selected: "💳 [Brand] ending [last4]". "Change" opens saved cards or add-new-card flow.

New cards use Stripe Elements embedded inline. Fields: card number, expiry, CVC, billing ZIP. "Save this card" checkbox defaults to checked for first-time card users.

**Cash payment (verified customers only):**

Only shown if `customers.cash_verified = true`. When selected:
- The tip section hides (no tips on cash orders)
- The auto-checkout section hides
- The auth hold line disappears
- The CTA changes to "Place Order — $XX.XX" (no hold language)
- A note appears: "Pay your driver $XX.XX in cash at delivery."

**Switching between card and cash:** If the customer has both options, tapping the radio buttons switches the checkout page sections and CTA accordingly.

### Delivery Section

**Address:** Customer's profile address, pre-filled. "Change" opens inline address editor with geocoding.

**Delivery type:**
- Door delivery (default) — standard fee from `fee_tiers`
- Hub pickup — reduced fee. Hub selector shown if multiple hubs available.

**Delivery method (contact vs. no-contact):**

Only shown if `no_contact_unlocked = true` on the customer record. If the customer hasn't unlocked no-contact, this section doesn't render — all deliveries are contact by default with no option shown.

When shown:
- Contact (default): driver hands order to customer
- No-contact: driver leaves at door. Optional "drop-off spot" text field (200 chars). Driver takes a delivery photo (required).

### Tip Section

Pre-set buttons: $0, $3, $5, $10, Custom. Default: $0 (no pre-selection). Tips are card-only — section is hidden when cash payment is selected.

Tip is stored separately on the order record and paid out to the driver.

### Auto-Checkout Section

Only shown for batch route orders with card payment. Hidden for partnered store solo orders and cash orders.

When enabled, if the customer doesn't complete checkout before the batch cutoff, the system automatically authorizes their saved card 1 hour before cutoff.

Preference stored on customer record, remembered across orders.

### CTA Button

The button text always shows the amount:
- Card: "Place Order — Hold $50.74 →"
- Cash: "Place Order — $48.32 →"
- Batch (before cutoff): "Submit Order — Hold at cutoff →" (auth happens later)

---

## Place Order Flow

### Card Payment — Solo Order (Partnered or On-Demand)

```
Customer taps "Place Order"
  │
  ▼
1. Client-side validation
   - Payment method selected
   - Delivery address set
   - All required fields filled
  │
  ▼
2. POST /api/orders
  │
  ▼
3. Server-side processing:
   a. Validate session, customer, store, cart
   b. Calculate all fees server-side
   c. Determine buffer: partnered = 5%, on-demand = 20%
   d. Auth hold = total × (1 + buffer)
   e. Create/attach Stripe Customer + PaymentMethod if saveCard
   f. Create Stripe PaymentIntent (capture_method: manual, confirm: true)
      idempotencyKey: 'order-auth-{orderId}'
   g. PaymentIntent succeeds:
      - Create order record (status: 'authorized')
      - Create order_items from cart_items
      - Clear cart
      - Return success
   h. PaymentIntent fails:
      - Return decline reason
      - Cart preserved for retry
  │
  ▼
4. Confirmation screen
```

### Card Payment — Batch Group Order

```
Customer taps "Submit Order"
  │
  ▼
1. POST /api/orders (with batchId)
  │
  ▼
2. Server-side processing:
   a. Validate session, customer, store, batch participation
   b. Calculate fees with 10% batch buffer
   c. Create order record (status: 'pending_auth')
      - NOT authorized yet — auth happens at cutoff
   d. Create order_items from shopping list
   e. Store payment method on order for later auth
   f. Update batch_participants.status to 'order_placed'
   g. Return success with note: "Card will be authorized at cutoff"
  │
  ▼
3. Confirmation screen:
   "Order submitted! Your card will be held for ~$XX.XX 
    at the batch cutoff (Wed 8pm)."
  │
  ▼
4. [At batch cutoff — system runs batch auth job]
   See "Batch Cutoff Auth Job" section below
```

### Cash Payment

```
Customer taps "Place Order" (cash)
  │
  ▼
1. POST /api/orders (paymentMethod: 'cash')
  │
  ▼
2. Server-side processing:
   a. Validate customer.cash_verified = true
   b. Validate total <= cash_order_max
   c. Calculate fees (no buffer, no auth)
   d. Create order record (status: 'pending_cash')
   e. Create order_items from cart
   f. Clear cart
   g. Return success
  │
  ▼
3. Confirmation screen:
   "Order placed! Pay $XX.XX cash to your driver at delivery."
```

---

## Batch Cutoff Auth Job

Runs at each batch's `order_cutoff` time. This is the critical job that ensures all cards are authorized before the driver starts shopping.

### Processing

```
For each weekly_batch WHERE cutoff just passed AND status = 'confirmed':

  Phase 1: Auth all pending orders
  ─────────────────────────────────
  For each order WHERE batch_participant_id IN this batch
    AND status = 'pending_auth':

    1. Recalculate fees (participant count may have changed)
    2. Auth hold = total × (1 + 0.10)
    3. Create Stripe PaymentIntent (manual capture)
       idempotencyKey: 'batch-auth-{orderId}'
    4. If auth succeeds:
       - Update order status to 'authorized'
       - SMS: "Your Walmart order is confirmed. Hold: $XX.XX.
         Delivery Thursday."
    5. If auth fails:
       - Update order status to 'auth_failed'
       - SMS: "Card declined for your Walmart order. Update your 
         card by [cutoff + 2 hours] or you'll be removed."
       - Set grace_deadline on the order

  Phase 2: Handle auto-checkout participants
  ───────────────────────────────────────────
  For each batch_participant WHERE status = 'list_submitted'
    AND customer.auto_checkout_enabled = true
    AND no order exists:

    1. Create order from shopping list
    2. Auth using saved card (same as Phase 1)
    3. If succeeds: create order, notify
    4. If fails: notify, give grace window

  Phase 3: Grace window expiry (runs cutoff + 2 hours)
  ─────────────────────────────────────────────────────
  For each order WHERE status = 'auth_failed'
    AND grace_deadline has passed:

    1. Remove participant from batch
    2. Recalculate fees for remaining participants
    3. Regenerate consolidated shopping list
    4. SMS to removed customer: "Your Walmart order has been 
       removed. Your shopping list is saved for next week."
    5. If removing this participant drops below min_participants:
       - Flag for admin review (don't auto-cancel the batch —
         admin decides)

  Phase 4: Skip uncommitted participants
  ───────────────────────────────────────
  For each batch_participant WHERE status IN ('committed', 'list_submitted')
    AND no order AND auto_checkout not enabled:

    1. Set status to 'skipped'
    2. SMS: "Your Walmart list wasn't checked out. Saved for next week."
```

### Key Rule

**The driver NEVER starts shopping until Phase 1 is complete and all remaining orders are in `authorized` status.** The driver app checks this before enabling the "Start Shopping" button.

---

## Capture Flow (After Fulfillment)

Triggered when the driver submits the receipt (unpartnered) or the store confirms fulfillment (partnered).

### Processing

```
POST /api/orders/:orderId/capture

1. Load order + original PaymentIntent
2. Calculate final amount:
   - Actual items total (from driver receipt or store fulfillment)
   - Shopping fee (unchanged)
   - Delivery fee (unchanged)
   - Service fee (recalculated on actual items total)
   - Tip (unchanged)
3. Compare final to authorized:
   a. Final ≤ authorized:
      - Capture for final amount
      - Remaining hold released automatically by Stripe
      - SMS: "Your [Store] order: Charged $XX.XX 
        (held $YY.YY — $ZZ.ZZ released)"
   b. Final > authorized (rare):
      - Capture for full authorized amount
      - Flag order: 'capture_overage'
      - Admin reviews and decides
4. Update order record with actual amounts
5. Update customer delivery stats (for no-contact unlock tracking)
```

### Cash Order Completion

```
Driver marks order as delivered + cash collected
  │
  ▼
1. Driver enters cash amount received
2. If amount matches expected: order status = 'cash_collected'
3. If amount doesn't match:
   - Flag for admin review
   - Record both expected and received amounts
4. No Stripe interaction — pure record-keeping
```

---

## Post-Delivery

### Tip Adjustment (Card Orders Only)

Within 24 hours of delivery, the customer can increase their tip from the order detail page.

Tip increases are a separate Stripe PaymentIntent (immediate capture, not auth/capture) charged to the customer's saved card.

Tip decreases are not allowed.

### Delivery Confirmation and Stats Update

After each successful delivery, the system updates the customer's reliability stats:

```sql
-- On successful contact delivery:
UPDATE customers SET
  successful_contact_deliveries = successful_contact_deliveries + 1,
  updated_at = NOW()
WHERE id = $1;

-- Check if no-contact should be unlocked:
-- (only if not already unlocked and not revoked)
UPDATE customers SET
  no_contact_unlocked = true,
  no_contact_unlocked_at = NOW()
WHERE id = $1
  AND successful_contact_deliveries >= [threshold from system_config]
  AND failed_deliveries = 0
  AND no_contact_unlocked = false
  AND no_contact_revoked = false
  AND created_at <= NOW() - INTERVAL '30 days';
```

### Dispute / Issue Handling

If a customer reports an issue (missing items, wrong items, etc.) from the order detail page:

1. Order flagged for admin review
2. Admin investigates and resolves (partial refund, full refund, reship, etc.)
3. Refunds processed via Stripe Refund API against the captured PaymentIntent
4. If the customer has no-contact delivery and reports a missing delivery:
   - No-contact access revoked: `no_contact_revoked = true`
   - Customer notified: "No-contact delivery has been paused on your account. Future deliveries will require a handoff."

---

## Payment Error Handling

### Card Decline at Checkout (Solo Orders)

Customer sees inline error:

```
⚠️ Payment failed
[Customer-friendly decline message]
[Try Again]  [Use Different Card]
```

Cart items preserved. No order created.

**Decline code mapping:**

| Stripe Code | Customer Message |
|---|---|
| `insufficient_funds` | Your card doesn't have enough available funds. |
| `card_declined` | Your card was declined. Please try a different card. |
| `expired_card` | Your card has expired. Please update your card. |
| `incorrect_cvc` | The security code is incorrect. Please check and try again. |
| `processing_error` | Temporary issue processing payment. Please try again. |
| (all others) | Payment couldn't be processed. Try a different card or try again later. |

### Card Decline at Batch Cutoff

Customer gets SMS with 2-hour grace window. If not resolved, removed from batch. See "Batch Cutoff Auth Job — Phase 3."

### Capture Failure

Order flagged for admin. See "Capture Flow" above. No automated retry.

### Overage at Capture

Capture full authorized amount. Flag for admin. Difference is typically small enough to write off.

---

## Backend Specification

### API Endpoints

---

#### `GET /api/cart`

**Authentication:** Required

**Purpose:** Return the customer's current cart with items, calculated fees, saved cards, delivery options, and auth hold estimate.

**Response:** Returns `store`, `deliveryContext`, `items[]`, `fees` (with `authBufferPct` and `authHold`), `savedCards[]`, `delivery` (address, hubs), `autoCheckout` availability, `cashAvailable`, `noContactAvailable`.

Key response fields:

```json
{
  "fees": {
    "itemSubtotal": 36.50,
    "isEstimated": false,
    "shoppingFee": null,
    "deliveryFee": 4.99,
    "serviceFee": 1.83,
    "total": 43.32,
    "authBufferPct": 0.05,
    "authHold": 45.49
  },
  "cashAvailable": true,
  "noContactAvailable": true,
  "autoCheckout": {
    "available": false,
    "enabled": false
  }
}
```

`cashAvailable` is true only if `customer.cash_verified = true`. `noContactAvailable` is true only if `customer.no_contact_unlocked = true AND no_contact_revoked = false`.

---

#### `POST /api/orders`

**Authentication:** Required

**Purpose:** Place an order.

**Request Body:**
```json
{
  "storeId": 3,
  "paymentMethod": "card",
  "paymentMethodId": "pm_xxx",
  "deliveryType": "door",
  "hubId": null,
  "contactDelivery": true,
  "dropOffSpot": null,
  "tipAmount": 5.00,
  "autoCheckout": false,
  "saveCard": true
}
```

**`paymentMethod`:** `"card"` or `"cash"`. If `"cash"`, `paymentMethodId`, `tipAmount`, `saveCard`, and `autoCheckout` are ignored.

**`contactDelivery`:** `true` = contact, `false` = no-contact. Server validates that no-contact is only allowed if `no_contact_unlocked = true`. If customer hasn't unlocked no-contact and sends `false`, server rejects with 403.

**Processing:** See Place Order flows above. Processing differs based on order type (solo card / batch card / cash).

**Success Response (201):**
```json
{
  "order": {
    "id": 142,
    "status": "authorized",
    "paymentMethod": "card",
    "storeName": "Calais IGA",
    "itemCount": 3,
    "total": 48.32,
    "authHold": 50.74,
    "deliveryDate": "2026-03-25",
    "deliveryType": "door",
    "contactDelivery": true
  }
}
```

For batch orders, status is `pending_auth` instead of `authorized`.

For cash orders:
```json
{
  "order": {
    "id": 143,
    "status": "pending_cash",
    "paymentMethod": "cash",
    "storeName": "Walmart Supercenter",
    "itemCount": 8,
    "total": 27.43,
    "authHold": null,
    "cashDue": 27.43,
    "deliveryDate": "2026-03-26"
  }
}
```

**Error Responses:**
- `400` — Validation error (empty cart, invalid delivery type, past cutoff)
- `402` — Payment declined (includes decline reason)
- `403` — Cash not verified, or no-contact not unlocked
- `409` — Order already exists for this batch participation
- `500` — Internal error

---

#### `POST /api/orders/:orderId/capture`

**Authentication:** Required (driver or store session)

**Purpose:** Capture payment after fulfillment.

**Request Body:**
```json
{
  "actualItemsTotal": 37.00,
  "adjustments": []
}
```

**Processing:** See Capture Flow above. Handles normal capture, overage, and failure.

---

#### `POST /api/orders/:orderId/cash-collected`

**Authentication:** Required (driver session)

**Purpose:** Record cash collection for a cash order.

**Request Body:**
```json
{
  "amountCollected": 27.43
}
```

**Processing:**
1. Validate order is cash and status is `pending_cash` or `out_for_delivery`
2. Record `cash_collected_amount` on order
3. If matches expected: status = `cash_collected`
4. If doesn't match: flag for admin, status = `cash_discrepancy`

---

#### `POST /api/orders/:orderId/adjust-tip`

**Authentication:** Required (customer)

**Purpose:** Increase tip after delivery. Card orders only.

**Request Body:**
```json
{
  "newTipAmount": 10.00
}
```

**Validation:** Order delivered within 24 hours. New tip > current tip. Customer has saved card.

---

#### `POST /api/admin/customers/:customerId/cash-verify`

**Authentication:** Required (admin session)

**Purpose:** Toggle cash payment verification for a customer.

**Request Body:**
```json
{
  "verified": true,
  "note": "Known community member, vouched by partner"
}
```

---

#### `POST /api/admin/customers/:customerId/no-contact`

**Authentication:** Required (admin session)

**Purpose:** Manually grant or revoke no-contact delivery.

**Request Body:**
```json
{
  "action": "revoke",
  "reason": "Reported missing delivery"
}
```

---

### Database Tables

```sql
-- ============================================================
-- ORDERS
-- ============================================================
CREATE TABLE orders (
  id                        SERIAL PRIMARY KEY,
  customer_id               INTEGER NOT NULL REFERENCES customers(id),
  store_id                  INTEGER NOT NULL REFERENCES stores(id),
  batch_participant_id      INTEGER REFERENCES batch_participants(id),

  -- Payment
  payment_method            VARCHAR(10) NOT NULL,  -- 'card' or 'cash'

  -- Status lifecycle
  status                    VARCHAR(30) NOT NULL DEFAULT 'pending_auth',
    -- pending_auth: batch order waiting for cutoff auth
    -- authorized: card hold placed
    -- auth_failed: card declined at cutoff (grace window active)
    -- fulfilling: driver/store working on it
    -- captured: card charged, order complete
    -- capture_failed: capture attempt failed
    -- capture_overage: actual exceeded auth
    -- delivered: physically delivered
    -- pending_cash: cash order awaiting delivery
    -- cash_collected: cash received by driver
    -- cash_discrepancy: cash amount doesn't match expected
    -- cancelled: cancelled before fulfillment
    -- refunded: refunded after capture

  -- Delivery
  delivery_type             VARCHAR(10) NOT NULL,   -- 'door' or 'hub'
  delivery_address          VARCHAR(300),
  delivery_lat              DECIMAL(10,7),
  delivery_lng              DECIMAL(10,7),
  hub_id                    INTEGER REFERENCES hubs(id),
  delivery_date             DATE NOT NULL,
  contact_delivery          BOOLEAN NOT NULL DEFAULT true,
  drop_off_spot             VARCHAR(200),
  delivery_photo_key        VARCHAR(255),

  -- Pricing
  item_subtotal             DECIMAL(8,2),
  is_estimated              BOOLEAN NOT NULL DEFAULT false,
  shopping_fee              DECIMAL(6,2),
  delivery_fee              DECIMAL(6,2) NOT NULL,
  service_fee               DECIMAL(6,2) NOT NULL,
  tip                       DECIMAL(6,2) NOT NULL DEFAULT 0,
  total                     DECIMAL(8,2) NOT NULL,

  -- Auth/capture (card orders only)
  auth_buffer_pct           DECIMAL(4,3),
  auth_hold_amount          DECIMAL(8,2),
  auth_grace_deadline       TIMESTAMPTZ,
  actual_items_total        DECIMAL(8,2),
  actual_service_fee        DECIMAL(6,2),
  actual_total              DECIMAL(8,2),
  captured_amount           DECIMAL(8,2),
  capture_difference        DECIMAL(8,2),

  -- Cash orders
  cash_due                  DECIMAL(8,2),
  cash_collected_amount     DECIMAL(8,2),

  -- Stripe (card orders only)
  stripe_payment_intent_id  VARCHAR(100),
  stripe_capture_id         VARCHAR(100),
  stripe_payment_method_id  VARCHAR(100),  -- stored for batch deferred auth

  -- Flags
  flagged                   BOOLEAN DEFAULT false,
  flag_reason               VARCHAR(100),

  -- Metadata
  item_count                INTEGER NOT NULL,
  created_at                TIMESTAMPTZ DEFAULT NOW(),
  updated_at                TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_customer ON orders(customer_id, created_at DESC);
CREATE INDEX idx_orders_store ON orders(store_id, status);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_delivery ON orders(delivery_date, status);
CREATE INDEX idx_orders_flagged ON orders(flagged) WHERE flagged = true;
CREATE INDEX idx_orders_pending_auth ON orders(status) WHERE status = 'pending_auth';
CREATE INDEX idx_orders_stripe ON orders(stripe_payment_intent_id)
  WHERE stripe_payment_intent_id IS NOT NULL;

-- ============================================================
-- ORDER ITEMS
-- ============================================================
CREATE TABLE order_items (
  id                        SERIAL PRIMARY KEY,
  order_id                  INTEGER NOT NULL REFERENCES orders(id),
  item_type                 VARCHAR(20) NOT NULL,

  -- Catalog items
  product_id                INTEGER REFERENCES products(id),
  unit_price                DECIMAL(8,2),

  -- Shopping list items
  estimated_price           DECIMAL(8,2),
  substitution_preference   VARCHAR(20),
  driver_notes              VARCHAR(300),

  -- Shared
  product_name              VARCHAR(200) NOT NULL,
  quantity                  INTEGER NOT NULL,

  -- Fulfillment
  actual_price              DECIMAL(8,2),
  actual_quantity           INTEGER,
  fulfilled                 BOOLEAN,
  substituted_with          VARCHAR(200),
  fulfillment_notes         VARCHAR(300),

  created_at                TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);

-- ============================================================
-- ORDER FLAGS (admin review queue)
-- ============================================================
CREATE TABLE order_flags (
  id              SERIAL PRIMARY KEY,
  order_id        INTEGER NOT NULL REFERENCES orders(id),
  flag_type       VARCHAR(50) NOT NULL,
    -- 'capture_exceeds_auth','capture_failed','auth_expiring',
    -- 'cash_discrepancy','delivery_issue','auth_failed_grace_expired'
  details         JSONB,
  resolved        BOOLEAN DEFAULT false,
  resolved_by     VARCHAR(100),
  resolved_at     TIMESTAMPTZ,
  resolution_note VARCHAR(500),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_order_flags_unresolved ON order_flags(resolved) WHERE resolved = false;

-- ============================================================
-- CASH VERIFICATION LOG (audit trail)
-- ============================================================
CREATE TABLE cash_verification_log (
  id              SERIAL PRIMARY KEY,
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  verified        BOOLEAN NOT NULL,
  admin_note      VARCHAR(500),
  verified_by     VARCHAR(100) NOT NULL,  -- admin username
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- CUSTOMER COLUMNS (added to existing customers table)
-- ============================================================
ALTER TABLE customers ADD COLUMN stripe_customer_id VARCHAR(100);
ALTER TABLE customers ADD COLUMN default_payment_method_id VARCHAR(100);
ALTER TABLE customers ADD COLUMN auto_checkout_enabled BOOLEAN DEFAULT false;
ALTER TABLE customers ADD COLUMN cash_verified BOOLEAN DEFAULT false;
ALTER TABLE customers ADD COLUMN cash_verified_at TIMESTAMPTZ;
ALTER TABLE customers ADD COLUMN cash_verified_by VARCHAR(100);
ALTER TABLE customers ADD COLUMN successful_contact_deliveries INTEGER DEFAULT 0;
ALTER TABLE customers ADD COLUMN failed_deliveries INTEGER DEFAULT 0;
ALTER TABLE customers ADD COLUMN no_contact_unlocked BOOLEAN DEFAULT false;
ALTER TABLE customers ADD COLUMN no_contact_unlocked_at TIMESTAMPTZ;
ALTER TABLE customers ADD COLUMN no_contact_revoked BOOLEAN DEFAULT false;
ALTER TABLE customers ADD COLUMN no_contact_revoked_at TIMESTAMPTZ;
ALTER TABLE customers ADD COLUMN no_contact_revoke_reason VARCHAR(200);
```

### Tables Referenced (Defined Elsewhere)
- `customers`, `stores`, `hubs` — C-01
- `products`, `cart_items` — C-03
- `batch_participants`, `weekly_batches` — C-06
- `fee_tiers`, `service_fee_config`, `shopping_fee_config` — A-01 / C-06
- `system_config` — A-01

---

## Config Values

| Config Key | Default | Description |
|---|---|---|
| `partnered_auth_buffer_pct` | 0.05 | 5% buffer for partnered store orders |
| `batch_auth_buffer_pct` | 0.10 | 10% buffer for batch group orders |
| `ondemand_auth_buffer_pct` | 0.20 | 20% buffer for on-demand solo orders |
| `cash_order_max` | 100.00 | Maximum order total for cash payment |
| `no_contact_unlock_threshold` | 5 | Successful contact deliveries needed to unlock no-contact |
| `no_contact_min_account_age_days` | 30 | Minimum account age for no-contact eligibility |
| `batch_auth_grace_hours` | 2 | Hours after cutoff decline before removing from batch |
| `tip_adjustment_window_hours` | 24 | Hours after delivery to adjust tip |
| `auth_expiry_warning_days` | 5 | Days after auth to flag for expiry monitoring |

---

## Scheduled Jobs

### Batch Cutoff Auth Job

**Trigger:** Checks every 15 minutes for batches whose cutoff just passed. Runs via node-cron.

**Logic:** See "Batch Cutoff Auth Job" section above (Phase 1-4).

### Grace Window Expiry Job

**Trigger:** Checks every 15 minutes for orders with `status = 'auth_failed'` whose `auth_grace_deadline` has passed.

**Logic:** Phase 3 of the batch cutoff job — remove failed participants, recalculate fees.

### Auth Expiry Monitor

**Trigger:** Daily at 6am EST.

**Logic:** Find orders with `status = 'authorized'` where `created_at > 5 days ago`. Flag for admin review — something went wrong if the order hasn't been fulfilled in 5 days.

### Cart Expiry Cleanup

**Trigger:** Daily at 3am EST.

**Logic:** Delete `cart_items` linked to batch participations where the batch cutoff has passed and no order was created. This cleans up stale cart data from customers who didn't check out.

---

## ECS Fargate Notes

- Stripe API calls are the latency bottleneck at checkout. A PaymentIntent creation takes 1-3 seconds. Show a loading spinner on the "Place Order" button.
- The batch cutoff auth job is the most critical scheduled job. It must complete before the driver starts shopping. At your scale (5-10 participants per batch), it runs in under 30 seconds. Log every step to CloudWatch.
- Idempotency keys on all Stripe calls prevent duplicate charges on network retries. Use the format `order-auth-{orderId}` and `order-capture-{orderId}`.
- Stripe webhook events (`payment_intent.succeeded`, `payment_intent.payment_failed`) can be used as a backup notification channel but are NOT required for the core flow. The API handles all state transitions synchronously. Webhooks can be added later for monitoring/alerting.
- The Stripe secret key is in AWS Secrets Manager. The Stripe publishable key (for Stripe Elements on the frontend) is a build-time environment variable — it's public by design.

## Third-Party API Costs

| Service | Operation | Cost | Monthly Estimate |
|---|---|---|---|
| Stripe | Card processing (2.9% + $0.30) | Per transaction | ~$30-50 (at $1,000-1,500 GMV) |
| Stripe | Auth/capture (no extra fee) | Included | $0 |
| Twilio SMS | Auth confirmations, decline notifications | $0.0079/msg | ~$3-5 |
| **Total** | | | **~$33-55/month** |
