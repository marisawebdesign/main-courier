# C-05: Order History, Smart Reorder & Post-Delivery

> **Spec Version:** 1.0
> **Status:** Complete — Ready for Development
> **Dependencies:** C-01 (authenticated session), C-03 (shopping list builder, product catalog), C-04 (orders, order_items, payment records), PostgreSQL
> **Relates to:** C-03 (smart reorder loads items into shopping list builder or cart), C-04 (order/payment records, tip adjustment, order_flags), C-06 (batch participation history feeds reorder intelligence), D-02 (driver fulfillment data populates order detail)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md.
> **Audit note:** This spec was written with the AUDIT-FRAMEWORK.md applied during authoring. A self-audit is included at the end.

---

## Overview

C-05 covers everything a customer sees and does AFTER placing an order: order history, order detail with fulfillment comparison, smart reorder, delivery rating, and issue reporting. It is the customer's primary tool for reviewing past activity and reordering efficiently.

### Design Principles

1. **Order history is the customer's receipt book.** Every order they've ever placed, with full detail on what was ordered, what was fulfilled, and what was charged.
2. **Smart reorder is conservative.** It does NOT blindly re-add everything. It categorizes items by reorder likelihood and lets the customer choose. One-time items are unchecked by default.
3. **Post-delivery actions are time-gated.** Rating and issue reporting are available for 7 days after delivery. Tip adjustment for 24 hours. After that, contact admin directly.
4. **Issue reporting is structured, not freeform.** Predefined categories with specific item selection. This generates actionable flags for admin, not vague complaints.
5. **All data is the customer's own.** Every endpoint filters by `customer_id` from the session. A customer can never see another customer's orders.

---

## Order History Page

Accessed from the Account menu or a "My Orders" link in the header/navigation.

### Page Layout — Mobile

```
┌─────────────────────────────────────────┐
│  ← Back                                │
│                                         │
│  My Orders                              │
│                                         │
│  ── This Week ────────────────────────  │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 🏪 Walmart · Thu Mar 26         │   │
│  │ 12 items · $31.86               │   │
│  │ 🟢 Delivered                     │   │
│  │                          [→]    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 🌾 Martha's Pies · Wed Mar 25   │   │
│  │ 2 items · $16.50                │   │
│  │ 🟡 Out for delivery             │   │
│  │                          [→]    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ── Last Week ────────────────────────  │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 🏪 Walmart · Thu Mar 19         │   │
│  │ 10 items · $28.42               │   │
│  │ 🟢 Delivered                     │   │
│  │                          [→]    │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ── Earlier ──────────────────────────  │
│                                         │
│  (older orders grouped by month)        │
│                                         │
│  ── March 2026 ──                       │
│  ┌─────────────────────────────────┐   │
│  │ 🏪 Hannaford · Wed Mar 12       │   │
│  │ 8 items · $45.20                │   │
│  │ 🟢 Delivered                     │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Load more]                            │
│                                         │
└─────────────────────────────────────────┘
```

### Order Card Contents

Each order in the history list displays:

| Element | Source | Details |
|---|---|---|
| Store icon | `stores.partnership_tier` | 🏪 for stores, 🌾 for producers |
| Store/producer name | `stores.name` or `producers.name` | Via `orders.store_id` |
| Delivery date | `orders.delivery_date` | Formatted as "Day Mon DD" |
| Item count | `orders.item_count` | Number of items in the order |
| Total charged | `orders.actual_total` if captured, `orders.total` if not yet | Actual charge after fulfillment, or estimated total if still pending |
| Status | `orders.status` | See status display mapping below |
| Tap target | Entire card | Navigates to order detail page |

### Status Display Mapping

| Order Status | Display Text | Color Indicator |
|---|---|---|
| `pending_auth` | Pending | ⚪ Gray |
| `authorized` | Confirmed | 🔵 Blue |
| `auth_failed` | Payment issue | 🔴 Red |
| `fulfilling` | Being prepared | 🟡 Yellow |
| `captured` | Complete | 🟢 Green |
| `delivered` | Delivered | 🟢 Green |
| `pending_cash` | Pending — cash at delivery | ⚪ Gray |
| `cash_collected` | Complete — cash paid | 🟢 Green |
| `cash_discrepancy` | Under review | 🟡 Yellow |
| `cancelled` | Cancelled | ⚫ Dark gray |
| `refunded` | Refunded | ⚫ Dark gray |
| `capture_failed` | Payment issue | 🔴 Red |
| `capture_overage` | Complete | 🟢 Green (customer doesn't need to know about overage) |

**Important:** Status indicators use BOTH color AND text. Color alone is insufficient for accessibility (Audit 6.3).

### Grouping and Pagination

Orders are grouped by time period:
- **This Week** — orders with `delivery_date` in the current calendar week
- **Last Week** — orders with `delivery_date` in the previous calendar week
- **Earlier** — grouped by month ("March 2026", "February 2026", etc.)

Pagination: Load 20 orders initially. "Load more" button fetches the next 20. Offset-based pagination (simple, adequate at this scale).

### Empty State

```
┌─────────────────────────────────────────┐
│  My Orders                              │
│                                         │
│  You haven't placed any orders yet.     │
│                                         │
│  Browse stores near you to get started. │
│                                         │
│  [Browse Stores →]                      │
│                                         │
└─────────────────────────────────────────┘
```

"Browse Stores" navigates to the home screen map.

### Loading State

While order history is being fetched, show 3 skeleton card placeholders (gray rectangles pulsing) with the "My Orders" header already rendered. This provides immediate visual feedback on slow connections.

---

## Order Detail Page

Accessed by tapping an order card in the history list.

### Page Layout

```
┌─────────────────────────────────────────┐
│  ← My Orders                            │
│                                         │
│  Order #142                              │
│  🏪 Walmart Supercenter                 │
│  Thu Mar 26 · Delivered                 │
│                                         │
│  ── Items ────────────────────────────  │
│                                         │
│  ✅ 2% Milk, 1 gallon             x1    │
│     Ordered: ~$3.50                     │
│     Charged: $3.29                      │
│                                         │
│  ✅ Cheerios, family size          x1    │
│     Ordered: ~$5.99                     │
│     Charged: $5.99                      │
│                                         │
│  🔄 Ground beef, 1 lb             x2    │
│     Ordered: ~$9.98                     │
│     Substituted: Ground turkey, 1 lb    │
│     Charged: $8.98                      │
│     Driver note: "Beef was out,         │
│      turkey was same price per lb"      │
│                                         │
│  ❌ Bananas, 1 bunch                    │
│     Not available — skipped             │
│     Charged: $0.00                      │
│                                         │
│  ── Charges ──────────────────────────  │
│                                         │
│  Items:                        $18.26   │
│  Shopping fee:                  $3.00   │
│  Delivery fee:                  $3.99   │
│  Service fee:                   $0.91   │
│  Tip:                           $5.00   │
│  ────────────────────────────────────   │
│  Total charged:                $31.16   │
│                                         │
│  Originally held: $32.92               │
│  Released: $1.76                        │
│                                         │
│  💳 Visa ending 4242                    │
│                                         │
│  ── Delivery ─────────────────────────  │
│                                         │
│  🤝 Contact delivery                    │
│  📍 42 Route 1, Weston, ME             │
│  🕐 Delivered at 3:42 PM               │
│  🚗 Driver: Mike                        │
│                                         │
│  ── Actions ──────────────────────────  │
│                                         │
│  [Reorder →]                            │
│  [Adjust Tip]         (if within 24hr)  │
│  [Rate Delivery]      (if within 7 days)│
│  [Report Issue]       (if within 7 days)│
│                                         │
└─────────────────────────────────────────┘
```

### Item Display — Fulfillment Comparison

Each order item shows what was ordered alongside what was actually fulfilled. This is critical for unpartnered store orders where the driver shops on behalf of the customer.

**Item status icons:**

| Icon | Meaning | Display |
|---|---|---|
| ✅ | Fulfilled as ordered | Show ordered price and charged price |
| 🔄 | Substituted | Show original item, substitution, and charged price. Show driver's fulfillment note if present. |
| ❌ | Skipped / unavailable | Show "Not available — skipped" and $0.00 charged |
| ⚠️ | Price different | Show ordered estimate and actual price (for items fulfilled as ordered but at different price) |

**For partnered store items:** The comparison is between catalog price at order time (`order_items.unit_price`) and actual price at fulfillment (`order_items.actual_price`). Most will match. Price differences show ⚠️.

**For unpartnered store items:** The comparison is between the customer's estimate (`order_items.estimated_price`) and the driver's receipt (`order_items.actual_price`). Substitutions and skips are common. The `substituted_with` and `fulfillment_notes` fields from `order_items` provide context.

### Charge Breakdown

Shows the final actual charges, not the estimates. Each line item:

| Line | Source | Notes |
|---|---|---|
| Items | `orders.actual_items_total` | Sum of actual prices for all fulfilled items |
| Shopping fee | `orders.shopping_fee` | Only for unpartnered stores. NULL for partnered. |
| Delivery fee | `orders.delivery_fee` | |
| Service fee | `orders.actual_service_fee` | Recalculated on actual items total |
| Tip | `orders.tip` | |
| Total charged | `orders.captured_amount` or `orders.cash_collected_amount` | |
| Originally held | `orders.auth_hold_amount` | Only for card orders |
| Released | `orders.auth_hold_amount - orders.captured_amount` | Only shown if positive (hold was partially released) |

**For cash orders:** No "Originally held" or "Released" lines. Instead show "💵 Paid cash at delivery: $XX.XX"

**For orders not yet captured** (status `authorized`, `fulfilling`): Show estimated totals with `~` prefix and a note: "Final amount will be determined after your order is fulfilled."

### Delivery Section

| Field | Source | Notes |
|---|---|---|
| Delivery type | `orders.contact_delivery` | "🤝 Contact delivery" or "📦 No-contact delivery" |
| Address | `orders.delivery_address` | Or hub name if hub pickup |
| Delivered at | Timestamp from delivery completion | Formatted as "h:mm AM/PM" |
| Driver | Driver first name from `drivers` table | First name only — no last name, no phone number |
| Delivery photo | `orders.delivery_photo_key` | Only for no-contact deliveries. Displayed as tappable thumbnail that opens full-size image. |

**If delivery hasn't happened yet** (status `authorized`, `fulfilling`): Show delivery date and time window instead: "📅 Thu Mar 26 · Afternoon delivery"

### Action Buttons

| Action | Condition | Behavior |
|---|---|---|
| Reorder | Always available on delivered/captured/cash_collected orders | Opens smart reorder flow (see below) |
| Adjust Tip | Card orders, within 24 hours of delivery, `status = 'delivered'` | Opens tip adjustment UI (defined in C-04, UI rendered here) |
| Rate Delivery | Within 7 days of delivery | Opens rating UI (see below) |
| Report Issue | Within 7 days of delivery | Opens issue reporting flow (see below) |

**Time gating logic:** The backend calculates whether each action is available based on the order's delivery timestamp and returns boolean flags in the API response. The frontend renders or hides buttons accordingly. Time checks happen server-side, not client-side (client clocks are unreliable).

### Order Detail — Error States

**Order not found (404):**
```
This order could not be found.
It may have been removed or you
may not have access to it.

[← Back to Orders]
```

**Network error loading order:**
```
Couldn't load order details.
Check your connection and try again.

[Retry]  [← Back to Orders]
```

---

## Smart Reorder

Accessed by tapping "Reorder" on any delivered order's detail page.

### Reorder Intelligence

When the customer taps "Reorder," the system analyzes the order's items against the customer's full order history at that store to categorize each item by reorder likelihood.

**Classification logic (server-side):**

```
For each item in the selected order:

  1. Count how many times this customer has ordered this item
     at this store (across ALL past orders):

     For partnered stores:
       SELECT COUNT(*) FROM order_items oi
       JOIN orders o ON o.id = oi.order_id
       WHERE o.customer_id = $1
       AND o.store_id = $2
       AND oi.product_id = $3
       AND o.status IN ('captured','delivered','cash_collected');

     For unpartnered stores (fuzzy match on description):
       SELECT COUNT(*) FROM order_items oi
       JOIN orders o ON o.id = oi.order_id
       WHERE o.customer_id = $1
       AND o.store_id = $2
       AND LOWER(TRIM(oi.product_name)) = LOWER(TRIM($3))
       AND o.status IN ('captured','delivered','cash_collected');

  2. Classify:
     - order_count >= 2 → "regular" (pre-checked ☑)
     - order_count = 1 → "one_time" (unchecked ☐)

  3. For partnered stores, also check product availability:
     - Product active AND available → "available"
     - Product active AND NOT available → "unavailable"
     - Product inactive/deleted → "removed"

  4. For partnered stores, check price changes:
     - Current price != price at time of order → flag price_changed
       with old_price and new_price
```

**Seasonal detection (simple heuristic):** Items ordered only once AND ordered during November-December are flagged as "possible seasonal item." This is a soft hint, not a hard rule. The flag text says "Ordered once in [month] — seasonal item?" The customer decides.

### Reorder Page — Unpartnered Store

```
┌─────────────────────────────────────────┐
│  ← Back to Order #142                   │
│                                         │
│  Reorder from Walmart                   │
│  Based on your Mar 19 order             │
│                                         │
│  ── Your Regulars ────────────────────  │
│  Items you order frequently             │
│                                         │
│  ☑ 2% Milk, 1 gallon            ~$3.50  │
│    Ordered 6 times                      │
│  ☑ Cheerios, family size         ~$5.99  │
│    Ordered 4 times                      │
│  ☑ Ground beef, 1 lb x2         ~$9.98  │
│    Ordered 5 times                      │
│  ☑ Bread, sliced                 ~$2.99  │
│    Ordered 3 times                      │
│  ☑ Eggs, 1 dozen                 ~$3.49  │
│    Ordered 8 times                      │
│  ☑ Toilet paper, 12 roll         ~$8.99  │
│    Ordered 2 times                      │
│                                         │
│  ── One-Time Items ───────────────────  │
│  Items you've only ordered once         │
│                                         │
│  ☐ Birthday candles              ~$3.99  │
│    Ordered once — want it again?        │
│  ☐ Cold medicine                 ~$7.99  │
│    Ordered once — want it again?        │
│  ☐ Wrapping paper                ~$4.99  │
│    Ordered once in December —           │
│    seasonal item?                       │
│                                         │
│  ── Add Something New ────────────────  │
│  [+ Add item]                           │
│                                         │
│  ────────────────────────────────────   │
│  Selected: 6 items · Est. ~$34.95       │
│  [Build Order from Selected →]          │
│                                         │
└─────────────────────────────────────────┘
```

### Reorder Page — Partnered Store

```
┌─────────────────────────────────────────┐
│  ← Back to Order #145                   │
│                                         │
│  Reorder from Calais IGA                │
│  Based on your Mar 20 order             │
│                                         │
│  ── Available ────────────────────────  │
│                                         │
│  ☑ Sourdough Bread               $4.50  │
│    Ordered 3 times                      │
│  ☑ Apple Pie                    $12.00  │
│    Ordered 2 times                      │
│  ☑ Honey 16oz                    $8.50  │
│    ⚠️ Was $8.00 — price increased      │
│    Ordered 4 times                      │
│  ☐ Seasonal Cider                $5.00  │
│    Ordered once in November —           │
│    seasonal item?                       │
│                                         │
│  ── Unavailable ──────────────────────  │
│                                         │
│  ✕ Pumpkin Bread                 $6.00  │
│    No longer available                  │
│                                         │
│  ── Add Something New ────────────────  │
│  [+ Browse Products →]                  │
│                                         │
│  ────────────────────────────────────   │
│  Selected: 3 items · $25.00             │
│  [Add Selected to Cart →]              │
│                                         │
└─────────────────────────────────────────┘
```

### "Build Order" / "Add to Cart" Behavior

**For unpartnered stores:** "Build Order from Selected" takes checked items and loads them into the shopping list builder (C-03) for the appropriate batch route. If no active batch exists for this store, the customer is prompted to join/create one via the Community tab (C-06). The items are pre-loaded into the shopping list with their previous quantities, substitution preferences, and estimated prices. The customer reviews and submits from the shopping list builder.

**For partnered stores:** "Add Selected to Cart" adds checked items to the cart at current catalog prices (not historical prices). If the cart already has items from a different store, the single-store conflict dialog from C-03 appears.

**"+ Add item" / "+ Browse Products":** Opens the shopping list builder (unpartnered) or product catalog (partnered) for that store, allowing the customer to add new items beyond what was in the original order.

### Edge Cases

**Reordering a very old order:** Item descriptions may not fuzzy-match well if the customer's language changed ("2% milk" vs "milk 2 percent"). The system does its best with case-insensitive trimmed matching. Items that don't match any recent orders are classified as "one_time" and unchecked. This is conservative — better to under-match than over-match.

**Reordering from a deactivated store:** If the store is no longer active, the reorder button is hidden. The order detail page shows the store name with "(no longer available)" and no reorder option.

**Reordering when no batch route exists (unpartnered):** "Build Order from Selected" checks for an active batch. If none exists, the customer sees:

```
No delivery route is scheduled for
Walmart this week.

Join the community to help start one,
or order on-demand.

[Join Community →]  [Order Solo →]
```

"Order Solo" is only shown if `on_demand_enabled = true`.

**Empty reorder (all items unavailable/removed):** If every item in the order is unavailable or removed:

```
None of the items from this order
are currently available.

Browse [Store Name] for what's
available now.

[Browse Products →]
```

---

## Delivery Rating

Available on the order detail page for 7 days after delivery.

### Rating UI

```
┌─────────────────────────────────────────┐
│  Rate Your Delivery                     │
│                                         │
│  Order #142 · Walmart · Mar 26          │
│                                         │
│     👍          👎                       │
│   Great       Not great                │
│                                         │
│  [Submit Rating]                        │
│                                         │
└─────────────────────────────────────────┘
```

Simple thumbs up / thumbs down. No text input, no stars, no elaborate survey. The customer taps one and submits.

**After submitting:**

```
Thanks for your feedback!

[← Back to Order]
```

The button changes to "Rated: 👍" or "Rated: 👎" and is no longer interactive.

**What happens with the rating:**
- Stored in `delivery_ratings` table
- Thumbs-down ratings automatically create an `order_flags` entry with `flag_type = 'negative_rating'` for admin review
- Admin can see rating patterns per driver in the admin dashboard
- Ratings are NOT visible to other customers or to the driver (driver sees aggregate stats, not individual ratings, through the admin)

### Rating — Edge Cases

**Customer taps rating twice quickly:** The submit button disables immediately after first tap. The API uses a unique constraint on `(order_id, customer_id)` to prevent duplicate ratings. If a duplicate request arrives, the API returns the existing rating (idempotent).

**Rating after 7 days:** The "Rate Delivery" button is not rendered. If a stale client somehow sends a rating request, the API validates `delivery_completed_at + 7 days > NOW()` and returns 403 with message "Rating window has closed."

---

## Issue Reporting

Available on the order detail page for 7 days after delivery. Structured categories, not freeform.

### Issue Reporting Flow

**Step 1: Select issue type**

```
┌─────────────────────────────────────────┐
│  Report an Issue                        │
│  Order #142 · Walmart · Mar 26          │
│                                         │
│  What happened?                         │
│                                         │
│  ○ Missing item(s)                      │
│  ○ Wrong item(s)                        │
│  ○ Damaged item(s)                      │
│  ○ Never received delivery              │
│  ○ Charged incorrectly                  │
│  ○ Other                                │
│                                         │
│  [Next →]                               │
│                                         │
└─────────────────────────────────────────┘
```

**Step 2a: Missing / Wrong / Damaged items — select which items**

```
┌─────────────────────────────────────────┐
│  Which items were [missing/wrong/       │
│  damaged]?                              │
│                                         │
│  ☐ 2% Milk, 1 gallon             $3.29  │
│  ☐ Cheerios, family size          $5.99  │
│  ☐ Ground turkey, 1 lb x2        $8.98  │
│  ☐ (select all that apply)              │
│                                         │
│  [Next →]                               │
│                                         │
└─────────────────────────────────────────┘
```

**Step 2b: Damaged items — optional photo**

```
┌─────────────────────────────────────────┐
│  Add a photo (optional)                 │
│                                         │
│  [📷 Take Photo]  [📁 Upload]          │
│                                         │
│  A photo helps us resolve this faster.  │
│                                         │
│  [Next →]                               │
│                                         │
└─────────────────────────────────────────┘
```

Photo uploaded to S3 via pre-signed URL (same pattern as profile photos in C-01).

**Step 2c: "Never received" or "Charged incorrectly" or "Other" — brief description**

```
┌─────────────────────────────────────────┐
│  Tell us more (optional)                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ e.g., "Package was not at my    │   │
│  │ door when I got home"           │   │
│  └─────────────────────────────────┘   │
│  Max 500 characters                     │
│                                         │
│  [Submit Report →]                      │
│                                         │
└─────────────────────────────────────────┘
```

**Step 3: Confirmation**

```
┌─────────────────────────────────────────┐
│  Report Submitted ✓                     │
│                                         │
│  We've received your report and will    │
│  review it within 24 hours.             │
│                                         │
│  If a credit is issued, it will be      │
│  applied to your card automatically.    │
│                                         │
│  [← Back to Order]                      │
│                                         │
└─────────────────────────────────────────┘
```

### What Happens with the Report

1. An `order_issues` row is created with the issue type, selected items, description, and photo key
2. An `order_flags` row is created with `flag_type` matching the issue type (e.g., `'customer_missing_items'`, `'customer_wrong_items'`, `'customer_never_received'`)
3. Admin dashboard shows the flag with full context: order details, selected items, customer description, photo
4. Admin resolves by choosing one of:
   - **Issue partial credit:** Stripe refund for the value of affected items. System creates a Stripe refund against the captured PaymentIntent. Order status remains `delivered` — a partial refund doesn't change the delivery status.
   - **Issue full refund:** Stripe refund for the entire order. Order status changes to `refunded`.
   - **No action needed:** Mark flag as resolved with a note explaining why.
   - **Contact customer:** Admin contacts the customer directly to resolve.
5. If the issue is "Never received delivery" AND the customer has no-contact delivery:
   - No-contact access is automatically revoked (`no_contact_revoked = true`)
   - Customer notified: "No-contact delivery has been paused on your account."
   - Flag includes this automatic action for admin visibility

### Issue Reporting — Edge Cases

**Customer reports issue on an order that's already been reported:** The "Report Issue" button shows "Issue Reported ✓" and is no longer interactive after the first report. A customer can only file ONE report per order. If they have additional issues, they contact admin directly. The unique constraint on `order_issues(order_id)` enforces this.

**Customer reports issue on a cash order:** Same flow, but credit is NOT automatic. The flag notes that this is a cash order and admin must handle resolution manually (cash refund, credit on next order, etc.).

**Submitting report fails (network error):** The submit button shows a loading spinner. On failure: "Couldn't submit your report. Check your connection and try again. [Retry]". The report data is preserved in the form so the customer doesn't have to re-enter it.

---

## Tip Adjustment UI

The tip adjustment mechanism is defined in C-04. The UI lives on the order detail page.

### Display

When the order is delivered and within 24 hours, the "Adjust Tip" button appears in the Actions section.

```
┌─────────────────────────────────────────┐
│  Adjust Tip                             │
│                                         │
│  Order #142 · Walmart · Mar 26          │
│  Driver: Mike                           │
│                                         │
│  Current tip: $5.00                     │
│                                         │
│  [$5]  [$8]  [$10]  [$15]  [Custom]    │
│                                         │
│  Additional charge: $0.00               │
│  (updates as you select)                │
│                                         │
│  [Update Tip →]                         │
│                                         │
└─────────────────────────────────────────┘
```

Pre-set buttons start at the current tip amount. Selecting a higher amount updates the "Additional charge" line. The current tip button is highlighted but not selectable (no change = no charge).

**Validation:**
- New tip must be > current tip (increases only)
- Customer must have a saved payment method
- Within 24 hours of delivery timestamp (server-validated)

**After updating:** "Tip updated to $10.00. $5.00 charged to your card." Button becomes non-interactive and shows "Tip: $10.00 ✓"

**Only for card orders.** Tip adjustment is not available for cash orders (tip section is hidden entirely).

---

## Backend Specification

### API Endpoints

---

#### `GET /api/orders`

**Authentication:** Required

**Purpose:** Return paginated order history for the authenticated customer.

**Query Parameters:**
- `page` (optional, default 1) — Page number
- `limit` (optional, default 20, max 50) — Orders per page

**Processing:**
1. Query orders WHERE `customer_id` = session customer, ordered by `created_at DESC`
2. Paginate with offset
3. For each order: compute display status, format dates, include store name

**Response:**
```json
{
  "orders": [
    {
      "id": 142,
      "storeId": 1,
      "storeName": "Walmart Supercenter",
      "storeIcon": "store",
      "partnershipTier": "unpartnered",
      "deliveryDate": "2026-03-26",
      "itemCount": 12,
      "total": 31.86,
      "isActualTotal": true,
      "status": "delivered",
      "statusDisplay": "Delivered",
      "statusColor": "green",
      "paymentMethod": "card",
      "createdAt": "2026-03-23T14:30:00-04:00"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 47,
    "totalPages": 3
  }
}
```

`isActualTotal` is `true` when `actual_total` or `captured_amount` is populated (order has been fulfilled). `false` means the `total` shown is the estimate.

---

#### `GET /api/orders/:orderId`

**Authentication:** Required

**Purpose:** Return full order detail including items, fulfillment comparison, charges, delivery info, and available actions.

**Authorization:** Order must belong to the authenticated customer. Query: `WHERE id = $1 AND customer_id = $2`.

**Response:**
```json
{
  "order": {
    "id": 142,
    "storeId": 1,
    "storeName": "Walmart Supercenter",
    "partnershipTier": "unpartnered",
    "status": "delivered",
    "statusDisplay": "Delivered",
    "paymentMethod": "card",
    "deliveryDate": "2026-03-26",
    "deliveryType": "door",
    "contactDelivery": true,
    "deliveryAddress": "42 Route 1, Weston, ME 04424",
    "deliveredAt": "2026-03-26T15:42:00-04:00",
    "deliveryPhotoKey": null,
    "driverFirstName": "Mike",
    "createdAt": "2026-03-23T14:30:00-04:00"
  },
  "items": [
    {
      "id": 1,
      "itemType": "shopping_list",
      "productName": "2% Milk, 1 gallon",
      "quantity": 1,
      "orderedPrice": 3.50,
      "isEstimatedPrice": true,
      "actualPrice": 3.29,
      "actualQuantity": 1,
      "fulfilled": true,
      "substitutedWith": null,
      "fulfillmentNotes": null,
      "fulfillmentStatus": "fulfilled"
    },
    {
      "id": 3,
      "itemType": "shopping_list",
      "productName": "Ground beef, 1 lb",
      "quantity": 2,
      "orderedPrice": 9.98,
      "isEstimatedPrice": true,
      "actualPrice": 8.98,
      "actualQuantity": 2,
      "fulfilled": true,
      "substitutedWith": "Ground turkey, 1 lb",
      "fulfillmentNotes": "Beef was out, turkey was same price per lb",
      "fulfillmentStatus": "substituted"
    },
    {
      "id": 4,
      "itemType": "shopping_list",
      "productName": "Bananas, 1 bunch",
      "quantity": 1,
      "orderedPrice": 1.29,
      "isEstimatedPrice": true,
      "actualPrice": null,
      "actualQuantity": 0,
      "fulfilled": false,
      "substitutedWith": null,
      "fulfillmentNotes": null,
      "fulfillmentStatus": "skipped"
    }
  ],
  "charges": {
    "itemsTotal": 18.26,
    "isActual": true,
    "shoppingFee": 3.00,
    "deliveryFee": 3.99,
    "serviceFee": 0.91,
    "tip": 5.00,
    "totalCharged": 31.16,
    "authHoldAmount": 32.92,
    "releasedAmount": 1.76,
    "cashDue": null,
    "cashCollected": null
  },
  "card": {
    "brand": "visa",
    "last4": "4242"
  },
  "actions": {
    "canReorder": true,
    "canAdjustTip": true,
    "tipAdjustDeadline": "2026-03-27T15:42:00-04:00",
    "canRate": true,
    "hasRated": false,
    "ratingDeadline": "2026-04-02T15:42:00-04:00",
    "canReportIssue": true,
    "hasReportedIssue": false,
    "issueReportDeadline": "2026-04-02T15:42:00-04:00"
  }
}
```

**Key details:**
- `fulfillmentStatus` is computed from `fulfilled`, `substitutedWith`, and `actualQuantity`: `"fulfilled"` (as ordered), `"substituted"` (different item), `"skipped"` (not available), `"price_changed"` (fulfilled but different price).
- `actions` flags are all computed server-side based on current time vs. deadlines. The frontend renders buttons based on these flags.
- `orderedPrice` maps to `unit_price` (partnered) or `estimated_price` (unpartnered). `isEstimatedPrice` tells the frontend whether to prefix with `~`.
- The `card` object ONLY contains brand and last4. Never full card details.

---

#### `GET /api/orders/:orderId/reorder`

**Authentication:** Required

**Purpose:** Return smart reorder analysis for a past order.

**Authorization:** Order must belong to the authenticated customer. Store must still be active.

**Processing:**
1. Load order items
2. For each item, count historical order frequency (see classification logic above)
3. For partnered stores: check current product availability and price
4. Detect seasonal patterns (November-December one-time items)
5. Return categorized items

**Response:**
```json
{
  "sourceOrder": {
    "id": 142,
    "storeId": 1,
    "storeName": "Walmart Supercenter",
    "partnershipTier": "unpartnered",
    "deliveryDate": "2026-03-19"
  },
  "activeBatch": {
    "batchId": 45,
    "deliveryDay": "thursday",
    "deliveryDate": "2026-03-26",
    "available": true
  },
  "items": [
    {
      "originalItemId": 1,
      "productName": "2% Milk, 1 gallon",
      "quantity": 1,
      "estimatedPrice": 3.50,
      "substitutionPreference": "similar_ok",
      "driverNotes": null,
      "classification": "regular",
      "orderCount": 6,
      "preChecked": true,
      "availability": null,
      "priceChanged": false,
      "currentPrice": null,
      "seasonal": false
    },
    {
      "originalItemId": 8,
      "productName": "Wrapping paper",
      "quantity": 1,
      "estimatedPrice": 4.99,
      "substitutionPreference": "skip",
      "driverNotes": null,
      "classification": "one_time",
      "orderCount": 1,
      "preChecked": false,
      "availability": null,
      "priceChanged": false,
      "currentPrice": null,
      "seasonal": true,
      "seasonalMonth": "December"
    }
  ],
  "storeActive": true,
  "onDemandAvailable": false
}
```

For partnered stores, additional fields per item:
```json
{
  "productId": 101,
  "availability": "available",
  "priceChanged": true,
  "currentPrice": 8.50,
  "originalPrice": 8.00
}
```

`availability` values: `"available"`, `"unavailable"` (temporarily out), `"removed"` (product deleted/deactivated).

**Error Responses:**
- `401` — Not authenticated
- `403` — Order doesn't belong to this customer
- `404` — Order not found
- `410` — Store is no longer active (return `storeActive: false` in body rather than a hard 410, so the frontend can show a helpful message)

---

#### `POST /api/orders/:orderId/rate`

**Authentication:** Required

**Purpose:** Submit a delivery rating.

**Request Body:**
```json
{
  "rating": "positive"
}
```

`rating` must be `"positive"` or `"negative"`. No other values.

**Validation:**
- Order must belong to authenticated customer
- Order status must be `delivered` or `cash_collected`
- Delivery must have been completed within 7 days (server checks `delivered_at + 7 days > NOW()`)
- No existing rating for this order (unique constraint)

**Processing:**
1. Insert `delivery_ratings` row
2. If rating is `negative`: create `order_flags` row with `flag_type = 'negative_rating'`
3. Return success

**Response:**
```json
{
  "rated": true,
  "rating": "positive"
}
```

**Error Responses:**
- `400` — Invalid rating value
- `403` — Rating window closed or order doesn't belong to customer
- `409` — Already rated this order

---

#### `POST /api/orders/:orderId/issues`

**Authentication:** Required

**Purpose:** Submit an issue report.

**Request Body:**
```json
{
  "issueType": "missing_items",
  "affectedItemIds": [1, 4],
  "description": null,
  "photoKey": null
}
```

**Field validation:**

| Field | Required | Validation |
|---|---|---|
| `issueType` | Yes | One of: `missing_items`, `wrong_items`, `damaged_items`, `never_received`, `charged_incorrectly`, `other` |
| `affectedItemIds` | Required for `missing_items`, `wrong_items`, `damaged_items` | Array of `order_items.id` values that belong to this order. Must be non-empty for item-specific issues. |
| `description` | Required for `never_received`, `charged_incorrectly`, `other`. Optional for item-specific issues. | String, max 500 chars. |
| `photoKey` | Optional | S3 key from a prior upload via pre-signed URL. Only meaningful for `damaged_items`. |

**Processing:**
1. Validate all fields and constraints
2. Validate order belongs to customer, within 7 days of delivery, no existing report
3. Insert `order_issues` row
4. Insert `order_flags` row with `flag_type = 'customer_[issueType]'`
5. If `issueType = 'never_received'` AND `orders.contact_delivery = false`:
   - Set `customers.no_contact_revoked = true`, `no_contact_revoked_at = NOW()`, `no_contact_revoke_reason = 'Reported never received delivery'`
   - Include this action in the flag details
6. Calculate suggested credit amount (sum of `actual_price` for affected items) and include in the flag details for admin reference — do NOT auto-issue the credit
7. Return success

**Response:**
```json
{
  "reported": true,
  "issueId": 15,
  "message": "Report submitted. We'll review within 24 hours."
}
```

**Error Responses:**
- `400` — Validation error (invalid issue type, missing required fields, invalid item IDs)
- `403` — Reporting window closed or order doesn't belong to customer
- `409` — Issue already reported for this order

---

### Database Tables

```sql
-- ============================================================
-- DELIVERY RATINGS
-- ============================================================
CREATE TABLE delivery_ratings (
  id              SERIAL PRIMARY KEY,
  order_id        INTEGER NOT NULL REFERENCES orders(id),
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  driver_id       INTEGER,  -- FK to drivers (defined in D-01)
  rating          VARCHAR(10) NOT NULL,  -- 'positive' or 'negative'
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(order_id)  -- one rating per order
);

CREATE INDEX idx_delivery_ratings_driver ON delivery_ratings(driver_id);
CREATE INDEX idx_delivery_ratings_customer ON delivery_ratings(customer_id);

-- ============================================================
-- ORDER ISSUES
-- ============================================================
CREATE TABLE order_issues (
  id                  SERIAL PRIMARY KEY,
  order_id            INTEGER NOT NULL REFERENCES orders(id),
  customer_id         INTEGER NOT NULL REFERENCES customers(id),
  issue_type          VARCHAR(30) NOT NULL,
    -- 'missing_items','wrong_items','damaged_items',
    -- 'never_received','charged_incorrectly','other'
  affected_item_ids   INTEGER[],              -- array of order_items.id values
  description         VARCHAR(500),
  photo_key           VARCHAR(255),
  suggested_credit    DECIMAL(8,2),           -- auto-calculated, admin reviews before issuing
  resolution_status   VARCHAR(20) DEFAULT 'pending',
    -- 'pending','partial_credit','full_refund','no_action','contacted'
  resolved_by         VARCHAR(100),
  resolved_at         TIMESTAMPTZ,
  resolution_note     VARCHAR(500),
  refund_amount       DECIMAL(8,2),           -- actual refund issued, if any
  stripe_refund_id    VARCHAR(100),
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(order_id)  -- one report per order
);

CREATE INDEX idx_order_issues_pending ON order_issues(resolution_status) WHERE resolution_status = 'pending';

-- ============================================================
-- CUSTOMER COLUMNS (delivery timestamp for time-gating)
-- ============================================================
-- Note: orders.delivered_at needs to be added to the orders table
-- to support time-gated actions. This is a TIMESTAMPTZ column
-- set when the driver confirms delivery.
ALTER TABLE orders ADD COLUMN delivered_at TIMESTAMPTZ;
CREATE INDEX idx_orders_delivered ON orders(delivered_at) WHERE delivered_at IS NOT NULL;
```

### Tables Referenced (Defined Elsewhere)
- `customers` — C-01
- `orders`, `order_items`, `order_flags` — C-04
- `stores`, `producers` — C-01
- `products` — C-03
- `batch_participants`, `weekly_batches` — C-06

---

## Config Values

| Config Key | Default | Description |
|---|---|---|
| `rating_window_days` | 7 | Days after delivery that rating is available |
| `issue_report_window_days` | 7 | Days after delivery that issue reporting is available |
| `tip_adjustment_window_hours` | 24 | Hours after delivery for tip increase (shared with C-04) |
| `reorder_seasonal_months` | `[11, 12]` | Months (1-12) considered "seasonal" for one-time item detection |
| `reorder_regular_threshold` | 2 | Minimum order count for an item to be classified as "regular" |

---

## Responsive Breakpoints

| Breakpoint | Order History | Order Detail | Reorder Page |
|---|---|---|---|
| < 768px | Full-width stacked cards | Full-screen page | Full-screen page |
| 768px - 1024px | Full-width stacked cards | Full-screen page | Full-screen page |
| > 1024px | Centered column (max 720px) | Centered column (max 720px) | Two-column (items + sidebar summary) |

---

## ECS Fargate Notes

- `GET /api/orders` with offset pagination is simple and efficient at this scale. If order volume reaches thousands per customer (unlikely), switch to cursor-based pagination.
- `GET /api/orders/:orderId/reorder` is the most expensive endpoint in this spec — it queries order history to count item frequencies. At < 100 orders per customer per store, this is instant. No caching needed.
- Issue report photos are uploaded directly to S3 via pre-signed URL (same as profile photos in C-01). The API never handles the file upload — it only generates the pre-signed URL and stores the resulting S3 key.
- Stripe refund API calls (triggered by admin resolution) use idempotency keys: `order-refund-{issueId}`.

## Third-Party API Costs

| Service | Operation | Cost | Monthly Estimate |
|---|---|---|---|
| S3 | Issue report photo storage | $0.023/GB | < $0.01 |
| Stripe | Refund processing | No additional fee | $0 |
| **Total** | | | **< $0.01/month** |

---

## Self-Audit (Per AUDIT-FRAMEWORK.md)

Audit performed during authoring against all applicable categories:

### Security (Category 1)
- ✅ All endpoints require authentication via session cookie
- ✅ Every order query filters by `customer_id` from session — no cross-customer data access
- ✅ Stripe refund uses idempotency keys
- ✅ No PII in API responses beyond what's necessary (driver first name only, card brand + last4 only)
- ✅ Issue report photos uploaded via S3 pre-signed URL — file never touches API server
- ✅ Time-gating for actions (rating, issues, tip) validated server-side, not client-side

### Data Integrity (Category 2)
- ✅ Unique constraints on `delivery_ratings(order_id)` and `order_issues(order_id)` — one per order
- ✅ `delivered_at` column added to orders table for time-gating calculations
- ✅ Issue resolution tracks who resolved, when, and what action was taken (audit trail)
- ✅ No-contact revocation on "never received" is logged with reason

### Completeness (Category 3)
- ✅ Empty states defined (no orders, all items unavailable for reorder)
- ✅ Loading states defined (skeleton cards for order history)
- ✅ Error states defined (order not found, network error, rating window closed)
- ✅ Edge cases addressed (double-tap on rating, reorder from deactivated store, reorder with no batch, old order fuzzy matching, cash order issue reporting)
- ✅ Every API endpoint has full request/response contracts with validation rules
- ✅ Every database table has complete schema with indexes

### Cross-Document Consistency (Category 3.3)
- ✅ Order status values match C-04 exactly (all 13 statuses mapped in display table)
- ✅ `order_items` fulfillment fields (`actual_price`, `substituted_with`, `fulfillment_notes`, `fulfilled`) match C-04 schema
- ✅ `order_flags` reused from C-04 for both negative ratings and issue reports
- ✅ Tip adjustment references C-04's mechanism and constraints
- ✅ No-contact revocation references C-04's customer column definitions
- ✅ Reorder route check references C-06 batch system

### Maintainability (Category 4)
- ✅ All time windows (rating, issues, tip) are in config tables, not hardcoded
- ✅ Seasonal month detection is configurable
- ✅ Reorder "regular" threshold is configurable
- ✅ Issue resolution includes who resolved and why (debuggable audit trail)

### Potential Gaps Identified
- ⚠️ **Missing: Notification for issue resolution.** When admin resolves an issue (credits issued, refund processed), the customer should be notified via SMS. Text: "Your report for Order #142 has been reviewed. A credit of $X.XX has been applied to your card." This needs to be added to the admin resolution flow (A-series spec).
- ⚠️ **Missing: `delivered_at` population.** The `delivered_at` column is defined here but the logic that SETS it (driver confirms delivery) lives in D-02 (Driver Pickup spec, not yet written). Flagging as a cross-spec dependency.
- ⚠️ **Missing: Reorder for bulk special orders.** Bulk specials from C-06 are one-time distribution routes. If a customer tries to reorder a bulk special order, the items won't match any current batch or catalog. The reorder endpoint should detect `orders` linked to a `bulk_special_participants` record and show: "This was a one-time bulk special. Check the Community tab for current specials." Not yet handled.
