# C-02: Home Screen & Navigation

> **Spec Version:** 3.0
> **Status:** Draft — Product catalog approach for partnered stores pending (see Open Questions).
> **Dependencies:** Google Maps API, PostgreSQL with PostGIS, C-01 (authenticated session), C-06 (Community & Route Building)
> **Relates to:** C-03 (Product Browsing), C-06 (Community & Route Building), C-08 (Rescue Bags), C-09 (Buying Club & Producers)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md.

---

## Overview

The home screen is the primary interface for authenticated customers. It is a full-bleed, map-centric dashboard with floating UI elements positioned strategically around the map. The map is always the dominant visual element — controls, panels, and content overlay it rather than pushing it into a column.

The home screen is NOT a shopping interface. Customers do not add items to their cart from this screen. It is a discovery and navigation tool that answers: "what's available, where is it, and how do I want to order?"

### Design Priorities
1. Map fills the viewport — all other UI elements float over or around it
2. Header is minimal — logo, cart, account. Nothing else.
3. Information appears only when relevant — no active order means no banner, no bags means no carousel
4. Toggles filter the map and surface contextual content without navigating away
5. Mobile-first — desktop enhances with a floating side panel, mobile uses a bottom sheet
6. Fast loading on slow connections — map tiles are the heaviest asset, everything else is lightweight HTML/CSS

### Store Tiers

The platform serves two fundamentally different types of stores. The home screen handles both but the customer experience differs based on the store's partnership tier:

**Tier 1 — Partnered local businesses:** Stores that have signed up for the platform and manage their own dashboard. They control their product catalog, pricing, and availability. The customer browses a real catalog and sees exact prices. Examples: local IGA (if partnered), Princeton Food Mart, cottage food producers.

**Tier 2 — Unpartnered corporate/big box stores:** Walmart, Hannaford, CVS, etc. No integration, no dashboard, no product catalog in the system. The customer builds a free-form shopping list and a driver personally shops for them. Delivery requires a viable batch route built through the Community tab, or the customer pays premium on-demand solo pricing.

The `stores` table includes a `partnership_tier` column (`'partnered'` or `'unpartnered'`) that controls which experience the customer gets when they interact with that store.

---

## Header

The header is persistent across ALL screens in the app, not just the home screen.

```
┌──────────────────────────────────────────────────────────┐
│  [Logo/App Name]                          🛒(3)    👤   │
└──────────────────────────────────────────────────────────┘
```

**Contents:**
- **Left:** App logo or name. Tapping it from any screen navigates to the home screen.
- **Right:** Cart icon and Account icon. Nothing else.
  - **Cart icon:** Shows a badge with the current item count (e.g., "3"). Badge is hidden when cart is empty. Tapping navigates to the Cart & Checkout flow (C-04).
  - **Account icon:** Tapping navigates to the Account page (C-11), which includes Orders, settings, profile, delivery preferences, and all account management.

**No other navigation in the header.** No Home link (logo serves this purpose), no Orders link (accessed via Account), no search bar, no hamburger menu.

**Styling:** The header should be visually lightweight — thin, not tall. It should not compete with the map for attention. Semi-transparent or subtle background is acceptable as long as the icons remain legible over map content that may scroll behind it on mobile.

---

## Active Order Banner

A conditional banner that appears directly below the header ONLY when the customer has one or more active orders (scheduled, in-progress, or out for delivery). When the customer has no active orders, this banner does not render — no empty state, no placeholder, no whitespace. The map extends right up to the header.

### Banner States

**Order scheduled (placed, awaiting delivery day):**
```
┌──────────────────────────────────────────────────────────┐
│  📦 Your Thursday order is scheduled · Delivering Mar 26  [→] │
└──────────────────────────────────────────────────────────┘
```

**Order in progress (driver is shopping):**
```
┌──────────────────────────────────────────────────────────┐
│  🛒 Your order is being prepared · Est. delivery 2-4pm   [→] │
└──────────────────────────────────────────────────────────┘
```

**Order out for delivery:**
```
┌──────────────────────────────────────────────────────────┐
│  🚗 Your order is on the way · Arriving soon              [→] │
└──────────────────────────────────────────────────────────┘
```

**Multiple active orders:**
```
┌──────────────────────────────────────────────────────────┐
│  📦 You have 2 active orders                              [→] │
└──────────────────────────────────────────────────────────┘
```

**Behavior:**
- Tapping the banner (or the [→] arrow) navigates to the order detail/tracking page (C-07). If multiple orders are active, it navigates to the orders list within the Account section.
- The banner is dismissible — the customer can swipe it away or tap an X to hide it for the current session. It reappears on the next app open if orders are still active.
- The banner should be styled as a subtle, colored strip — not an aggressive alert. It's informational, not urgent (except the "on the way" state, which can be slightly more prominent).
- On both mobile and desktop, the banner sits between the header and the map.

### Backend Logic for Banner

The `GET /api/home` endpoint returns active order data:
```json
{
  "activeOrders": [
    {
      "id": 142,
      "status": "scheduled",
      "deliveryDate": "2026-03-26",
      "deliveryWindow": "2-4pm",
      "itemCount": 8
    }
  ]
}
```

If `activeOrders` is an empty array, the frontend does not render the banner. The query:
```sql
SELECT id, status, delivery_date, delivery_window,
  (SELECT COUNT(*) FROM order_items WHERE order_id = orders.id) as item_count
FROM orders
WHERE customer_id = $1
AND status IN ('scheduled', 'confirmed', 'shopping', 'in_transit')
ORDER BY delivery_date ASC;
```

Ensure an index exists on `orders(customer_id, status)` for fast lookups.

---

## Map

### Layout

The map fills the entire viewport between the header (+ optional banner) at the top and the surprise bag strip (if visible) at the bottom. On desktop, the map is full width — the floating right panel overlays it. On mobile, the map is full screen behind the bottom sheet.

### Default State

- Centers on the customer's stored address coordinates
- Zoom level shows the customer's nearest 2-3 stores and nearby hubs — roughly the delivery zone extent
- Customer's location shown as a distinct "you are here" marker (visually unique — different shape, color, or subtle pulse animation)

### Pin Types

Three visually distinct pin types:

| Pin Type | Icon/Color | Represents |
|---|---|---|
| Store | Blue shopping bag icon | Partner grocery stores |
| Producer | Green leaf/wheat icon | Cottage food producers and farms |
| Hub | Gray/brown package icon (slightly smaller) | Community pickup hub locations |

### Map Controls

- **Zoom +/−:** Bottom-left corner of the map, above the surprise bag strip
- **Recenter (📍):** Near the zoom controls. Returns map to customer's location and default zoom.
- **Map/List toggle:** Floating pill in the top-left corner of the map: `[Map · List]`

### Pin Popup Cards

Tapping any pin opens a popup card anchored to that pin. Only one popup open at a time. Tapping a different pin replaces the previous. Tapping the map background closes any open popup.

On mobile, if a popup would be obscured by the bottom sheet, the map auto-pans.

#### Partnered Store Popup (Tier 1)

Partnered stores have a product catalog. The customer can browse and shop directly.

```
┌─────────────────────────────┐
│  🏪 Calais IGA              │
│  Calais · 42 mi             │
│  Groceries, local products  │
│  ✓ Partner store            │
│                             │
│  [View Products →]          │
└─────────────────────────────┘
```

"View Products →" navigates to the store's catalog page (C-03).

#### Unpartnered Store Popup (Tier 2) — Route-Dependent

Unpartnered stores (Walmart, Hannaford, etc.) show different content based on whether a viable batch route exists. The system checks the current week's batch state for this store + zone before rendering the card.

**State A — Active route exists this week:**

A batch route is forming or confirmed for this store. The customer can join and build a shopping list.

```
┌─────────────────────────────┐
│  🏪 Walmart Supercenter     │
│  Houlton · 35 mi            │
│  Groceries, household,      │
│  pharmacy                    │
│                             │
│  Thu route available         │
│  5 neighbors · $3.00 fee    │
│                             │
│  [Build Shopping List →]    │
└─────────────────────────────┘
```

"Build Shopping List →" navigates to the shopping list builder for this store (C-03), where the customer can build their list and commit to this week's batch.

**State B — Interest is building but no route yet:**

People have expressed interest through the Community tab but the threshold hasn't been met.

```
┌─────────────────────────────┐
│  🏪 Walmart Supercenter     │
│  Houlton · 35 mi            │
│  Groceries, household,      │
│  pharmacy                    │
│                             │
│  Route building: 3 of 5     │
│  Join to help start a route │
│                             │
│  [Join via Community →]     │
└─────────────────────────────┘
```

"Join via Community →" navigates to the Community tab with this store's route section focused. The customer cannot build a shopping list until the route threshold is met.

**State C — No interest, no route:**

Nobody in this zone has expressed interest in this store yet.

```
┌─────────────────────────────┐
│  🏪 Walmart Supercenter     │
│  Houlton · 35 mi            │
│  Groceries, household,      │
│  pharmacy                    │
│                             │
│  No route scheduled          │
│  Be the first to plan one   │
│                             │
│  [Express Interest →]       │
│  [Order Now — solo $15+]    │
└─────────────────────────────┘
```

"Express Interest →" navigates to the Community tab to register interest. "Order Now" is always available as the premium solo option (if `on_demand_enabled` is true), clearly showing the higher fee.

#### How Route State Is Determined

The `GET /api/home` endpoint checks batch state for each unpartnered store:

```sql
-- For each unpartnered store in the customer's zone:
-- 1. Check for an active weekly batch this week
SELECT wb.id, wb.status, wb.participant_count, wb.delivery_date,
  sc.base_shopping_fee / GREATEST(wb.participant_count, 1) as current_fee
FROM weekly_batches wb
JOIN shopping_fee_config sc ON sc.store_id = wb.store_id
WHERE wb.store_id = $1
AND wb.zone_id = $2
AND wb.week_start = date_trunc('week', CURRENT_DATE)
AND wb.status IN ('forming', 'confirmed');

-- 2. If no batch exists, count active route interests
SELECT COUNT(*) as interest_count
FROM route_interests
WHERE store_id = $1
AND zone_id = $2
AND status = 'active';
```

The result determines which popup state (A, B, or C) to render.

#### Producer Popup

```
┌─────────────────────────────┐
│  🌾 Martha's Pies           │
│  Danforth · 3 mi            │
│  Homemade pies and baked    │
│  goods                       │
│                             │
│  [View Products →]          │
└─────────────────────────────┘
```

"View Products →" navigates to the producer detail page (C-09). Producers are always Tier 1 (they manage their own listings).

#### Hub Popup

```
┌─────────────────────────────┐
│  📦 Topsfield General Store │
│  1.2 mi from you            │
│  Pickup hub                  │
│  No refrigeration            │
│  Hours: Mon-Sat 7am-6pm    │
└─────────────────────────────┘
```

Hub cards have NO action button — hubs are pickup locations only.

### Toggle Effects on Map Pins

When a toggle is active, relevant pins are full opacity and irrelevant pins are dimmed. Pins are NEVER hidden.

| Active Toggle | Full Opacity | Dimmed (40% opacity) |
|---|---|---|
| None (default) | All pins | None |
| Schedule & Save | Stores, Hubs | Producers |
| Order Now | All pins | None |
| Community | Producers, Hubs + anonymous neighbor indicators | Stores |

When "Community" is active, anonymized neighbor indicators appear on the map (fuzzy area circles, not exact location pins). These are specified in C-06.

---

## Floating Right Panel (Desktop Only)

A floating panel overlaying the right side of the map. The map continues behind it. The panel has a solid or semi-transparent background with sufficient opacity for readability.

**Dimensions:** ~320-380px wide, height determined by content (not full viewport). Positioned with margin from the top and right edges.

**Rounded corners, subtle shadow** to separate from the map.

### Panel Structure

```
┌──────────────────────┐
│                      │
│  Schedule & Save     │
│  ──────────────────  │
│  Order Now           │
│  ──────────────────  │
│  Community           │
│                      │
│  ════════════════    │
│                      │
│  (context content)   │
│                      │
└──────────────────────┘
```

### Toggle Buttons

Three buttons stacked vertically. One active at a time. Tapping an active toggle deactivates it.

**Visual states:**
- Inactive: outlined/ghost
- Active: filled/solid with distinct color
- Hover (desktop): slight background fill

### Context Content

**No toggle active:** Area below the divider is empty or collapsed. Panel shows only the three buttons — minimal footprint.

**"Schedule & Save" active:**
```
Next delivery: Thu Mar 26
Order by: Wed 8pm
⏰ 2d 4h remaining

Orders $75+: $3.99 delivery

[Browse Stores →]
```

- Delivery info from `zone_delivery_days`
- Countdown calculated client-side
- Fee estimate from `fee_tiers` config table
- "Browse Stores" opens list view filtered to stores/hubs

**"Order Now" active (on_demand_enabled = true):**
```
Delivery: next available run

Convenience fee: $8.99+

[Start an Order →]
```

**"Order Now" active (on_demand_enabled = false, launch state):**
```
On-demand delivery is
coming soon.

Schedule an order to
save now.

[Schedule & Save →]
```

**"Community" active:**
```
📅 This Week
──────────────────

Walmart · Houlton
Thu · Re-forming
5 of 7 back so far
✓ You're in
Fee: $3.00
[Edit my list →]

Hannaford · Lincoln
Wed · Re-forming
3 of 5 back so far
[I'm in →]

──────────────────

📋 Building Routes
──────────────────

CVS · Houlton
4 interested · Need 1 more
[Join →]

──────────────────

🌾 Local Producers
[Browse Producers →]
```

The Community toggle is a **route planning tool**, not a social feature. It serves three purposes:
1. **This Week** — Shows batch routes that ran previously and are re-forming for the current week. Customers rejoin with one tap ("I'm in"). Their previous shopping list is pre-loaded for editing. Shows how many people are back from last week and the current batch shopping fee.
2. **Building Routes** — Shows routes that don't have enough participants yet. Customers join to help reach the threshold. Once met, the route moves to "This Week" and all participants are notified.
3. **Local producer browsing** — "Browse Producers" opens list view filtered to producers.

**Weekly re-formation cycle:** Each Monday, the system creates new suggested batches for routes that ran the previous week. Last week's participants are notified and can rejoin with one tap. Shopping lists from the previous week are pre-staged for easy re-use. See C-06 for the complete weekly cycle, commitment model, and batch route lifecycle.

**Key principle:** Customers participate anonymously. No names, no addresses, no personal details are shared between neighbors. They see counts ("5 of 7 back") but never identities. The system handles clustering and fee calculation behind the scenes.

**Commitment model:** Tapping "I'm in" is a weekly commitment, not a recurring subscription. Each week requires an active opt-in. If a customer doesn't rejoin for 3 consecutive weeks, their interest is marked dormant and they stop receiving weekly notifications. They can re-express interest at any time. After the order cutoff passes, committed orders are locked in and will be shopped/charged. See C-06 for details.

---

## Bottom Sheet (Mobile Only)

Replaces the floating panel on mobile. Draggable, three snap positions.

### Collapsed (Peek) — Default State

```
┌─────────────────────────────┐
│                             │
│       FULL-SCREEN MAP       │
│                             │
├─────────────────────────────┤
│  ▬▬▬                        │
│ [Schedule & Save]           │
│ [Order Now]  [Community]    │
└─────────────────────────────┘
```

Shows drag handle and three toggle buttons only.

### Half-Expanded

```
├─────────────────────────────┤
│  ▬▬▬                        │
│ [Schedule & Save]           │
│ [Order Now]  [Community]    │
│                             │
│  (context content for       │
│   active toggle)            │
│                             │
│  ── Surprise Bags ────────  │
│  [🥖 IGA $5] [🥧 $4]  →   │
│                             │
│  [Map · List]               │
└─────────────────────────────┘
```

Toggle context content, surprise bag carousel, and Map/List toggle. Map visible and interactive in top half.

### Full-Expanded (List View)

```
├─────────────────────────────┤
│  [Map · List]               │
│                             │
│  ── Stores ───────────────  │
│  🏪 Walmart Supercenter     │
│     Houlton · 35 mi   [→]  │
│  ─────────────────────────  │
│  🏪 Hannaford               │
│     Lincoln · 48 mi   [→]  │
│  ─────────────────────────  │
│  🏪 Calais IGA              │
│     Calais · 42 mi   [→]  │
│                             │
│  ── Local Producers ──────  │
│  🌾 Martha's Pies           │
│     Danforth · 3 mi   [→]  │
│  ─────────────────────────  │
│  🍯 Wilson Honey Farm       │
│     Topsfield · 5 mi  [→]  │
│                             │
│  ── Pickup Hubs ──────────  │
│  📦 Topsfield General       │
│     1.2 mi from you        │
│  📦 Danforth Post Office    │
│     3.4 mi from you        │
└─────────────────────────────┘
```

Map hidden. Tapping "Map" collapses sheet back.

### Bottom Sheet Behavior
- Drag handle: visible centered gray bar
- Snaps to three positions on release
- Swipe up expands, swipe down collapses
- Map interactive in collapsed and half states
- Map hidden when sheet is full

---

## Surprise Bag Strip (Desktop) / Carousel (Mobile)

### Desktop — Full-Width Bottom Strip

Fixed to the bottom of the viewport below the map.

```
┌──────────────────────────────────────────────────────────────────┐
│  Surprise Bags   [🥖 IGA · $5] [🥧 Hannaford · $4] [🍞 $3] → │
└──────────────────────────────────────────────────────────────────┘
```

- "Surprise Bags" label on the left
- Horizontal scrollable row of bag cards
- Left/right arrows at edges for keyboard/mouse navigation
- Cards snap to position
- **If no bags are available, this strip does not render.** Map extends to viewport bottom.

### Mobile — Inside Bottom Sheet

The carousel lives inside the bottom sheet, visible in the half-expanded state. No fixed strip on mobile.

### Bag Card Format

```
┌────────────────────┐
│  [Thumbnail]       │
│                    │
│  Calais IGA        │
│  Surprise Bag      │
│  $5.00  ~~$15~~    │
│  3 left · Exp today│
└────────────────────┘
```

- Thumbnail: store/producer image from S3 via CloudFront
- Source name, "Surprise Bag" label
- Price with crossed-out estimated retail value
- Quantity remaining and expiry indicator

**Tapping a card** opens the Surprise Bag Detail View (C-08).

---

## List View

Accessible via the Map/List toggle.

### Desktop

List renders as a floating panel on the left side of the map (mirroring the right panel style), or replaces the map entirely. Choose based on Google Maps performance during implementation.

### Mobile

Bottom sheet expands to full, map hidden.

### Content

Filtered by active toggle:

| Active Toggle | Shows |
|---|---|
| None (default) | All stores, producers, hubs |
| Schedule & Save | Stores and hubs |
| Order Now | All stores, producers, hubs |
| Community | Producers only (hubs below) |

**Store row:**
```
🏪 Walmart Supercenter
   Houlton · 35 mi
   Groceries, household, pharmacy          [→]
```

**Producer row:**
```
🌾 Martha's Pies
   Danforth · 3 mi
   Homemade pies and baked goods           [→]
```

**Hub row:**
```
📦 Topsfield General Store
   1.2 mi from you
```

- Tapping store/producer [→] navigates to detail page (C-03 / C-09)
- Hub rows are informational only
- Sorted by distance (nearest first) within each section

---

## Full Desktop Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  [Logo]                                          🛒(3)     👤  │
├──────────────────────────────────────────────────────────────────┤
│  📦 Your Thursday order is scheduled · Mar 26             [→]   │ ← Only if active order
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                                                                  │
│  [Map · List]                          ┌──────────────────────┐  │
│                                        │ Schedule & Save      │  │
│                                        │ ──────────────────── │  │
│              FULL-WIDTH MAP            │ Order Now            │  │
│                                        │ ──────────────────── │  │
│   [📍 You are here]                   │ Community            │  │
│                                        │                      │  │
│   [🏪]  [🌾]   [📦]                   │ ════════════════     │  │
│                                        │                      │  │
│              [🏪]     [🍯]            │ (context content)    │  │
│                                        │                      │  │
│                                        └──────────────────────┘  │
│   [+ / -]  [📍]                                                  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│  Surprise Bags   [🥖 IGA · $5] [🥧 Hannaford · $4] [🍞 $3] → │ ← Only if bags exist
└──────────────────────────────────────────────────────────────────┘
```

## Full Mobile Layout

```
┌─────────────────────────────┐
│ [Logo]          🛒(3)  👤  │
├─────────────────────────────┤
│ 📦 Thu order · Mar 26  [→] │ ← Only if active order
├─────────────────────────────┤
│                             │
│       FULL-SCREEN MAP       │
│                             │
│ [Map · List]                │
│                             │
│  [📍] [🏪]  [🌾]  [📦]   │
│                             │
│       [+ / -]  [📍]       │
│                             │
├─────────────────────────────┤ ← Bottom sheet
│  ▬▬▬                        │
│ [Schedule & Save]           │
│ [Order Now]  [Community]    │
│                             │
│  (context + surprise bags   │
│   when half/full expanded)  │
└─────────────────────────────┘
```

---

## Returning User Experience

When an authenticated customer opens the app:

1. Header renders with logo, cart badge (if items in cart), and account icon
2. Active order banner renders if active orders exist — otherwise nothing
3. Map loads centered on their location with all zone pins
4. Floating panel (desktop) or bottom sheet (mobile) renders with no toggle active
5. Surprise bag strip (desktop) or carousel in sheet (mobile) renders if bags are available
6. No toggle active — panel shows only the three buttons, no context content

Clean, map-first, zero clutter.

---

## Driver Scaling Context

| Customer Mode | Driver Type | Availability |
|---|---|---|
| Schedule & Save (partnered stores) | Route drivers (dedicated, fixed schedule) | Scheduled delivery days per zone |
| Batch routes (unpartnered stores) | Route drivers initially, dedicated drivers as routes mature | Weekly, builds through Community tab |
| Order Now | Flex drivers (on-demand, flexible) | When drivers are active |
| Community (producer orders) | Either — rides on next applicable route | Based on delivery choice at checkout |

**Route maturity and hiring path:**

Batch routes for unpartnered stores progress through maturity stages tracked in the `route_history` table:

| Stage | Criteria | Driver |
|---|---|---|
| Interest | People expressing interest, threshold not met | No route runs |
| First batch | Threshold met, first week runs | Your partner handles alongside other routes |
| Recurring | 3+ consecutive weeks, 5+ regular participants | Your partner or an existing route driver |
| Dedicated driver | 6-8+ consecutive weeks, 8+ weekly participants | Hire a driver specifically for this route |

When a route reaches the "dedicated driver" stage, the route's weekly fee revenue reliably covers a driver's weekly cost. The admin dashboard tracks route maturity and flags when routes are ready for a dedicated hire. This is the path from "one person doing everything" to "stable employment for drivers running reliable weekly routes to Walmart and Hannaford."

At launch: partner is the only driver, running scheduled routes. "Order Now" shows "coming soon" via `on_demand_enabled` feature flag = `false`. Batch routes build organically through the Community tab. Flag flipped when flex drivers are hired for on-demand.

---

## Backend Specification

### API Endpoints

#### `GET /api/home`

**Authentication:** Required (session cookie)

**Purpose:** Return all data needed to render the home screen in a single request.

**Processing:**
1. Read customer profile (id, first_name, lat, lng, zone_id)
2. Query active orders (status IN scheduled, confirmed, shopping, in_transit)
3. Query stores in zone with distance and partnership_tier
4. For each unpartnered store: query weekly batch state (active batch, interest count, or none)
5. Query active producers in zone with distance
6. Query active hubs in zone with distance
7. Query available surprise bags (active, not expired, quantity_remaining > 0)
8. Query next delivery window from `zone_delivery_days`
9. Query this week's batch routes the customer is participating in (for Community toggle "This Week" section)
10. Query building routes (interest exists but threshold not met) for the customer's zone
11. Read `on_demand_enabled` feature flag
12. Read `fee_tiers` for zone (for Schedule & Save fee estimate)
13. Read cart item count

**Response:**
```json
{
  "customer": {
    "firstName": "Martha",
    "lat": 45.6912,
    "lng": -67.8834
  },
  "zone": {
    "id": 1,
    "name": "Houlton North Corridor"
  },
  "activeOrders": [
    {
      "id": 142,
      "status": "scheduled",
      "deliveryDate": "2026-03-26",
      "deliveryWindow": "2-4pm",
      "itemCount": 8
    }
  ],
  "nextDelivery": {
    "dayOfWeek": "thursday",
    "date": "2026-03-26",
    "cutoffTime": "2026-03-25T20:00:00-04:00"
  },
  "feeEstimate": {
    "thresholdAmount": 75.00,
    "feeAtThreshold": 3.99,
    "label": "Orders $75+: $3.99 delivery"
  },
  "stores": [
    {
      "id": 1,
      "name": "Walmart Supercenter",
      "town": "Houlton",
      "description": "Groceries, household, pharmacy",
      "lat": 46.1281,
      "lng": -67.8401,
      "distanceMiles": 35.2,
      "partnershipTier": "unpartnered",
      "batchState": {
        "status": "active",
        "batchId": 42,
        "deliveryDay": "thursday",
        "deliveryDate": "2026-03-26",
        "participantCount": 5,
        "previousWeekCount": 7,
        "currentShoppingFee": 3.00,
        "customerIsParticipant": true,
        "customerHasList": true
      }
    },
    {
      "id": 2,
      "name": "Hannaford",
      "town": "Lincoln",
      "description": "Groceries, deli, bakery",
      "lat": 45.3622,
      "lng": -68.5061,
      "distanceMiles": 48.1,
      "partnershipTier": "unpartnered",
      "batchState": {
        "status": "building",
        "interestCount": 3,
        "threshold": 5,
        "customerHasInterest": false
      }
    },
    {
      "id": 3,
      "name": "Calais IGA",
      "town": "Calais",
      "description": "Groceries, local products",
      "lat": 45.1834,
      "lng": -67.2765,
      "distanceMiles": 42.0,
      "partnershipTier": "partnered",
      "batchState": null
    }
  ],
  "producers": [
    {
      "id": 1,
      "name": "Martha's Pies",
      "town": "Danforth",
      "description": "Homemade pies and baked goods",
      "lat": 45.6558,
      "lng": -67.8706,
      "distanceMiles": 3.4
    }
  ],
  "hubs": [
    {
      "id": 1,
      "name": "Topsfield General Store",
      "address": "123 Main St, Topsfield",
      "lat": 45.4132,
      "lng": -67.7234,
      "distanceMiles": 1.2,
      "hasRefrigeration": false,
      "hours": "Mon-Sat 7am-6pm"
    }
  ],
  "communityRoutes": {
    "thisWeek": [
      {
        "batchId": 42,
        "storeName": "Walmart Supercenter",
        "storeId": 1,
        "deliveryDay": "thursday",
        "deliveryDate": "2026-03-26",
        "participantCount": 5,
        "previousWeekCount": 7,
        "currentShoppingFee": 3.00,
        "customerStatus": "committed",
        "customerHasList": true
      }
    ],
    "building": [
      {
        "storeId": 2,
        "storeName": "Hannaford",
        "interestCount": 3,
        "threshold": 5,
        "customerHasInterest": false
      }
    ]
  },
  "surpriseBags": [
    {
      "id": 1,
      "sourceName": "Calais IGA",
      "sourceType": "store",
      "sourceId": 3,
      "thumbnailKey": "bags/calais-iga-default.jpg",
      "price": 5.00,
      "estimatedRetailValue": 15.00,
      "contentsHint": "Mix of bakery items and deli",
      "quantityRemaining": 3,
      "expiresAt": "2026-03-20T18:00:00-04:00"
    }
  ],
  "features": {
    "onDemandEnabled": false
  },
  "cartItemCount": 3
}
```

**Key response details:**
- `stores[].partnershipTier` determines which popup card the frontend renders (catalog browse vs. route-dependent states)
- `stores[].batchState` is only present for unpartnered stores. `null` for partnered stores.
- `stores[].batchState.status` is one of: `"active"` (batch route exists this week), `"building"` (interest gathering, threshold not met), `"none"` (no interest)
- `communityRoutes` provides the data for the Community toggle panel, separated into "This Week" (active batches) and "Building" (gathering interest)
- `customerIsParticipant` / `customerHasInterest` flags tell the frontend whether to show "I'm in" vs. "Edit my list" vs. "Join"

**Error Responses:**
- `401` — Not authenticated
- `500` — Internal error (global handler)

---

#### `GET /api/surprise-bags`

**Authentication:** Required

**Purpose:** Refresh surprise bag data independently. Poll every 5 minutes or on screen return.

**Query Parameters:** `zone_id` (required)

**Response:** Same `surpriseBags` array as home endpoint.

---

#### `GET /api/surprise-bags/:id`

**Authentication:** Required

**Purpose:** Full detail for a specific bag (tapped from carousel).

**Response:**
```json
{
  "id": 1,
  "sourceName": "Calais IGA",
  "sourceType": "store",
  "sourceId": 3,
  "sourceDescription": "Local grocery store in Calais",
  "sourceAddress": "361 South St, Calais",
  "thumbnailKey": "bags/calais-iga-default.jpg",
  "price": 5.00,
  "estimatedRetailValue": 15.00,
  "contentsHint": "Mix of bakery items and deli",
  "dietaryNotes": "May contain gluten, dairy, and nuts",
  "quantityRemaining": 3,
  "expiresAt": "2026-03-20T18:00:00-04:00",
  "pickupInstructions": "Available at the deli counter after 3pm"
}
```

**Error Responses:**
- `401` — Not authenticated
- `404` — Not found or expired
- `500` — Internal error

---

### Database Tables (New for This Flow)

```sql
-- ============================================================
-- SURPRISE BAGS
-- ============================================================
CREATE TABLE surprise_bags (
  id                      SERIAL PRIMARY KEY,
  source_type             VARCHAR(20) NOT NULL,  -- 'store' or 'producer'
  source_id               INTEGER NOT NULL,       -- references stores.id or producers.id
  thumbnail_key           VARCHAR(255),
  price                   DECIMAL(6,2) NOT NULL,
  estimated_retail_value  DECIMAL(6,2),
  contents_hint           VARCHAR(300),
  dietary_notes           VARCHAR(300),
  quantity_total          INTEGER NOT NULL,
  quantity_remaining      INTEGER NOT NULL,
  pickup_instructions     VARCHAR(300),
  expires_at              TIMESTAMPTZ NOT NULL,
  zone_id                 INTEGER NOT NULL REFERENCES zones(id),
  active                  BOOLEAN NOT NULL DEFAULT true,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_surprise_bags_zone ON surprise_bags(zone_id);
CREATE INDEX idx_surprise_bags_expires ON surprise_bags(expires_at);
CREATE INDEX idx_surprise_bags_active ON surprise_bags(active, zone_id, expires_at);

-- ============================================================
-- INDEX ON ORDERS FOR ACTIVE ORDER BANNER QUERY
-- ============================================================
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- ============================================================
-- STORE TIER COLUMN (added to existing stores table from C-01)
-- ============================================================
ALTER TABLE stores ADD COLUMN partnership_tier VARCHAR(20) NOT NULL DEFAULT 'unpartnered';
  -- 'partnered': full integration, product catalog, business dashboard
  -- 'unpartnered': no integration, shopping list model, batch routes

-- ============================================================
-- ROUTE INTERESTS (community-driven route building)
-- ============================================================
CREATE TABLE route_interests (
  id              SERIAL PRIMARY KEY,
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  store_id        INTEGER NOT NULL REFERENCES stores(id),
  zone_id         INTEGER NOT NULL REFERENCES zones(id),
  preferred_day   VARCHAR(10),          -- 'monday', 'thursday', etc.
  status          VARCHAR(20) DEFAULT 'active', -- 'active', 'dormant', 'cancelled'
  last_order_at   TIMESTAMPTZ,          -- tracks recency for dormancy detection
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(customer_id, store_id)
);

CREATE INDEX idx_route_interests_store_zone ON route_interests(store_id, zone_id, status);
CREATE INDEX idx_route_interests_customer ON route_interests(customer_id);

-- ============================================================
-- WEEKLY BATCHES (one per store per zone per week)
-- ============================================================
CREATE TABLE weekly_batches (
  id                SERIAL PRIMARY KEY,
  store_id          INTEGER NOT NULL REFERENCES stores(id),
  zone_id           INTEGER NOT NULL REFERENCES zones(id),
  delivery_day      VARCHAR(10) NOT NULL,     -- 'thursday'
  week_start        DATE NOT NULL,            -- Monday of the batch week
  delivery_date     DATE NOT NULL,
  order_cutoff      TIMESTAMPTZ NOT NULL,
  status            VARCHAR(20) DEFAULT 'forming',
    -- forming: open for commitments
    -- confirmed: cutoff passed, enough participants
    -- shopping: driver is at the store
    -- delivering: driver is on the route
    -- complete: all delivered
    -- cancelled: didn't meet minimum threshold
  participant_count INTEGER DEFAULT 0,
  min_participants  INTEGER NOT NULL,         -- from config
  driver_id         INTEGER REFERENCES drivers(id),
  previous_batch_id INTEGER REFERENCES weekly_batches(id), -- link to last week
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_batches_store_week ON weekly_batches(store_id, zone_id, week_start);
CREATE INDEX idx_batches_status ON weekly_batches(status, delivery_date);

-- ============================================================
-- BATCH PARTICIPANTS (customer commitment per weekly batch)
-- ============================================================
CREATE TABLE batch_participants (
  id              SERIAL PRIMARY KEY,
  batch_id        INTEGER NOT NULL REFERENCES weekly_batches(id),
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  status          VARCHAR(20) DEFAULT 'committed',
    -- committed: tapped "I'm in"
    -- list_submitted: shopping list is ready
    -- complete: order delivered
    -- skipped: opted out this week
    -- cancelled: cancelled after committing (before cutoff)
  shopping_fee    DECIMAL(6,2),             -- their share of batch fee
  rejoined_from   INTEGER REFERENCES batch_participants(id), -- link to last week
  committed_at    TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(batch_id, customer_id)
);

CREATE INDEX idx_participants_batch ON batch_participants(batch_id);
CREATE INDEX idx_participants_customer ON batch_participants(customer_id);

-- ============================================================
-- ROUTE HISTORY (admin view — tracks route maturity over time)
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
    -- interest: gathering interest, no batch has run
    -- first_batch: first week ran
    -- recurring: 3+ consecutive weeks
    -- dedicated_driver: 6-8+ weeks, dedicated hire justified
  dedicated_driver_id   INTEGER REFERENCES drivers(id),
  last_batch_date       DATE,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(store_id, zone_id, delivery_day)
);

-- ============================================================
-- SHOPPING FEE CONFIG (admin-configurable per store)
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
-- ITEM ALIASES (fuzzy matching dictionary, grows over time)
-- ============================================================
CREATE TABLE item_aliases (
  id              SERIAL PRIMARY KEY,
  canonical_name  VARCHAR(200) NOT NULL,  -- "2% Milk, 1 Gallon"
  alias           VARCHAR(200) NOT NULL,  -- "2 percent milk, gallon"
  store_id        INTEGER REFERENCES stores(id),
  confirmed_count INTEGER DEFAULT 1,      -- incremented each time driver confirms match
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_item_aliases_store ON item_aliases(store_id);
CREATE INDEX idx_item_aliases_canonical ON item_aliases(canonical_name);
```

### Tables Referenced (Defined Elsewhere)
- `customers`, `zones`, `stores`, `producers`, `hubs` — C-01
- `zone_delivery_days` — A-02
- `orders`, `order_items` — C-04
- `feature_flags` — A-01
- `fee_tiers` — A-01
- `convenience_fee_config` — A-01 (future, for on-demand pricing)
- `drivers` — D-01 (referenced by weekly_batches and route_history)

### Scheduled Job: Weekly Route Reformation

Runs every Monday morning (configurable) as an ECS scheduled task or in-app cron. This is the heartbeat of the batch system.

```
1. For each route_history with consecutive_weeks > 0:
   a. Create a new weekly_batches row for this week
   b. Link to last week's batch via previous_batch_id
   c. Find all completed participants from last week's batch
   d. Send each a "route is re-forming" notification (SMS via Twilio)
   e. Pre-stage their shopping list from last week (available for 
      one-tap re-use when they tap "I'm in")

2. For each route_interest grouping where threshold is met
   but no batch has ever run:
   a. Create the first weekly_batches row
   b. Notify all interested customers: "Route is starting!"
   c. Update route_history to maturity_stage = 'first_batch'

3. For route_interests where customer hasn't participated in 
   3+ consecutive weeks:
   a. Mark status = 'dormant'
   b. Stop sending weekly notifications
   c. Decrement active interest counts

4. Update route_history: increment consecutive_weeks for routes 
   that ran last week, reset to 0 for routes that didn't,
   recalculate avg_participants and avg_revenue
```

---

## Responsive Breakpoints

| Breakpoint | Map | Right Panel | Surprise Bags | Bottom Sheet |
|---|---|---|---|---|
| < 768px | Full viewport behind sheet | None | Inside bottom sheet | Yes |
| 768px - 1024px | Full viewport | Floating, ~300px | Full-width bottom strip | No |
| > 1024px | Full viewport | Floating, ~360px | Full-width bottom strip | No |

---

## Open Questions

1. **Partnered store catalog approach:** How customers browse Tier 1 (partnered) store products. Pending C-03 decision — real product catalog with photos and prices.
2. **Shopping list builder for unpartnered stores:** The detailed UX for building a freeform shopping list for Walmart/Hannaford (quick-add buttons, saved lists, item templates, substitution preferences). To be detailed in C-03 as the second path alongside catalog browsing.
3. **Payment flow for unpartnered store orders:** How the actual grocery cost is charged to the customer when the driver shops on their behalf. Pre-authorization via Stripe (hold estimated amount, capture actual after shopping) is the likely approach. To be detailed in C-04 (Cart & Checkout).
4. **On-demand availability display:** When on-demand is enabled, how to display estimated delivery time. Deferred to D-01 and on-demand feature build.
5. **Desktop list view placement:** Float as left panel or replace map? Choose during implementation.
6. **Consolidated shopping list generation for drivers:** How the system merges multiple customer shopping lists into one optimized driver list grouped by store department. To be detailed in D-02 (Pickup Checklist).
7. **Item alias matching:** The fuzzy matching system for deduplicating freeform shopping list items across customers. Simple string similarity + driver confirmation at MVP, growing dictionary over time. To be detailed in D-02.

---

## ECS Fargate Notes

- `GET /api/home` is now the heaviest endpoint — 12-15 queries including batch state checks for each unpartnered store. At your current data volume (2-3 unpartnered stores, handful of batches) this is still under 100ms total. If it becomes slow as store/route count grows, batch state for unpartnered stores can be pre-computed and cached with a 5-minute TTL.
- The weekly route reformation job runs once per week. It should execute within seconds at your scale. Run it as an in-app cron job (node-cron), not a separate ECS scheduled task — simpler and avoids spinning up another Fargate task.
- Surprise bag images and store thumbnails in S3 served via CloudFront.
- Google Maps JS API loaded on frontend only. API key restricted to CloudFront domain.
- Map tile loading is the heaviest network operation. Floating panel, surprise bag strip, and the order banner should all render independently of map readiness.

## Third-Party API Costs

| Service | Cost | Monthly Estimate |
|---|---|---|
| Google Maps JS API | $7 / 1,000 loads | ~$1-3 |
| S3 + CloudFront (bag images) | Negligible | < $0.01 |
| **Total** | | **~$1-3/month** |
