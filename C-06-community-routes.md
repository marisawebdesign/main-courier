# C-06: Community & Route Building

> **Spec Version:** 2.0
> **Status:** Complete — Ready for Development
> **Dependencies:** C-01 (authenticated session), C-02 (home screen Community toggle), PostgreSQL
> **Relates to:** C-02 (Home Screen — Community toggle and store cards), C-03 (Shopping List Builder / Product Catalog), D-02 (Driver Pickup — consolidated shopping lists), S-01 (Store Dashboard — deal/bulk special posting)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md.

---

## Overview

The Community tab is a logistics planning tool that enables customers to anonymously coordinate batch delivery routes for ALL stores in their zone — both unpartnered big box stores (Walmart, Hannaford) and partnered local businesses. It also surfaces bulk specials from local vendors: one-time distribution routes organized around specific products that partnered businesses need to move in volume.

This is NOT a social feature. Nobody meets, nobody chats, nobody sees names or addresses. The system handles all matching and coordination behind the scenes. Customers see counts, fees, and anonymous participant profiles — nothing more.

### The Core Problem

A solo personal shopping trip to Walmart costs ~$15 in shopping fees because a driver spends 45+ minutes in the store for one customer. That's unaffordable for the target population. But if 6 people in the same zone all need Walmart groceries the same week, the driver goes once, shops a consolidated list in ~90 minutes, and the fee splits to ~$3 per person.

The Community tab is the mechanism that gets those 6 people to coordinate without knowing each other. It also gives local vendors a channel to move surplus product in volume through dedicated one-time distribution routes.

### Design Principles

1. **Anonymity is absolute between customers.** No names, no addresses, no phone numbers, no exact locations shared. Ever.
2. **The route is the unit of coordination, not the people.** Customers interact with routes, not with each other.
3. **Weekly opt-in, never auto-recurring.** Each week requires an active "I'm in" tap. No surprise charges.
4. **Frequency-aware planning.** The system knows who shops weekly vs. biweekly vs. monthly and plans accordingly.
5. **One-tap re-engagement.** Rejoining last week's route with your previous shopping list is a single tap.
6. **Transparency on fees.** The customer always sees the current batch fee and how many participants drive it.
7. **Bulk specials reward participation.** Only customers with proven route participation can access bulk special routes — this rewards loyalty and ensures reliable commitment.
8. **All store types benefit.** Partnered stores and producers get route coordination and bulk special distribution. Unpartnered stores get the batch shopping system.

---

## How Routes Work: The Weekly Lifecycle

### The Cycle

```
Monday morning (automated):
  System creates new weekly batches for routes that ran last week
  Notifies last week's participants based on their frequency preference
  Pre-stages their previous shopping lists for one-tap reuse

Monday → Cutoff (e.g., Wednesday 8pm):
  Customers rejoin, new customers join
  Shopping lists are built and submitted
  Batch participant count and fee update in real time

Cutoff passes:
  Orders lock — no more changes
  System generates consolidated shopping list for driver
  If minimum threshold not met → batch cancelled, participants notified, nobody charged

Delivery day:
  Driver shops consolidated list at the store
  Driver delivers to all participants on the route
  Batch marked complete

Following Monday:
  Cycle restarts
  System suggests re-forming the same route
```

### Route Formation Stages

Routes progress through maturity stages tracked in the `route_history` table:

| Stage | Criteria | What Happens |
|---|---|---|
| **Interest** | People expressing interest, threshold not met | No batch runs. Route appears under "Building Routes" in the Community tab. |
| **First Batch** | Threshold met for the first time | First weekly batch is created. Listed under "This Week." Driver (partner initially) handles it. |
| **Recurring** | 3+ consecutive weeks with enough participants | Route is considered established. System auto-suggests re-formation each week. |
| **Dedicated Driver** | 6-8+ consecutive weeks, 8+ average participants | Admin dashboard flags this route as ready for a dedicated driver hire. |

Maturity stages are updated by the weekly automated job and are visible in the admin dashboard. They are NOT visible to customers — customers see batch status (forming, confirmed, etc.) not maturity labels.

---

## Customer Frequency Preferences

When a customer expresses interest in a route, they indicate how often they plan to participate:

| Frequency | Meaning | Notification Cadence |
|---|---|---|
| **Weekly** | Plan to order every week | Notified every Monday when the route re-forms |
| **Every other week** | Roughly twice a month | Notified on their "on" weeks (2+ weeks since last order). Softer "route is available" on off weeks. |
| **Monthly** | Once a month, usually a bigger order | Notified when 3+ weeks since last order. Not contacted otherwise. |
| **Flexible** | No set schedule, joins when needed | Notified every week with lower-priority framing |

Frequency is a planning signal, not a binding commitment. Customers can change their frequency at any time with one tap. The system uses frequency to:

1. **Estimate weekly participation** for route viability
2. **Send notifications at the right cadence** (don't pester monthly shoppers every week)
3. **Display meaningful route composition** to other participants

### Expected Participation Calculation

```
expected_weekly_participants =
  count(weekly, active) × 1.0 +
  count(biweekly, active) × 0.5 +
  count(monthly, active) × 0.25 +
  count(flexible, active) × 0.3
```

Multipliers are stored in `system_config` and adjustable through admin. The result is used to:
- Determine if a building route is close to viable
- Estimate the shopping fee for display ("Fee at ~5 participants: $3.00")
- Inform the admin dashboard about route health

---

## Anonymous Participant Profiles

When a customer views a route's details, they see a list of anonymous participant rows. Each row shows three data points:

| Data Point | What It Shows | Why It's Useful |
|---|---|---|
| **Frequency** | Weekly, biweekly, monthly, flexible | Tells the customer how reliable this route is |
| **Tenure** | How many weeks this person has been on the route | Longer tenure = more likely to keep showing up |
| **Typical order size** | Small (under $50), medium ($50-100), large ($100+) | Gives a sense of route scale and fee dynamics |

**What is NOT shown:** Name, address, phone number, exact location, specific items ordered, payment info, delivery preference (door/hub).

**Typical order size** is computed automatically from order history — not self-reported. The system averages the customer's last 4 order totals from this store and categorizes. If they have no history yet (new participant), it shows "New" instead of a size.

**The customer's own row** is highlighted so they can see themselves in the group:

```
👤 Weekly · 8 weeks · Medium
👤 Weekly · 5 weeks · Large
👤 Biweekly · 6 weeks · Small
👤 Biweekly · 3 weeks · Medium
👤 Monthly · 2 weeks · Large
⭐ You · Weekly · 4 weeks · Medium
```

---

## Preset Route Feedback

Instead of freeform notes, customers can select from preset logistical feedback options. These are pure logistics signals — no open-ended messaging. Each preset can be selected (voted for) by any participant. Selecting a preset IS the vote — there's no separate upvote action.

### Preset Categories

**Timing:**
- "I'd prefer an earlier cutoff"
- "A later cutoff would help me"
- "Could we try a different delivery day?"

**Route Health:**
- "This route works great for me"
- "I might need to reduce my frequency soon"
- "This route needs more participants — please spread the word"

**Shopping:**
- "I'm interested in splitting bulk items with others"
- "Produce quality has been inconsistent — please be selective"
- "I usually need brand-specific items — substitutions are tricky for me"

### How Presets Display

```
📋 Route Feedback
──────────────────────────────

"This route works great for me"        5 ✓
"I'm interested in splitting bulk"     3 ✓
"A later cutoff would help me"         2
"Produce quality inconsistent"         1

[Select feedback →]
```

Each preset shows the text and the selection count. If the customer has selected it, it shows a checkmark. Tapping opens the full list of presets organized by category — the customer taps to select/deselect. One tap per preset, toggle behavior.

Presets with zero selections don't display in the route view — only presets that at least one person has selected appear. This keeps the view clean.

### Admin Visibility

Your partner sees the feedback in the admin dashboard with selection counts. This informs route adjustments — if 4 of 6 people select "a later cutoff would help me," she knows to consider adjusting the schedule. She can also add, edit, or deactivate presets through the admin config without code changes.

---

## Customer Flows

### Flow 1: Expressing Interest in a New Route

Customer taps an unpartnered store pin on the map (State B or C from C-02) and is routed to the Community tab.

```
Screen: Express Interest

🏪 Walmart Supercenter · Houlton

Be one of the first to plan a
delivery route for this store.

When [X] neighbors join, a weekly
batch route will start — and
everyone saves on shopping fees.

How often would you order?
  ○ Weekly
  ○ Every other week
  ○ Monthly
  ○ Flexible / not sure

What day works best?
  ○ Monday
  ○ Tuesday
  ○ Wednesday
  ○ Thursday
  ○ Friday
  ○ No preference

[I'm interested →]
```

**On submit:** Creates a `route_interests` row. Updates the interest count for this store/zone. If the threshold is now met, triggers route creation (see Flow 3). If not, the customer sees:

```
You're in! 3 of 5 neighbors needed.

We'll notify you when the route
has enough participants to start.

In the meantime, you can order
on-demand for a $15 shopping fee.
```

### Flow 2: Rejoining a Re-Forming Route (Returning Participant)

Customer receives a notification (SMS or in-app) on Monday:

```
Thursday's Walmart route is
re-forming. 4 of 6 are back.

Fee at 4: $3.75

[I'm in — load my list →]
[Skip this week]
[Change my frequency]
```

**"I'm in — load my list":** Creates a `batch_participants` row with status `committed`. Loads their shopping list from last week's batch participation (items are copied from the previous week's `order_items` linked to their previous `batch_participants` record). Customer lands on their pre-filled shopping list and can edit, add, remove items, then submit.

**"Skip this week":** No `batch_participants` row created. No further action. They'll be notified again next week (according to their frequency preference).

**"Change my frequency":** Opens a simple frequency selector (weekly/biweekly/monthly/flexible). Updates their `route_interests.frequency`. Adjusts future notification cadence.

### Flow 3: First-Time Route Creation (Threshold Met)

When enough interest accumulates for a store/zone/day combination:

1. System creates a `weekly_batches` row with status `forming`
2. System creates a `route_history` row with maturity `first_batch`
3. All interested customers for this store/zone receive notification:

```
Great news! A Thursday Walmart
route is starting this week.

5 neighbors are interested.
Shopping fee: $3.00

[I'm in — build my list →]
```

4. Customers who tap "I'm in" become `batch_participants` and proceed to build their shopping list
5. Route appears under "This Week" in the Community tab for all zone customers

### Flow 4: Joining an Existing Route (New Participant)

Customer sees an active route in the Community tab or through an unpartnered store card (State A from C-02).

```
Thursday Walmart Route · Houlton
5 participants this week
Fee: $3.00

Route active for 8 weeks

👤 Weekly · 8 weeks · Medium
👤 Weekly · 5 weeks · Large
👤 Biweekly · 6 weeks · Small
👤 Biweekly · 3 weeks · Medium
👤 Monthly · 2 weeks · Large

How often would you order?
  ○ Weekly
  ○ Every other week
  ○ Monthly
  ○ Flexible

[Join this week →]
```

**On submit:** Creates `route_interests` row (if they don't have one for this store) AND `batch_participants` row for this week's batch. Customer proceeds to build their shopping list.

### Flow 5: Batch Cutoff — Threshold Met

When the order cutoff passes and the batch has enough participants:

1. Batch status changes to `confirmed`
2. All participants with status `committed` or `list_submitted` receive confirmation:

```
Thursday's Walmart route is
confirmed! 6 participants.

Your shopping fee: $2.50
Your driver will shop Thursday
morning and deliver by afternoon.

Your list: 12 items
[Review my list →]
```

3. System generates consolidated shopping list for the driver (detailed in D-02)

### Flow 6: Batch Cutoff — Threshold NOT Met

When the cutoff passes and the batch doesn't have enough participants:

1. Batch status changes to `cancelled`
2. All committed participants are notified:

```
Thursday's Walmart route didn't
get enough participants this week.
(3 of 5 needed)

Your shopping list is saved for
next week.

You can still order on-demand:
[Order Now — $15 shopping fee →]
```

3. Nobody is charged. Shopping lists are preserved in the system for next week's attempt.
4. `route_history.consecutive_weeks` resets to 0 if this was a recurring route that failed to form.

---

## Community Tab — Full UI Specification

This is the content that appears in the floating right panel (desktop) or bottom sheet (mobile) when the Community toggle is active on the home screen.

### Desktop Panel Content

```
┌──────────────────────┐
│  ✅ Community         │
│                      │
│  ════════════════    │
│                      │
│  📅 This Week        │
│  ──────────────────  │
│                      │
│  Walmart · Houlton   │
│  Thu · Re-forming    │
│  5 of 7 back so far  │
│  👤👤👤👤 Wkly       │
│  👤 Biwkly           │
│  Fee: $3.00          │
│  ✓ You're in         │
│  [Edit list →]       │
│  [Route details →]   │
│                      │
│  Hannaford · Lincoln │
│  Wed · Re-forming    │
│  3 of 5 back so far  │
│  👤👤 Wkly           │
│  👤 Monthly          │
│  Fee: $5.00          │
│  [I'm in →]          │
│                      │
│  Calais IGA · Calais │
│  Wed · Scheduled     │
│  4 neighbors ordering│
│  Delivery: $4.99     │
│  [Browse & order →]  │
│                      │
│  ──────────────────  │
│                      │
│  🥔 Bulk Specials    │
│  ──────────────────  │
│                      │
│  Wilson Farm         │
│  Potatoes · $0.50/lb │
│  Sat · 6 of 8 needed│
│  Extended radius     │
│  [Commit — qty →]   │
│                      │
│  Sue's Sourdough     │
│  Bread loaves · $3ea │
│  Thu · 10 of 12     │
│  [Commit — qty →]   │
│                      │
│  ──────────────────  │
│                      │
│  📋 Building Routes  │
│  ──────────────────  │
│                      │
│  CVS · Houlton       │
│  4 of 5 needed       │
│  👤👤 Wkly 👤 Flex   │
│  👤 Monthly          │
│  [Join →]            │
│                      │
│  Target · Bangor     │
│  2 of 5 needed       │
│  👤 Wkly 👤 Flex     │
│  [Express interest →]│
│                      │
│  ──────────────────  │
│                      │
│  🌾 Local Producers  │
│  7 producers in your │
│  area                │
│  [Browse Producers →]│
│                      │
└──────────────────────┘
```

### Section: This Week

Shows batch routes that are currently forming for the current week. Includes BOTH unpartnered store batch routes AND partnered store scheduled routes — all delivery activity in one place.

**Unpartnered store route cards (Walmart, Hannaford, etc.):**
- Store name and town
- Delivery day
- Re-formation status: "X of Y back so far"
- Anonymous frequency breakdown
- Current shopping fee at current batch size
- Customer's status: "You're in" (with "Edit list") or "I'm in" button
- "Route details" link

**Partnered store route cards (Calais IGA, etc.):**
- Store name and town
- Delivery day (from zone_delivery_days schedule)
- Neighbor count: "X neighbors ordering"
- Delivery fee (from fee_tiers, not shopping fee — no personal shopping needed)
- "Browse & order →" links to their catalog (C-03/C-09)
- No shopping list builder — partnered stores have their own catalog

**Sorting:** Routes the customer is participating in first. Then routes they have interest in. Then other available routes. Within each group, unpartnered and partnered are interleaved by delivery day (Monday routes before Thursday routes).

### Section: Bulk Specials

One-time distribution routes from partnered local vendors (stores and producers). **Only visible to eligible customers** who have met the participation threshold (configurable, default: 2+ completed route participations).

**For eligible customers:** Shows available bulk specials with product info, vendor name, price, participant count vs. minimum, delivery day, and a "Commit" button.

**For ineligible customers:** Shows a locked state:
```
🥔 Bulk Specials
──────────────────

Bulk specials are available to
active community participants.

Complete [X] more route deliveries
to unlock bulk specials.

[See available routes →]
```

This creates a clear incentive: participate in routes → unlock bulk specials → get better deals.

**Per-special card contents:**
- Vendor name (store or producer)
- Product name and price per unit
- Delivery day
- Participant count vs. minimum: "X of Y needed"
- "Extended radius" badge if the special has a wider delivery area than normal
- "Commit — qty" button (opens quantity selector before committing)

**Sorting:** Specials closest to their cutoff date first. Specials closest to their participant threshold second.

### Section: Building Routes

Shows store/zone combinations where interest exists but the threshold hasn't been met. Unchanged from previous spec.

**Per-route card contents:**
- Store name and town
- Interest count vs. threshold: "X of Y needed"
- Anonymous frequency breakdown of current interests
- "Join" or "Express interest" button

**Sorting:** Routes closest to threshold first.

### Section: Local Producers

Simple link to browse Tier 1 cottage food producers and farms. "Browse Producers" opens the list view filtered to producers.

---

## Route Detail View

Accessed by tapping "Route details" on any active route card. This can be a modal, a slide-over panel, or a separate page — choose based on implementation.

```
┌─────────────────────────────────┐
│  ← Back                        │
│                                 │
│  Thursday Walmart Route         │
│  Houlton · 35 mi                │
│  Active for 8 consecutive weeks │
│                                 │
│  ── This Week ────────────────  │
│  Status: Forming                │
│  Participants: 5                │
│  Cutoff: Wed 8pm (1d 6h left)  │
│  Shopping fee: $3.00            │
│                                 │
│  ── Participants ─────────────  │
│  👤 Weekly · 8 weeks · Medium   │
│  👤 Weekly · 8 weeks · Large    │
│  👤 Weekly · 5 weeks · Medium   │
│  👤 Biweekly · 6 weeks · Small  │
│  👤 Monthly · 2 weeks · Large   │
│  ⭐ You · Weekly · 4 wks · Med  │
│                                 │
│  ── Route Feedback ───────────  │
│  "This route works great"  5 ✓  │
│  "Interested in bulk split" 3   │
│  "Later cutoff would help"  2   │
│  [Select feedback →]            │
│                                 │
│  ── Route Stats ──────────────  │
│  Avg participants: 6.2          │
│  Avg fee: $2.80                 │
│  Weeks active: 8                │
│  Your participation: 4 of 8 wks│
│                                 │
└─────────────────────────────────┘
```

### Route Stats Section

Computed from `route_history` and `batch_participants` history:

- **Avg participants:** Rolling average over the route's lifetime
- **Avg fee:** Rolling average shopping fee
- **Weeks active:** `route_history.consecutive_weeks`
- **Your participation:** Count of batches this customer completed out of total batches run

These stats help the customer assess route reliability — a route with 8 consecutive weeks and 6+ average participants is clearly stable.

---

## Notification Flows

All notifications sent via Twilio SMS (transactional) or in-app notification. The system determines the channel based on user preferences (defined in C-11 Account & Settings). SMS is the default and fallback.

### Weekly Route Re-Formation (Monday morning)

Sent to last week's participants based on their frequency:

**Weekly participants (always notified):**
```
Thu Walmart route is re-forming.
[X] of [Y] neighbors are back.
Fee: $[X.XX]

Tap to rejoin: [link]
```

**Biweekly participants (on their "on" week):**
```
It's been 2 weeks — Thu Walmart
route is forming. [X] neighbors in.
Fee: $[X.XX]

Tap to rejoin: [link]
```

**Biweekly participants (on their "off" week):**
```
Thu Walmart route is available
this week if you need it.
[X] neighbors in. Fee: $[X.XX]

Join: [link]
```

**Monthly participants (3+ weeks since last order):**
```
Monthly check-in: Thu Walmart
route has [X] neighbors this week.
Fee: $[X.XX]

Join: [link]
```

**Monthly participants (less than 3 weeks):** No notification sent.

**Flexible participants (always notified, softer tone):**
```
Thu Walmart route: [X] neighbors,
$[X.XX] fee. Join if you need to.

[link]
```

### Route Threshold Met (First-Time)

Sent to all interested parties when a building route hits its threshold:

```
A Thu Walmart route is starting!
[X] neighbors are in. Fee: $[X.XX]

Build your list: [link]
```

### Batch Confirmed (Cutoff Passed, Threshold Met)

Sent to all committed participants:

```
Thu Walmart route confirmed!
[X] participants. Fee: $[X.XX]
Your list: [X] items.
Delivery: Thursday afternoon.
```

### Batch Cancelled (Cutoff Passed, Threshold NOT Met)

Sent to all committed participants:

```
Thu Walmart route didn't get
enough participants ([X] of [Y]).
Your list is saved for next week.
No charge.
```

### Dormancy Warning

Sent to participants who haven't ordered in 2 consecutive expected windows:

```
We noticed you haven't joined the
Thu Walmart route in a while.

Want to stay on the notification
list? Reply YES to stay or we'll
pause your notifications.
```

If no reply within 48 hours → mark as dormant, stop notifications.

---

## Bulk Specials System

Bulk specials are one-time distribution routes created by partnered vendors (stores and producers) to move surplus or seasonal product in volume. They function like a flash community route organized around a specific product rather than a specific store.

### How Bulk Specials Differ from Regular Routes

| Aspect | Regular Community Route | Bulk Special Route |
|---|---|---|
| Created by | System (weekly reformation) or community interest | Vendor (through their dashboard) |
| Frequency | Weekly recurring | One-time |
| Product scope | Full shopping list from a store | Single product or small set from one vendor |
| Radius | Standard zone radius | Extended (configurable per special by vendor) |
| Eligibility | Anyone in the zone | Previous route participants only (configurable threshold) |
| Minimum participants | From system config (default 5) | Set by vendor per special |
| Fee structure | Shopping fee split across batch | Route/delivery fee only — vendor prepares the product, no personal shopping labor |
| Vendor involvement | None (unpartnered) or passive catalog (partnered) | Active — vendor creates the special, sets terms, prepares the product |

### Eligibility

Only customers with proven route participation history can see and commit to bulk specials. This rewards loyal participants and ensures reliable commitment.

**Eligibility check:**
```sql
SELECT COUNT(*) as completed_participations
FROM batch_participants bp
JOIN weekly_batches wb ON wb.id = bp.batch_id
WHERE bp.customer_id = $1
AND bp.status = 'complete'
AND wb.status = 'complete';
```

Compare against `bulk_special_eligibility_threshold` (default: 2, configurable in admin). Start low to maximize early adoption, increase if needed.

### Bulk Special Lifecycle

```
Vendor creates a bulk special via their dashboard
  │
  ▼
Special appears in Community tab for eligible customers
Notification sent to eligible customers in zone + extended radius
  │
  ▼
Customers commit with their desired quantity
Participant count and remaining quantity update in real time
  │
  ▼
Cutoff passes:
  ├── Minimum met → Special confirmed
  │   Vendor notified to prepare product
  │   Driver assigned for pickup and distribution
  │   Participants charged
  │
  └── Minimum NOT met → Special cancelled
      All participants notified, nobody charged
      Vendor notified
  │
  ▼
Distribution day:
  Driver picks up from vendor, delivers to all participants
  Special marked complete
```

### What the Vendor Posts (Dashboard Side)

Partnered stores and producers create bulk specials through a "Bulk Specials" section in their dashboard (detailed in S-01/A-04):

- Product name and description
- Photo (optional)
- Price per unit and unit type (lbs / each / dozen / bushel / pint / quart)
- Minimum and maximum order per customer
- Total available quantity
- Preferred distribution day
- Prep time needed (same day / 1 day / 2 days)
- Extended radius in miles (50 / 60 / 75)
- Minimum participants to run
- Cutoff date/time
- Optional note to customers (300 chars)

After posting, the vendor sees the special's live status in their dashboard: committed participants, total quantity claimed, whether minimum is met, time until cutoff.

### Customer Flow: Committing to a Bulk Special

**Step 1:** Eligible customer sees the special in Community tab "Bulk Specials" section.

**Step 2:** Taps "Commit — qty" which opens quantity selector:

```
Wilson Farm — Russet Potatoes
$0.50/lb · Min 10 lbs · Max 25 lbs
142 lbs remaining

How many lbs?
  [ - ]  15  [ + ]

Subtotal: $7.50
Route fee: $2.50
────────────
Total: $10.00

Distribution: Saturday Mar 29
📍 Door delivery or hub pickup

"Huge harvest this week —
 stock up while they last!"

Runs if 8+ people commit.
Currently 6 committed.

[Commit to this special →]
```

**Step 3:** On commit: participant row created, quantity decremented, participant count updated. If this commit meets the minimum, all participants notified the special is confirmed.

**Step 4:** If minimum met by cutoff → vendor prepares, driver distributes, participants charged. If not → cancelled, nobody charged.

### Customer Flow: Ineligible Customer

```
🥔 Bulk Specials
──────────────────

Bulk specials are exclusive to
active community participants.

You've completed 1 of 2 required
route deliveries.

Complete 1 more to unlock access
to bulk deals from local vendors.

[See available routes →]
```

Progress indicator motivates continued participation.

### Bulk Special Notifications

**New special posted (eligible customers in zone + extended radius):**
```
New bulk special from Wilson Farm!
Potatoes $0.50/lb (min 10 lbs)
Available Saturday. 8 needed.

Commit: [link]
```

**Special confirmed (sent to committed participants):**
```
Wilson Farm potato special confirmed!
[X] participants. Your order: 15 lbs.
Delivery: Saturday afternoon.
Total charge: $10.00
```

**Special cancelled (sent to committed participants):**
```
Wilson Farm potato special didn't
get enough participants ([X] of [Y]).
No charge.
```

### Bulk Special Economics

Bulk specials are high-margin for the platform because:
- No personal shopping labor — vendor prepares everything
- High volume per stop — one pickup at the vendor, distribute to 10+ customers
- The route fee covers delivery cost
- Vendor sets the product price, platform adds delivery fee on top

Example: 200 lb potato special with 10 participants each ordering 20 lbs:
- Product revenue to vendor: 10 × (20 × $0.50) = $100
- Route fee revenue to platform: 10 × $2.50 = $25
- Driver time: ~1 hour for pickup + delivery
- Platform profit on delivery side: $25 minus driver cost

And it drives route participation for regular weekly routes (the eligibility gate), which is the core of the business.

---

## Backend Specification

### API Endpoints

---

#### `GET /api/community`

**Authentication:** Required (session cookie)

**Purpose:** Return all community route data for the customer's zone. This powers the Community toggle panel content.

**Processing:**
1. Get customer's zone_id from session
2. Check customer's bulk special eligibility (count completed participations)
3. Query this week's active batches for the zone (from `weekly_batches` WHERE week_start = current week AND status IN forming, confirmed)
4. For each batch: get participant count, frequency breakdown, whether this customer is a participant, current shopping fee
5. Query partnered store routes for the zone (from `zone_delivery_days` + order counts for current week)
6. Query building routes (from `route_interests` grouped by store/zone/day)
7. If customer is eligible: query active bulk specials for the zone + extended radius
8. Query active producers in the zone

**Response:**
```json
{
  "thisWeek": [
    {
      "batchId": 42,
      "storeId": 1,
      "storeName": "Walmart Supercenter",
      "storeTown": "Houlton",
      "partnershipTier": "unpartnered",
      "deliveryDay": "thursday",
      "deliveryDate": "2026-03-26",
      "cutoffTime": "2026-03-25T20:00:00-04:00",
      "status": "forming",
      "participantCount": 5,
      "previousWeekCount": 7,
      "currentShoppingFee": 3.00,
      "frequencyBreakdown": {
        "weekly": 4,
        "biweekly": 1,
        "monthly": 0,
        "flexible": 0
      },
      "customer": {
        "isParticipant": true,
        "status": "committed",
        "hasList": true
      }
    },
    {
      "storeId": 3,
      "storeName": "Calais IGA",
      "storeTown": "Calais",
      "partnershipTier": "partnered",
      "deliveryDay": "wednesday",
      "deliveryDate": "2026-03-25",
      "neighborCount": 4,
      "deliveryFee": 4.99
    }
  ],
  "bulkSpecials": {
    "eligible": true,
    "eligibilityProgress": {
      "completed": 4,
      "required": 2
    },
    "specials": [
      {
        "id": 7,
        "vendorName": "Wilson Honey Farm",
        "vendorType": "producer",
        "vendorId": 3,
        "productName": "Russet Potatoes",
        "productDescription": "Fresh harvest from our farm",
        "thumbnailKey": "specials/wilson-potatoes.jpg",
        "pricePerUnit": 0.50,
        "unitType": "lb",
        "minPerCustomer": 10,
        "maxPerCustomer": 25,
        "quantityTotal": 200,
        "quantityRemaining": 142,
        "participantCount": 6,
        "minParticipants": 8,
        "deliveryDay": "saturday",
        "deliveryDate": "2026-03-29",
        "cutoffTime": "2026-03-28T12:00:00-04:00",
        "extendedRadiusMiles": 60,
        "routeFee": 2.50,
        "vendorNote": "Huge harvest — stock up while they last!",
        "customer": {
          "isCommitted": false
        }
      }
    ]
  },
  "building": [
    {
      "storeId": 5,
      "storeName": "CVS",
      "storeTown": "Houlton",
      "interestCount": 4,
      "threshold": 5,
      "frequencyBreakdown": {
        "weekly": 2,
        "biweekly": 0,
        "monthly": 1,
        "flexible": 1
      },
      "customer": {
        "hasInterest": false
      }
    }
  ],
  "producers": [
    {
      "id": 1,
      "name": "Martha's Pies",
      "town": "Danforth",
      "description": "Homemade pies and baked goods",
      "distanceMiles": 3.4
    }
  ]
}
```

**Key response details:**
- `thisWeek` now includes both unpartnered batch routes AND partnered store scheduled routes. The `partnershipTier` field tells the frontend which card type to render.
- `bulkSpecials.eligible` tells the frontend whether to show specials or the locked/progress state.
- `bulkSpecials.eligibilityProgress` shows completed vs. required participations for the progress indicator.
- If `eligible` is false, `specials` is an empty array — the backend doesn't send special details to ineligible customers.

---

#### `GET /api/community/route/:storeId`

**Authentication:** Required

**Purpose:** Return detailed route information including anonymous participant profiles, feedback, and stats. Powers the Route Detail View.

**Response:**
```json
{
  "store": {
    "id": 1,
    "name": "Walmart Supercenter",
    "town": "Houlton"
  },
  "deliveryDay": "thursday",
  "currentBatch": {
    "id": 42,
    "status": "forming",
    "participantCount": 5,
    "cutoffTime": "2026-03-25T20:00:00-04:00",
    "currentShoppingFee": 3.00
  },
  "participants": [
    {
      "frequency": "weekly",
      "tenureWeeks": 8,
      "typicalOrderSize": "medium",
      "isCurrentCustomer": false
    },
    {
      "frequency": "weekly",
      "tenureWeeks": 5,
      "typicalOrderSize": "large",
      "isCurrentCustomer": false
    },
    {
      "frequency": "biweekly",
      "tenureWeeks": 6,
      "typicalOrderSize": "small",
      "isCurrentCustomer": false
    },
    {
      "frequency": "weekly",
      "tenureWeeks": 4,
      "typicalOrderSize": "medium",
      "isCurrentCustomer": true
    }
  ],
  "feedback": [
    {
      "presetId": 1,
      "text": "This route works great for me",
      "selectionCount": 5,
      "customerSelected": true
    },
    {
      "presetId": 7,
      "text": "Interested in splitting bulk items",
      "selectionCount": 3,
      "customerSelected": false
    }
  ],
  "stats": {
    "consecutiveWeeks": 8,
    "avgParticipants": 6.2,
    "avgShoppingFee": 2.80,
    "customerParticipation": {
      "weeksParticipated": 4,
      "totalWeeksAvailable": 8
    }
  }
}
```

**Important:** The `participants` array NEVER includes customer IDs, names, or any identifying information. The `isCurrentCustomer` flag lets the frontend highlight the customer's own row. The backend filters out all PII before returning this response.

---

#### `POST /api/community/interest`

**Authentication:** Required

**Purpose:** Express interest in a store route. Creates or updates a `route_interests` row.

**Request Body:**
```json
{
  "storeId": 1,
  "frequency": "weekly",
  "preferredDay": "thursday"
}
```

**Validation:**
- `storeId` must reference an active unpartnered store in the customer's zone
- `frequency` must be one of: `weekly`, `biweekly`, `monthly`, `flexible`
- `preferredDay` must be a valid day name or `null` for no preference

**Processing:**
1. Check if `route_interests` row exists for this customer + store
   - If exists and status is `dormant`: reactivate (set status = `active`, update frequency/day)
   - If exists and status is `active`: update frequency/day
   - If doesn't exist: create new row
2. Check if total active interest for this store/zone/day now meets the threshold:
   ```sql
   SELECT COUNT(*) FROM route_interests
   WHERE store_id = $1 AND zone_id = $2 AND status = 'active'
   AND (preferred_day = $3 OR preferred_day IS NULL);
   ```
3. If threshold met AND no `weekly_batches` row exists for this week:
   - Create `weekly_batches` row with status `forming`
   - Create or update `route_history` row
   - Notify all interested customers (trigger Twilio SMS via notification queue)

**Response:**
```json
{
  "interest": {
    "storeId": 1,
    "frequency": "weekly",
    "preferredDay": "thursday",
    "status": "active"
  },
  "routeState": {
    "interestCount": 5,
    "threshold": 5,
    "thresholdMet": true,
    "batchCreated": true,
    "batchId": 42
  }
}
```

---

#### `POST /api/community/batch/:batchId/join`

**Authentication:** Required

**Purpose:** Join a specific weekly batch (commit for this week).

**Request Body:**
```json
{
  "loadPreviousList": true
}
```

**Processing:**
1. Validate batch exists, is in `forming` status, and belongs to customer's zone
2. Validate customer doesn't already have a `batch_participants` row for this batch
3. Create `batch_participants` row with status `committed`
4. If `loadPreviousList` is true, find the customer's most recent `batch_participants` entry for this store/zone and copy their order items as a draft shopping list (linked to the new participation record)
5. Increment `weekly_batches.participant_count`
6. Recalculate and update shopping fee for all participants in this batch

**Response:**
```json
{
  "participation": {
    "batchId": 42,
    "status": "committed",
    "shoppingFee": 2.73,
    "previousListLoaded": true,
    "previousItemCount": 12
  }
}
```

---

#### `POST /api/community/batch/:batchId/skip`

**Authentication:** Required

**Purpose:** Skip this week's batch (for returning participants who received a notification).

**Processing:**
1. If a `batch_participants` row exists with status `committed`: validate cutoff hasn't passed, then update status to `skipped`, decrement participant count, recalculate fees
2. If no row exists: no action needed (they just don't join)

**Response:**
```json
{
  "skipped": true
}
```

---

#### `PATCH /api/community/interest/:storeId`

**Authentication:** Required

**Purpose:** Update frequency or preferred day for an existing interest.

**Request Body:**
```json
{
  "frequency": "biweekly",
  "preferredDay": "wednesday"
}
```

**Processing:** Update the `route_interests` row for this customer + store.

---

#### `POST /api/community/route/:storeId/feedback`

**Authentication:** Required

**Purpose:** Select or deselect a preset feedback option for a route.

**Request Body:**
```json
{
  "presetId": 7,
  "selected": true
}
```

**Processing:**
- If `selected: true`: insert `route_feedback` row (or no-op if already exists due to unique constraint)
- If `selected: false`: delete `route_feedback` row

**Response:**
```json
{
  "presetId": 7,
  "selected": true,
  "totalSelections": 4
}
```

---

#### `GET /api/community/bulk-specials`

**Authentication:** Required

**Purpose:** Return all active bulk specials visible to this customer. Checks eligibility and returns appropriate response.

**Processing:**
1. Check eligibility (count completed batch participations)
2. If ineligible: return eligibility progress only, no special details
3. If eligible: query active bulk specials where the customer's location is within the special's extended radius and the special hasn't expired or sold out

**Response (eligible):**
```json
{
  "eligible": true,
  "eligibilityProgress": { "completed": 4, "required": 2 },
  "specials": [
    {
      "id": 7,
      "vendorName": "Wilson Honey Farm",
      "vendorType": "producer",
      "vendorId": 3,
      "productName": "Russet Potatoes",
      "productDescription": "Fresh harvest from our farm",
      "thumbnailKey": "specials/wilson-potatoes.jpg",
      "pricePerUnit": 0.50,
      "unitType": "lb",
      "minPerCustomer": 10,
      "maxPerCustomer": 25,
      "quantityRemaining": 142,
      "participantCount": 6,
      "minParticipants": 8,
      "deliveryDate": "2026-03-29",
      "cutoffTime": "2026-03-28T12:00:00-04:00",
      "extendedRadiusMiles": 60,
      "routeFee": 2.50,
      "vendorNote": "Huge harvest — stock up!",
      "customer": { "isCommitted": false }
    }
  ]
}
```

**Response (ineligible):**
```json
{
  "eligible": false,
  "eligibilityProgress": { "completed": 1, "required": 2 },
  "specials": []
}
```

---

#### `POST /api/community/bulk-specials/:specialId/commit`

**Authentication:** Required

**Purpose:** Commit to a bulk special with a specific quantity.

**Request Body:**
```json
{
  "quantity": 15
}
```

**Validation:**
- Customer must be eligible (completed participations >= threshold)
- Special must be active, not expired, not past cutoff
- Quantity must be >= min_per_customer and <= max_per_customer
- Quantity must be <= quantity_remaining
- Customer must not already be committed to this special

**Processing:**
1. Validate all constraints above
2. In a single transaction:
   a. Create `bulk_special_participants` row
   b. Decrement `bulk_specials.quantity_remaining` by the committed quantity
   c. Increment `bulk_specials.participant_count`
3. Check if participant_count now >= min_participants
   - If yes and special was not yet confirmed: update status to `confirmed`, notify all participants and vendor

**Response:**
```json
{
  "commitment": {
    "specialId": 7,
    "quantity": 15,
    "unitType": "lb",
    "subtotal": 7.50,
    "routeFee": 2.50,
    "total": 10.00,
    "deliveryDate": "2026-03-29"
  },
  "specialState": {
    "participantCount": 7,
    "minParticipants": 8,
    "confirmed": false,
    "quantityRemaining": 127
  }
}
```

**Error Responses:**
- `401` — Not authenticated
- `403` — Customer not eligible for bulk specials
- `400` — Quantity validation failed (below min, above max, exceeds remaining)
- `409` — Customer already committed to this special
- `404` — Special not found, expired, or past cutoff

---

#### `DELETE /api/community/bulk-specials/:specialId/commit`

**Authentication:** Required

**Purpose:** Cancel commitment to a bulk special (before cutoff only).

**Processing:**
1. Validate the special hasn't passed its cutoff
2. Find the customer's `bulk_special_participants` row
3. In a transaction: delete the row, increment quantity_remaining, decrement participant_count
4. If participant_count drops below min_participants and special was confirmed: revert to `forming` status, notify all remaining participants

**Response:**
```json
{
  "cancelled": true,
  "refundedQuantity": 15
}
```

---

### Database Tables

All tables below were introduced in C-02 v3.0. Reproduced here as the authoritative reference with any refinements.

```sql
-- ============================================================
-- ROUTE INTERESTS
-- Customer interest signals for unpartnered store routes
-- ============================================================
CREATE TABLE route_interests (
  id                          SERIAL PRIMARY KEY,
  customer_id                 INTEGER NOT NULL REFERENCES customers(id),
  store_id                    INTEGER NOT NULL REFERENCES stores(id),
  zone_id                     INTEGER NOT NULL REFERENCES zones(id),
  frequency                   VARCHAR(20) NOT NULL,  -- 'weekly','biweekly','monthly','flexible'
  preferred_day               VARCHAR(10),           -- 'thursday', etc. or NULL
  status                      VARCHAR(20) DEFAULT 'active',  -- 'active','dormant','cancelled'
  last_order_at               TIMESTAMPTZ,
  consecutive_participations  INTEGER DEFAULT 0,
  typical_order_size          VARCHAR(10),           -- 'small','medium','large' (computed)
  created_at                  TIMESTAMPTZ DEFAULT NOW(),
  updated_at                  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(customer_id, store_id)
);

CREATE INDEX idx_route_interests_store_zone ON route_interests(store_id, zone_id, status);
CREATE INDEX idx_route_interests_customer ON route_interests(customer_id);

-- ============================================================
-- WEEKLY BATCHES
-- One batch per store per zone per week
-- ============================================================
CREATE TABLE weekly_batches (
  id                  SERIAL PRIMARY KEY,
  store_id            INTEGER NOT NULL REFERENCES stores(id),
  zone_id             INTEGER NOT NULL REFERENCES zones(id),
  delivery_day        VARCHAR(10) NOT NULL,
  week_start          DATE NOT NULL,
  delivery_date       DATE NOT NULL,
  order_cutoff        TIMESTAMPTZ NOT NULL,
  status              VARCHAR(20) DEFAULT 'forming',
    -- forming, confirmed, shopping, delivering, complete, cancelled
  participant_count   INTEGER DEFAULT 0,
  min_participants    INTEGER NOT NULL,
  driver_id           INTEGER,  -- FK to drivers table (defined in D-01)
  previous_batch_id   INTEGER REFERENCES weekly_batches(id),
  created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_batches_store_week ON weekly_batches(store_id, zone_id, week_start);
CREATE INDEX idx_batches_status ON weekly_batches(status, delivery_date);

-- ============================================================
-- BATCH PARTICIPANTS
-- Customer commitment to a specific weekly batch
-- ============================================================
CREATE TABLE batch_participants (
  id              SERIAL PRIMARY KEY,
  batch_id        INTEGER NOT NULL REFERENCES weekly_batches(id),
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  status          VARCHAR(20) DEFAULT 'committed',
    -- committed, list_submitted, complete, skipped, cancelled
  shopping_fee    DECIMAL(6,2),
  rejoined_from   INTEGER REFERENCES batch_participants(id),
  committed_at    TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(batch_id, customer_id)
);

CREATE INDEX idx_participants_batch ON batch_participants(batch_id);
CREATE INDEX idx_participants_customer ON batch_participants(customer_id);

-- ============================================================
-- ROUTE HISTORY
-- Tracks route maturity over time (admin view)
-- ============================================================
CREATE TABLE route_history (
  id                    SERIAL PRIMARY KEY,
  store_id              INTEGER NOT NULL REFERENCES stores(id),
  zone_id               INTEGER NOT NULL REFERENCES zones(id),
  delivery_day          VARCHAR(10) NOT NULL,
  consecutive_weeks     INTEGER DEFAULT 0,
  avg_participants      DECIMAL(4,1),
  avg_revenue           DECIMAL(8,2),
  maturity_stage        VARCHAR(20) DEFAULT 'interest',
    -- interest, first_batch, recurring, dedicated_driver
  dedicated_driver_id   INTEGER,  -- FK to drivers (defined in D-01)
  last_batch_date       DATE,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(store_id, zone_id, delivery_day)
);

-- ============================================================
-- SHOPPING FEE CONFIG
-- Admin-configurable per unpartnered store
-- ============================================================
CREATE TABLE shopping_fee_config (
  id                SERIAL PRIMARY KEY,
  store_id          INTEGER NOT NULL REFERENCES stores(id),
  base_shopping_fee DECIMAL(6,2) NOT NULL,  -- solo fee (e.g., $15.00)
  min_shopping_fee  DECIMAL(6,2) NOT NULL,  -- floor when batched (e.g., $3.00)
  active            BOOLEAN DEFAULT true,
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(store_id)
);

-- ============================================================
-- ROUTE NOTE PRESETS
-- Admin-configurable preset feedback options
-- ============================================================
CREATE TABLE route_note_presets (
  id          SERIAL PRIMARY KEY,
  category    VARCHAR(30) NOT NULL,  -- 'timing','health','shopping'
  text        VARCHAR(300) NOT NULL,
  active      BOOLEAN DEFAULT true,
  sort_order  INTEGER DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- ROUTE FEEDBACK
-- Customer selections of preset feedback per route
-- ============================================================
CREATE TABLE route_feedback (
  id              SERIAL PRIMARY KEY,
  store_id        INTEGER NOT NULL REFERENCES stores(id),
  zone_id         INTEGER NOT NULL REFERENCES zones(id),
  delivery_day    VARCHAR(10) NOT NULL,
  preset_id       INTEGER NOT NULL REFERENCES route_note_presets(id),
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(store_id, zone_id, delivery_day, preset_id, customer_id)
);

CREATE INDEX idx_route_feedback_route ON route_feedback(store_id, zone_id, delivery_day);

-- ============================================================
-- BULK SPECIALS
-- One-time distribution routes from partnered vendors
-- ============================================================
CREATE TABLE bulk_specials (
  id                    SERIAL PRIMARY KEY,
  vendor_type           VARCHAR(20) NOT NULL,  -- 'store' or 'producer'
  store_id              INTEGER REFERENCES stores(id),
  producer_id           INTEGER REFERENCES producers(id),
  zone_id               INTEGER NOT NULL REFERENCES zones(id),
  product_name          VARCHAR(200) NOT NULL,
  product_description   VARCHAR(500),
  thumbnail_key         VARCHAR(255),
  price_per_unit        DECIMAL(8,2) NOT NULL,
  unit_type             VARCHAR(20) NOT NULL,  -- 'lb','each','dozen','bushel','pint','quart'
  min_per_customer      INTEGER NOT NULL DEFAULT 1,
  max_per_customer      INTEGER,
  quantity_total        INTEGER NOT NULL,
  quantity_remaining    INTEGER NOT NULL,
  min_participants      INTEGER NOT NULL,
  participant_count     INTEGER DEFAULT 0,
  route_fee             DECIMAL(6,2) NOT NULL,  -- delivery fee per participant
  extended_radius_miles DECIMAL(6,2),           -- NULL = standard zone radius
  delivery_day          VARCHAR(10) NOT NULL,
  delivery_date         DATE NOT NULL,
  cutoff_time           TIMESTAMPTZ NOT NULL,
  prep_time_days        INTEGER DEFAULT 1,
  vendor_note           VARCHAR(300),
  status                VARCHAR(20) DEFAULT 'open',
    -- open: accepting commitments
    -- confirmed: minimum met, vendor preparing
    -- distributing: driver is delivering
    -- complete: all delivered
    -- cancelled: minimum not met or vendor cancelled
  driver_id             INTEGER,  -- FK to drivers (defined in D-01)
  active                BOOLEAN DEFAULT true,
  created_at            TIMESTAMPTZ DEFAULT NOW()
);

-- Check constraint: exactly one vendor source
ALTER TABLE bulk_specials ADD CONSTRAINT bulk_specials_vendor_check
  CHECK (
    (store_id IS NOT NULL AND producer_id IS NULL) OR
    (store_id IS NULL AND producer_id IS NOT NULL)
  );

CREATE INDEX idx_bulk_specials_zone ON bulk_specials(zone_id, active, status);
CREATE INDEX idx_bulk_specials_cutoff ON bulk_specials(cutoff_time);
CREATE INDEX idx_bulk_specials_store ON bulk_specials(store_id) WHERE store_id IS NOT NULL;
CREATE INDEX idx_bulk_specials_producer ON bulk_specials(producer_id) WHERE producer_id IS NOT NULL;

-- ============================================================
-- BULK SPECIAL PARTICIPANTS
-- Customer commitments to bulk specials
-- ============================================================
CREATE TABLE bulk_special_participants (
  id              SERIAL PRIMARY KEY,
  special_id      INTEGER NOT NULL REFERENCES bulk_specials(id),
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  quantity        INTEGER NOT NULL,       -- quantity in the special's unit_type
  subtotal        DECIMAL(8,2) NOT NULL,  -- quantity × price_per_unit
  route_fee       DECIMAL(6,2) NOT NULL,  -- delivery fee for this participant
  total           DECIMAL(8,2) NOT NULL,  -- subtotal + route_fee
  status          VARCHAR(20) DEFAULT 'committed',
    -- committed, confirmed, delivered, cancelled
  committed_at    TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(special_id, customer_id)
);

CREATE INDEX idx_bulk_participants_special ON bulk_special_participants(special_id);
CREATE INDEX idx_bulk_participants_customer ON bulk_special_participants(customer_id);
```

### Shopping Fee Calculation

Used wherever the current batch fee needs to be displayed or charged:

```sql
-- Get the current fee for a batch
SELECT
  GREATEST(
    sfc.base_shopping_fee / GREATEST(wb.participant_count, 1),
    sfc.min_shopping_fee
  ) AS current_fee
FROM weekly_batches wb
JOIN shopping_fee_config sfc ON sfc.store_id = wb.store_id
WHERE wb.id = $1;
```

When a participant joins or leaves a batch, recalculate the fee for all participants:

```sql
UPDATE batch_participants bp
SET shopping_fee = GREATEST(
  sfc.base_shopping_fee / GREATEST(wb.participant_count, 1),
  sfc.min_shopping_fee
)
FROM weekly_batches wb
JOIN shopping_fee_config sfc ON sfc.store_id = wb.store_id
WHERE bp.batch_id = wb.id
AND wb.id = $1;
```

### Scheduled Job: Weekly Route Reformation

Runs Monday morning. Configurable time in `system_config`. Implemented as in-app cron (node-cron) inside the API container.

```
Step 1: Re-form routes that ran last week
─────────────────────────────────────────
For each route_history WHERE last_batch_date = last week:
  a. Create new weekly_batches row for this week
     - Link previous_batch_id to last week's batch
     - Set delivery_date based on delivery_day for this week
     - Set order_cutoff based on zone config
     - Set min_participants from system_config
  b. Find completed participants from last week's batch:
     SELECT customer_id, frequency
     FROM batch_participants bp
     JOIN route_interests ri ON ri.customer_id = bp.customer_id
       AND ri.store_id = (SELECT store_id FROM weekly_batches WHERE id = bp.batch_id)
     WHERE bp.batch_id = [last_week_batch_id]
     AND bp.status = 'complete'
     AND ri.status = 'active'
  c. For each participant, based on their frequency:
     - weekly: send re-formation notification
     - biweekly: send if last_order_at >= 2 weeks ago, softer msg otherwise
     - monthly: send if last_order_at >= 3 weeks ago, skip otherwise
     - flexible: send with casual framing

Step 2: Check building routes for threshold
───────────────────────────────────────────
For each store/zone/day with active interests but no batch this week:
  a. Calculate expected_weekly_participants using frequency weights
  b. If expected >= threshold AND no weekly_batch exists:
     - Create weekly_batches row
     - Create/update route_history (maturity = 'first_batch')
     - Notify all interested customers

Step 3: Handle dormancy
───────────────────────
For each route_interest WHERE status = 'active':
  If customer has not participated in a batch for this store
  in 3+ consecutive weeks where they were expected to:
    a. Mark status = 'dormant'
    b. Stop future notifications for this route

Step 4: Update route history
────────────────────────────
For each route_history:
  a. If last week's batch completed:
     - Increment consecutive_weeks
     - Update avg_participants (rolling average)
     - Update avg_revenue
     - Update maturity_stage based on thresholds
     - Update last_batch_date
  b. If last week's batch was cancelled or didn't exist:
     - Reset consecutive_weeks to 0
     - Update maturity_stage (may regress)
```

### Config Values (in system_config or dedicated tables)

| Config Key | Default | Description |
|---|---|---|
| `route_min_participants` | 5 | Minimum batch participants for a route to run |
| `route_dormancy_weeks` | 3 | Weeks of inactivity before marking interest dormant |
| `frequency_weight_weekly` | 1.0 | Expected participation multiplier for weekly frequency |
| `frequency_weight_biweekly` | 0.5 | Expected participation multiplier for biweekly |
| `frequency_weight_monthly` | 0.25 | Expected participation multiplier for monthly |
| `frequency_weight_flexible` | 0.3 | Expected participation multiplier for flexible |
| `route_recurring_threshold` | 3 | Consecutive weeks for a route to be considered recurring |
| `route_dedicated_driver_threshold` | 6 | Consecutive weeks to flag for dedicated driver hire |
| `route_dedicated_driver_avg_participants` | 8 | Avg participants needed for dedicated driver flag |
| `batch_reformation_day` | `monday` | Day of week the automated job runs |
| `batch_reformation_time` | `06:00` | Time (EST) the automated job runs |
| `bulk_special_eligibility_threshold` | 2 | Completed route participations required to access bulk specials |
| `bulk_special_default_route_fee` | 2.50 | Default delivery fee per participant for bulk specials |
| `bulk_special_max_radius_miles` | 75 | Maximum extended radius a vendor can set for a special |

All configurable through admin interface.

---

## ECS Fargate Notes

- The `GET /api/community` endpoint now runs 6-8 queries (added partnered store routes, eligibility check, and bulk specials). Still trivial at your data volume.
- The weekly reformation job is the most complex automated process. Run as node-cron inside the API container. Log each step to CloudWatch.
- Bulk special cutoff enforcement can run as a separate cron job (hourly check for specials past cutoff that need to be confirmed or cancelled). Lightweight — just a few queries.
- SMS notifications triggered by the reformation job and bulk special events should be queued and sent asynchronously. Use a simple in-process queue or batch the Twilio calls.
- When a participant joins or skips a batch, the fee recalculation runs for ALL participants in that batch. At 5-10 participants this is instant.
- Bulk special images stored in S3, served via CloudFront, same pattern as other images.

## Third-Party API Costs for This Feature

| Service | Operation | Cost | Monthly Estimate |
|---|---|---|---|
| Twilio SMS | Weekly re-formation notifications | $0.0079/msg | ~$5-10 |
| Twilio SMS | Threshold met / batch confirmed / cancelled | $0.0079/msg | ~$2-5 |
| Twilio SMS | Bulk special notifications (new, confirmed, cancelled) | $0.0079/msg | ~$2-5 |
| S3 + CloudFront | Bulk special product images | Negligible | < $0.01 |
| **Total** | | | **~$9-20/month** |
