# C-03: Product Browsing & Shopping Lists

> **Spec Version:** 1.0
> **Status:** Complete — Ready for Development
> **Dependencies:** C-01 (authenticated session), C-02 (store pin navigation), C-06 (batch route context for unpartnered stores), PostgreSQL
> **Relates to:** C-02 (store popup cards link here), C-04 (Cart & Checkout — items flow from here into cart), C-06 (batch participation unlocks shopping list builder), C-09 (Producer profiles — shared catalog components), S-01 (Store Dashboard — product management), D-02 (Driver Pickup — reads submitted shopping lists)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md.

---

## Overview

C-03 is the product browsing and list-building experience. It covers the screen a customer lands on after tapping a store or producer from the home screen map. The experience is fundamentally different depending on the store's partnership tier:

**Path 1 — Partnered store/producer catalog:** The customer browses a real product catalog managed by the business. Photos, descriptions, exact prices, allergen info. Standard e-commerce product browsing. The customer adds items to their cart directly.

**Path 2 — Unpartnered store shopping list builder:** No catalog exists. The customer builds a freeform text-based shopping list. A driver will personally shop it at the store. Prices are estimates until the driver completes the shopping trip.

Both paths add items to the same cart (C-04), but items are typed differently: `catalog` items have a linked product record with known pricing, while `shopping_list` items are freeform text with estimated pricing.

### Design Principles

1. **One store per order.** The cart can only contain items from a single store at a time. Multi-store orders are a future feature.
2. **Partnered stores show real products.** Every listed product has a photo, name, price, and description. No empty or placeholder listings.
3. **Unpartnered stores acknowledge the personal shopping model.** The customer understands a human is shopping for them. Substitution preferences and driver notes are first-class fields.
4. **Fast reordering.** Previous orders and saved lists load with minimal taps. The customer always lands on an editable list before submitting.
5. **Price transparency.** Partnered store prices are exact. Unpartnered store prices are clearly labeled as estimates. All fees are visible before committing.

---

## Shared Store Header

Both paths share the same header component at the top of the page. The header provides store context and delivery information.

```
┌─────────────────────────────────────────┐
│  ← Back                                │
│                                         │
│  [Store photo]                          │
│  🏪 Calais IGA                          │
│  Calais · 42 mi                         │
│  Groceries, local products              │
│                                         │
│  📅 Wed delivery · 4 neighbors ordering │
│  🚚 Delivery fee: $4.99                │
└─────────────────────────────────────────┘
```

**Fields:**
- Back navigation (returns to home screen map)
- Store/producer profile photo (from S3 via CloudFront)
- Store name
- Town and distance from customer
- Store description
- Delivery context line (varies by tier — see below)
- Fee context line

### Delivery Context by Store Tier

**Partnered store:**
```
📅 Wed delivery · 4 neighbors ordering
🚚 Delivery fee: $4.99
```
Shows the next scheduled delivery day from `zone_delivery_days`, how many other customers in the zone are ordering from this store this week, and the standard delivery fee.

**Unpartnered store — active batch route:**
```
📅 Thu batch route · 5 participants
🛒 Shopping fee: $3.00 · Delivery: $3.99
```
Shows the batch delivery day, participant count, and both the batch shopping fee and delivery fee.

**Unpartnered store — no active route:**
```
⚡ On-demand · Solo shopping trip
🛒 Shopping fee: $15.00 · Delivery: $5.99
```
Shows the solo pricing. Only visible when `on_demand_enabled` feature flag is true.

**Producer (always partnered):**
```
📅 Rides on next route to your area
🌾 Buying Club · Direct from producer
```
Producer orders are combined with the next applicable delivery route. No separate delivery fee — it's bundled into the route. The "Buying Club" label maintains food sovereignty compliance.

---

## Path 1: Partnered Store & Producer Catalog

### Page Layout — Mobile

```
┌─────────────────────────────────────────┐
│  [Store Header — as above]              │
│                                         │
│  🔍 Search products...                  │
│                                         │
│  [All] [Produce] [Dairy] [Bakery] →     │
│                                         │
│  ┌───────────┐  ┌───────────┐           │
│  │  [Photo]  │  │  [Photo]  │           │
│  │  Sourdough│  │  Apple    │           │
│  │  Bread    │  │  Pie      │           │
│  │  $4.50    │  │  $12.00   │           │
│  │  [Add]    │  │  [Add]    │           │
│  └───────────┘  └───────────┘           │
│  ┌───────────┐  ┌───────────┐           │
│  │  [Photo]  │  │  [Photo]  │           │
│  │  Honey    │  │  Blueberry│           │
│  │  16oz     │  │  Jam 8oz  │           │
│  │  $8.00    │  │  $6.00    │           │
│  │  [Add]    │  │  [Add]    │           │
│  └───────────┘  └───────────┘           │
│                                         │
│  (scroll for more)                      │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  🛒 Cart: 3 items · $20.50      │   │
│  │  [View Cart →]                   │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Page Layout — Desktop

```
┌──────────────────────────────────────────────────────────────────┐
│  [Store Header — wider, photo and text side by side]             │
│                                                                  │
│  🔍 Search products...                                           │
│                                                                  │
│  [All] [Produce] [Dairy] [Bakery] [Meat] [Pantry] [Household]  │
│                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │ [Photo] │  │ [Photo] │  │ [Photo] │  │ [Photo] │           │
│  │ Sourdough│ │ Apple   │  │ Honey   │  │ Jam     │           │
│  │ Bread   │  │ Pie     │  │ 16oz    │  │ 8oz     │           │
│  │ $4.50   │  │ $12.00  │  │ $8.00   │  │ $6.00   │           │
│  │ [Add]   │  │ [Add]   │  │ [Add]   │  │ [Add]   │           │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Search

A text input at the top of the product area. Filters the product grid as the customer types (client-side filtering for stores with < 200 products, API search for larger catalogs). Debounced at 300ms. Clears with an X button.

Search matches against product name, description, and category name. Results replace the grid in real time — no separate search results page.

If the search returns zero results: "No products match '[query]'. Try a different search or browse categories."

### Category Filters

Horizontal scrollable row of pill buttons. "All" is always first and selected by default. Other categories come from the store's product catalog — only categories that contain at least one active product are shown.

Tapping a category filters the grid to products in that category. Tapping "All" resets. Only one category active at a time. The active pill is visually highlighted.

Categories are defined per-store by the business through their dashboard. Common examples: Produce, Dairy, Bakery, Meat, Pantry, Beverages, Household, Personal Care. Producers may have simpler categories: Baked Goods, Preserves, Honey, Seasonal.

### Product Cards

Each product card in the grid contains:

| Element | Details |
|---|---|
| Photo | Product image from S3 via CloudFront. Aspect ratio 1:1 (square crop). Required — every listed product must have a photo. |
| Name | Product name, max 2 lines with ellipsis truncation |
| Size/weight (optional) | "16oz", "1 lb", "dozen" — only shown if the product has a `unit_label` |
| Price | Formatted as $X.XX. For products sold by weight, shown as "$X.XX/lb" |
| Add button | Tapping adds 1 unit to cart. Button transforms to quantity stepper after first add. |

**Add button behavior:**

1. Initial state: `[Add]` button
2. Customer taps Add → item added to cart with quantity 1, brief checkmark animation, button transforms to `[ - 1 + ]` quantity stepper
3. Customer can adjust quantity with stepper. Tapping `−` at quantity 1 removes the item from cart, button reverts to `[Add]`
4. The floating cart bar at the bottom updates item count and total in real time

**Card tap behavior:**

Tapping the card itself (not the Add button) opens the product detail view. On mobile this is a full-screen page. On desktop this is a modal overlay.

### Product Detail View

```
┌─────────────────────────────────────────┐
│  ← Back to [Store Name]                │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │                                 │   │
│  │      [Large product photo]      │   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Sourdough Bread                        │
│  $4.50                                  │
│                                         │
│  Fresh baked sourdough, made weekly     │
│  from local flour. Crusty exterior,     │
│  soft tangy interior. Best enjoyed      │
│  within 3 days.                         │
│                                         │
│  Ingredients: flour, water, salt,       │
│  sourdough starter                      │
│                                         │
│  ⚠️ Contains: Wheat, Gluten            │
│                                         │
│  Quantity:  [ - ]  1  [ + ]             │
│                                         │
│  [Add to Cart — $4.50]                 │
│                                         │
└─────────────────────────────────────────┘
```

**Fields:**
- Back navigation with store name
- Large product photo (full width on mobile, constrained on desktop)
- Product name
- Price (updates with quantity: "Add to Cart — $9.00" at quantity 2)
- Description (full text, no truncation)
- Ingredients (optional — populated by business in their dashboard)
- Allergen warnings (optional — displayed with ⚠️ icon, from product `allergens` field)
- Quantity selector
- "Add to Cart" button with running total

If the product is already in the cart, the detail view opens with the current cart quantity pre-filled and the button reads "Update Cart — $X.XX".

### Food Sovereignty Producer Products

For products from food sovereignty producers (Track 2 in Maine's food sovereignty framework), the product detail view includes an additional notice:

```
│  🌾 Buying Club Product                │
│  Made by [Producer Name] in             │
│  [Town], Maine                          │
│  Sold directly to you through the       │
│  Danforth Delivers Buying Club          │
│                                         │
│  This product is sold under Maine's     │
│  Food Sovereignty Act. By purchasing,   │
│  you acknowledge a direct transaction   │
│  with the producer.                     │
```

This maintains the legal framework: the producer is the seller, the customer is buying directly, and the platform facilitates but does not sell. The exact language is stored in `system_config` so it can be updated by legal review without a code change.

### Floating Cart Bar

A persistent bar at the bottom of the screen (above the mobile nav bar if applicable) that shows the current cart state:

```
┌─────────────────────────────────────────┐
│  🛒 Cart: 3 items · $20.50  [View →]   │
└─────────────────────────────────────────┘
```

- Only visible when the cart has items
- Shows item count and running total (product prices only, no fees — fees shown at checkout)
- "View Cart" navigates to the cart page (C-04)
- On desktop: positioned at the bottom-right corner as a floating element
- On mobile: fixed to the bottom of the viewport

### Empty State

If a partnered store has zero active products (possible for a new store that hasn't finished setting up their catalog):

```
[Store Name] is setting up their
product catalog.

Check back soon, or browse other
stores in your area.

[← Back to Map]
```

### Out of Stock / Unavailable

Products can be marked as unavailable by the business without being deleted. Unavailable products show in the grid but are visually dimmed with an "Unavailable" label. The Add button is disabled. Tapping the card shows the detail view with a note: "This product is currently unavailable. Check back later."

---

## Path 2: Unpartnered Store Shopping List Builder

### Access Control

The shopping list builder is only accessible when the customer has an active relationship with a batch route for this store. Specifically, one of:

1. They are a `batch_participants` row with status `committed` or `list_submitted` for an active `weekly_batches` row for this store
2. `on_demand_enabled` is true (they can order solo at premium pricing)

If neither condition is met, tapping the store shows the appropriate Community tab redirect (States B or C from C-02).

### Page Layout — Mobile

```
┌─────────────────────────────────────────┐
│  [Store Header — with batch context]    │
│                                         │
│  Your driver will personally shop       │
│  for you at this store.                 │
│                                         │
│  ── Your Shopping List ───────────────  │
│                                         │
│  (empty state or item list)             │
│                                         │
│  ── Quick Add ────────────────────────  │
│  [Milk] [Bread] [Eggs] [Butter] →      │
│                                         │
│  ── Saved Lists ──────────────────────  │
│  📋 My weekly basics (8 items)  [Load] │
│  📋 Last order (12 items)       [Load] │
│                                         │
│  ── Fee Summary ──────────────────────  │
│  (fee breakdown)                        │
│                                         │
│  [Submit List →]                        │
│                                         │
└─────────────────────────────────────────┘
```

### Page Layout — Desktop

```
┌──────────────────────────────────────────────────────────────────┐
│  [Store Header — wider]                                          │
│                                                                  │
│  ┌──────────────────────────────┐  ┌──────────────────────────┐ │
│  │  Your Shopping List           │  │  Quick Add               │ │
│  │                               │  │  [Milk] [Bread] [Eggs]  │ │
│  │  (item list)                  │  │  [Butter] [Chicken]     │ │
│  │                               │  │  [Ground Beef] [Cereal] │ │
│  │  [+ Add another item]        │  │  [Rice] [Pasta]         │ │
│  │                               │  │  [Canned Soup]          │ │
│  │                               │  │  [Toilet Paper]         │ │
│  │                               │  │  [Paper Towels]         │ │
│  │                               │  │                          │ │
│  │                               │  │  ── Saved Lists ──────  │ │
│  │                               │  │  📋 Weekly basics [Load]│ │
│  │                               │  │  📋 Last order   [Load]│ │
│  │                               │  │                          │ │
│  │                               │  │  ── Fee Summary ──────  │ │
│  │                               │  │  Est. total: ~$77.34    │ │
│  │                               │  │  [Submit List →]        │ │
│  └──────────────────────────────┘  └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

On desktop, the shopping list and the sidebar (quick-add, saved lists, fees, submit) sit side by side.

### Empty State

When the customer first lands on the shopping list builder with no items:

```
┌─────────────────────────────────────────┐
│  Your shopping list is empty.           │
│                                         │
│  Add items below, or load a             │
│  saved list to get started.             │
│                                         │
│  [+ Add your first item]               │
│                                         │
│  ── Quick Add ────────────────────────  │
│  [Milk] [Bread] [Eggs] [Butter] →      │
│                                         │
│  ── Saved Lists ──────────────────────  │
│  📋 Last order (12 items)       [Load] │
└─────────────────────────────────────────┘
```

### Shopping List Items

Each item in the list displays as a row:

```
┌─────────────────────────────────────────┐
│  2% Milk, 1 gallon                 x1   │
│  Sub: Store brand okay                  │
│  ~$3.50                                 │
│                          [edit] [🗑️]   │
└─────────────────────────────────────────┘
```

**Row contents:**
- Item description (freeform text the customer entered)
- Quantity
- Substitution preference (abbreviated: "Store brand okay", "Closest match", "Skip if out")
- Estimated price (if the customer entered one, prefixed with ~ to indicate it's an estimate)
- Edit and delete buttons

Items are ordered by the sequence they were added. No automatic sorting or grouping — the customer controls the order.

### Adding an Item

Tapping "+ Add another item" or "+ Add your first item" opens the item entry form.

**Mobile:** Full-screen modal sliding up from the bottom.
**Desktop:** Inline expansion below the last item, or a modal.

```
┌─────────────────────────────────────────┐
│  Add an item                     [✕]   │
│                                         │
│  What do you need?                      │
│  ┌─────────────────────────────────┐   │
│  │ e.g., "2% milk, 1 gallon"      │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Quantity:  [ - ]  1  [ + ]             │
│                                         │
│  If it's not available:                 │
│  ● Get a similar item (store brand ok)  │
│  ○ Get the closest match                │
│  ○ Skip it — don't substitute           │
│                                         │
│  Notes for driver (optional):           │
│  ┌─────────────────────────────────┐   │
│  │ e.g., "Prefer Hannaford brand"  │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Est. price (optional): $               │
│  ┌─────────────────────────────────┐   │
│  │ e.g., 3.50                      │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [Add to List]                          │
│                                         │
└─────────────────────────────────────────┘
```

**Fields:**

| Field | Required | Details |
|---|---|---|
| Item description | Yes | Freeform text, max 200 chars. The customer describes what they want in plain language. |
| Quantity | Yes | Integer, default 1, min 1, max 99 |
| Substitution preference | Yes | Radio buttons, default "Get a similar item (store brand ok)". Three options. |
| Notes for driver | No | Freeform text, max 300 chars. Brand preferences, size specifics, location in store, etc. |
| Estimated price | No | Decimal, the customer's best guess at the per-unit price. Used for fee estimate display only. |

**Substitution preference options:**
1. **Get a similar item (store brand ok)** — Default. Driver uses judgment to find a comparable substitute. Most flexible.
2. **Get the closest match** — Driver tries to find the most similar product (same brand different size, same flavor different brand, etc.) but doesn't make a big leap.
3. **Skip it — don't substitute** — If the exact item isn't available, remove it from the order. No substitution.

### Editing an Item

Tapping "edit" on an item row opens the same form pre-filled with the item's current values. The button reads "Save Changes" instead of "Add to List." The customer can modify any field. Available at any point before the batch cutoff.

### Deleting an Item

Tapping the delete icon (🗑️) shows a brief confirmation: "Remove [item name]?" with [Remove] and [Cancel] buttons. On confirm, the item is removed from the list and the fee summary updates.

### Quick Add

A section of tappable pill buttons below the shopping list. Each pill represents a common grocery item.

**Behavior:** Tapping a quick-add pill opens the item entry form pre-filled with a sensible default:

| Pill Label | Pre-filled Description | Default Qty | Default Sub |
|---|---|---|---|
| Milk | Milk, 1 gallon | 1 | Similar item ok |
| Bread | Bread, sliced, 1 loaf | 1 | Similar item ok |
| Eggs | Eggs, 1 dozen | 1 | Similar item ok |
| Butter | Butter, 1 lb | 1 | Similar item ok |
| Chicken | Chicken breast, 1 lb | 1 | Closest match |
| Ground Beef | Ground beef, 1 lb | 1 | Closest match |
| Cereal | Cereal, family size | 1 | Similar item ok |
| Rice | Rice, white, 5 lb bag | 1 | Similar item ok |
| Pasta | Pasta, 1 lb box | 1 | Similar item ok |
| Canned Soup | Canned soup | 1 | Similar item ok |
| Toilet Paper | Toilet paper, 12 roll | 1 | Similar item ok |
| Paper Towels | Paper towels, 6 roll | 1 | Similar item ok |
| Bananas | Bananas, 1 bunch | 1 | Skip if out |
| Apples | Apples, 3 lb bag | 1 | Closest match |
| Potatoes | Potatoes, 5 lb bag | 1 | Similar item ok |
| Onions | Onions, 3 lb bag | 1 | Similar item ok |
| Frozen Pizza | Frozen pizza | 1 | Closest match |
| Peanut Butter | Peanut butter, jar | 1 | Similar item ok |
| Dish Soap | Dish soap, liquid | 1 | Similar item ok |
| Laundry Detergent | Laundry detergent | 1 | Similar item ok |

The customer can edit any pre-filled value before tapping "Add to List", or accept the defaults and add immediately.

**Quick-add data source:**

At launch, quick-add pills are pre-seeded from the `item_aliases` table with `source = 'preseed'`. As real orders accumulate, the system tracks which items are most frequently ordered at each store. The quick-add list is regenerated periodically (weekly cron job) to surface the most popular items for each store, replacing low-usage pre-seeds.

The quick-add list is **per-store** — popular items at Walmart may differ from Hannaford. Pre-seeds are shared across all stores initially.

Scroll horizontally on mobile. Wrap into multiple rows on desktop.

### Saved Lists

Below the quick-add section. Shows saved list templates and the customer's most recent submitted list.

**"Last order"** — Automatically saved. Shows the item count from the customer's most recent completed shopping list for this store. Tapping "Load" replaces the current list with last order's items.

**Custom saved lists** — The customer can save their current list as a named template. Maximum 5 saved lists per store per customer.

```
── Saved Lists ──────────────────────────
📋 My weekly basics (8 items)     [Load]
📋 Holiday shopping (22 items)    [Load]
📋 Last order (12 items)          [Load]

[Save current list as template →]
```

**"Save current list as template"** prompts for a name:
```
Save as template
Name: [My weekly basics___________]
[Save]  [Cancel]
```

**Loading a saved list:** If the current list has items, show a confirmation: "Replace your current list with '[Template Name]' (X items)? Your current items will be removed. [Replace] [Cancel]"

If the current list is empty, load immediately without confirmation.

Loading a saved list or previous order copies the items into the current list. The customer must review and tap "Submit List" to finalize. No automatic submission.

### Fee Summary

Always visible at the bottom of the shopping list builder (mobile) or in the sidebar (desktop). Updates in real time as items are added, edited, or removed.

```
── Fee Summary ──────────────────────────
Est. items total:    ~$67.00
Shopping fee:         $3.00  (batch of 5)
Delivery fee:         $3.99
Service fee (5%):     $3.35
                     ────────
Est. total:          ~$77.34

Items charged at actual store price.
Your card is charged after the driver
shops and sends you the receipt.

[Submit List →]
```

**Fee breakdown fields:**

| Fee | Source | Notes |
|---|---|---|
| Est. items total | Sum of customer's estimated prices | Prefixed with ~ to indicate estimate. Shows "—" if no estimates entered. |
| Shopping fee | `shopping_fee_config.base_shopping_fee / participant_count`, floored at `min_shopping_fee` | Shows "(batch of X)" to indicate the fee is shared. For solo on-demand: shows full fee, no batch label. |
| Delivery fee | From `fee_tiers` based on order value estimate | Standard tier-based delivery fee. |
| Service fee | 5% of estimated items total, capped at $5.00 | From `service_fee_config` |
| Est. total | Sum of all above | Prefixed with ~ |

**Important note displayed:** "Items charged at actual store price. Your card is charged after the driver shops and sends you the receipt." This sets expectations that the final total may differ from the estimate.

### Submit List

The "Submit List" button finalizes the shopping list.

**Validation before submit:**
- List must have at least 1 item
- Each item must have a description (non-empty)
- Each item must have a quantity >= 1
- Each item must have a substitution preference selected

**On submit:**
1. Items are saved to `shopping_list_items` table linked to the customer's `batch_participants` record
2. `batch_participants.status` updated from `committed` to `list_submitted`
3. Fee estimates saved on the participation record
4. Customer sees confirmation:

```
List submitted! ✓

Your driver will shop these 12 items
at Walmart on Thursday.

You can edit your list until the
cutoff: Wed 8pm (1d 6h from now)

[Edit List]  [Back to Home]
```

After submission, the customer can return to this page at any time before the batch cutoff to edit their list. Changes are saved immediately — no need to re-submit. After the cutoff, the list is locked and the page shows a read-only view with a message: "Your list is locked. Your driver will shop it on [delivery day]."

---

## Cart Integration

Both paths add items to the cart, but items are structured differently in the `cart_items` table.

### Partnered Store Items (Catalog)

When a customer taps "Add" on a partnered store product:

```sql
INSERT INTO cart_items (
  customer_id, store_id, item_type, product_id,
  product_name, unit_price, quantity
) VALUES (
  $1, $2, 'catalog', $3,
  'Sourdough Bread', 4.50, 1
);
```

- `item_type = 'catalog'`
- `product_id` references the `products` table
- `unit_price` is exact (from the product record)
- No substitution preference or driver notes (product is known)

### Unpartnered Store Items (Shopping List)

When a customer submits their shopping list:

```sql
INSERT INTO cart_items (
  customer_id, store_id, item_type, batch_participant_id,
  product_name, estimated_price, quantity,
  substitution_preference, driver_notes
) VALUES (
  $1, $2, 'shopping_list', $3,
  '2% Milk, 1 gallon', 3.50, 1,
  'similar_ok', 'Prefer store brand'
);
```

- `item_type = 'shopping_list'`
- `product_id` is NULL (no linked product)
- `batch_participant_id` links to the batch participation record
- `estimated_price` instead of `unit_price` (may be NULL if customer didn't estimate)
- `substitution_preference` field: `'similar_ok'`, `'closest_match'`, or `'skip'`
- `driver_notes` field for per-item notes

### Single-Store Enforcement

When adding an item to the cart, the system checks if the cart already contains items from a different store:

```sql
SELECT DISTINCT store_id FROM cart_items
WHERE customer_id = $1 AND store_id != $2;
```

If a different store is found, the customer sees:

```
You have items from [Current Store]
in your cart.

Start a new order for [New Store]?
Your [Current Store] items will be
saved for later.

[Switch to New Store]  [Stay]
```

"Switch to New Store" moves current cart items to a `saved_cart` state (not deleted, just parked) and starts a fresh cart for the new store. "Stay" returns to the current store's page.

---

## Backend Specification

### API Endpoints

---

#### `GET /api/stores/:storeId/products`

**Authentication:** Required

**Purpose:** Return the product catalog for a partnered store. Used for Path 1 browsing.

**Query Parameters:**
- `category` (optional) — Filter by category slug
- `search` (optional) — Text search across product name and description
- `page` (optional, default 1) — Pagination for large catalogs
- `limit` (optional, default 50) — Products per page

**Processing:**
1. Validate store exists and is active
2. If `category` provided: filter by `products.category_id` matching the slug
3. If `search` provided: filter by `product_name ILIKE '%search%' OR product_description ILIKE '%search%'`
4. Return paginated results sorted by category sort_order, then product sort_order

**Response:**
```json
{
  "store": {
    "id": 3,
    "name": "Calais IGA",
    "partnershipTier": "partnered",
    "town": "Calais",
    "description": "Groceries, local products",
    "profilePhotoKey": "stores/calais-iga.jpg",
    "distanceMiles": 42.0
  },
  "deliveryContext": {
    "deliveryDay": "wednesday",
    "deliveryDate": "2026-03-25",
    "neighborCount": 4,
    "deliveryFee": 4.99
  },
  "categories": [
    { "id": 1, "name": "Produce", "slug": "produce", "productCount": 12 },
    { "id": 2, "name": "Dairy", "slug": "dairy", "productCount": 8 },
    { "id": 3, "name": "Bakery", "slug": "bakery", "productCount": 5 }
  ],
  "products": [
    {
      "id": 101,
      "name": "Sourdough Bread",
      "description": "Fresh baked sourdough, made weekly from local flour.",
      "ingredients": "flour, water, salt, sourdough starter",
      "allergens": ["wheat", "gluten"],
      "photoKey": "products/calais-iga/sourdough.jpg",
      "price": 4.50,
      "unitLabel": null,
      "categoryId": 3,
      "categoryName": "Bakery",
      "available": true,
      "cartQuantity": 0
    },
    {
      "id": 102,
      "name": "Apple Pie",
      "description": "Whole apple pie, 9 inch. Made fresh.",
      "ingredients": "apples, flour, butter, sugar, cinnamon",
      "allergens": ["wheat", "dairy"],
      "photoKey": "products/calais-iga/apple-pie.jpg",
      "price": 12.00,
      "unitLabel": null,
      "categoryId": 3,
      "categoryName": "Bakery",
      "available": true,
      "cartQuantity": 2
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 45,
    "totalPages": 1
  }
}
```

**Notes:**
- `cartQuantity` tells the frontend whether to show the Add button or the quantity stepper for each product. The backend checks the customer's active cart.
- `photoKey` is transformed to full CloudFront URL on the frontend: `https://cdn.yourapp.com/${photoKey}`
- `available: false` products are included but the frontend dims them and disables the Add button.

---

#### `GET /api/stores/:storeId/products/:productId`

**Authentication:** Required

**Purpose:** Return full detail for a single product. Used for the product detail view.

**Response:**
```json
{
  "id": 101,
  "name": "Sourdough Bread",
  "description": "Fresh baked sourdough, made weekly from local flour. Crusty exterior, soft tangy interior. Best enjoyed within 3 days.",
  "ingredients": "flour, water, salt, sourdough starter",
  "allergens": ["wheat", "gluten"],
  "photoKey": "products/calais-iga/sourdough.jpg",
  "price": 4.50,
  "unitLabel": null,
  "categoryId": 3,
  "categoryName": "Bakery",
  "available": true,
  "cartQuantity": 0,
  "store": {
    "id": 3,
    "name": "Calais IGA",
    "town": "Calais"
  },
  "producer": null,
  "foodSovereignty": false
}
```

For producer products with food sovereignty status:
```json
{
  "producer": {
    "id": 5,
    "name": "Sue's Sourdough",
    "town": "Weston",
    "sovereigntyTrack": "track_2"
  },
  "foodSovereignty": true,
  "foodSovereigntyNotice": "This product is sold under Maine's Food Sovereignty Act..."
}
```

---

#### `GET /api/stores/:storeId/shopping-list`

**Authentication:** Required

**Purpose:** Return the customer's current shopping list for an unpartnered store. Used for Path 2.

**Processing:**
1. Validate store is unpartnered
2. Find the customer's active `batch_participants` record for this store's current week batch (or on-demand order)
3. Return the shopping list items, quick-add suggestions, saved lists, and fee breakdown

**Response:**
```json
{
  "store": {
    "id": 1,
    "name": "Walmart Supercenter",
    "partnershipTier": "unpartnered",
    "town": "Houlton",
    "description": "Groceries, household, pharmacy",
    "distanceMiles": 35.2
  },
  "batchContext": {
    "batchId": 42,
    "deliveryDay": "thursday",
    "deliveryDate": "2026-03-26",
    "cutoffTime": "2026-03-25T20:00:00-04:00",
    "participantCount": 5,
    "shoppingFee": 3.00,
    "deliveryFee": 3.99,
    "listLocked": false,
    "participantStatus": "committed"
  },
  "items": [
    {
      "id": 1,
      "description": "2% Milk, 1 gallon",
      "quantity": 1,
      "substitutionPreference": "similar_ok",
      "driverNotes": "Prefer store brand",
      "estimatedPrice": 3.50,
      "sortOrder": 1
    },
    {
      "id": 2,
      "description": "Cheerios, family size",
      "quantity": 1,
      "substitutionPreference": "similar_ok",
      "driverNotes": null,
      "estimatedPrice": 5.99,
      "sortOrder": 2
    }
  ],
  "quickAdd": [
    { "label": "Milk", "defaultDescription": "Milk, 1 gallon", "defaultQuantity": 1, "defaultSubstitution": "similar_ok" },
    { "label": "Bread", "defaultDescription": "Bread, sliced, 1 loaf", "defaultQuantity": 1, "defaultSubstitution": "similar_ok" },
    { "label": "Eggs", "defaultDescription": "Eggs, 1 dozen", "defaultQuantity": 1, "defaultSubstitution": "similar_ok" }
  ],
  "savedLists": [
    { "id": 1, "name": "My weekly basics", "itemCount": 8 },
    { "id": 2, "name": "Last order", "itemCount": 12, "isLastOrder": true }
  ],
  "feeEstimate": {
    "estimatedItemsTotal": 67.00,
    "shoppingFee": 3.00,
    "deliveryFee": 3.99,
    "serviceFee": 3.35,
    "estimatedTotal": 77.34
  }
}
```

---

#### `POST /api/stores/:storeId/shopping-list/items`

**Authentication:** Required

**Purpose:** Add an item to the shopping list.

**Request Body:**
```json
{
  "description": "2% Milk, 1 gallon",
  "quantity": 1,
  "substitutionPreference": "similar_ok",
  "driverNotes": "Prefer store brand",
  "estimatedPrice": 3.50
}
```

**Validation:**
- `description` required, max 200 chars, non-empty after trim
- `quantity` required, integer, 1-99
- `substitutionPreference` required, one of: `similar_ok`, `closest_match`, `skip`
- `driverNotes` optional, max 300 chars
- `estimatedPrice` optional, decimal >= 0
- Batch must not be past cutoff
- Customer must have an active batch participation

**Response:** Returns the created item with its `id` and updated fee estimate.

---

#### `PATCH /api/stores/:storeId/shopping-list/items/:itemId`

**Authentication:** Required

**Purpose:** Update a shopping list item.

**Request Body:** Same fields as POST, all optional (only provided fields are updated).

**Validation:** Same constraints. Batch must not be past cutoff.

---

#### `DELETE /api/stores/:storeId/shopping-list/items/:itemId`

**Authentication:** Required

**Purpose:** Remove an item from the shopping list.

**Validation:** Batch must not be past cutoff. Item must belong to this customer's list.

---

#### `POST /api/stores/:storeId/shopping-list/submit`

**Authentication:** Required

**Purpose:** Submit the shopping list (mark it ready for the driver).

**Validation:**
- List must have at least 1 item
- All items must pass validation
- Batch must not be past cutoff
- Customer must be in `committed` status

**Processing:**
1. Validate all items
2. Update `batch_participants.status` to `list_submitted`
3. Save fee estimates on the participation record
4. Return confirmation with cutoff time

---

#### `POST /api/stores/:storeId/shopping-list/save-template`

**Authentication:** Required

**Purpose:** Save the current shopping list as a named template.

**Request Body:**
```json
{
  "name": "My weekly basics"
}
```

**Validation:**
- `name` required, max 50 chars
- Customer can have max 5 saved templates per store
- Name must be unique among this customer's templates for this store

---

#### `POST /api/stores/:storeId/shopping-list/load-template/:templateId`

**Authentication:** Required

**Purpose:** Load a saved template or previous order, replacing the current list.

**Processing:**
1. Delete all current shopping list items for this batch participation
2. Copy items from the template to the current list
3. Return the loaded items

---

#### `POST /api/cart/items`

**Authentication:** Required

**Purpose:** Add a partnered store product to the cart.

**Request Body:**
```json
{
  "storeId": 3,
  "productId": 101,
  "quantity": 1
}
```

**Validation:**
- Product must exist, be active, and be available
- Product must belong to the specified store
- Store must be partnered
- Single-store enforcement: if cart has items from a different store, return `409` with the conflict details

**Processing:**
1. Check for existing cart item with same product: if exists, increment quantity
2. If new: insert `cart_items` row with `item_type = 'catalog'`
3. Return updated cart state

**Response:**
```json
{
  "cartItem": {
    "id": 15,
    "productId": 101,
    "productName": "Sourdough Bread",
    "unitPrice": 4.50,
    "quantity": 1,
    "itemType": "catalog"
  },
  "cart": {
    "storeId": 3,
    "storeName": "Calais IGA",
    "itemCount": 3,
    "subtotal": 20.50
  }
}
```

**Conflict Response (409):**
```json
{
  "error": "cart_store_conflict",
  "currentStore": {
    "id": 1,
    "name": "Walmart Supercenter"
  },
  "requestedStore": {
    "id": 3,
    "name": "Calais IGA"
  },
  "message": "Cart contains items from Walmart Supercenter. Switch stores or stay?"
}
```

---

#### `PATCH /api/cart/items/:itemId`

**Authentication:** Required

**Purpose:** Update quantity of a cart item.

**Request Body:**
```json
{
  "quantity": 3
}
```

**Validation:** Quantity must be 1-99. Setting to 0 deletes the item.

---

#### `DELETE /api/cart/items/:itemId`

**Authentication:** Required

**Purpose:** Remove an item from the cart.

---

### Database Tables

```sql
-- ============================================================
-- PRODUCT CATEGORIES (per-store, managed by business)
-- ============================================================
CREATE TABLE product_categories (
  id          SERIAL PRIMARY KEY,
  store_id    INTEGER REFERENCES stores(id),
  producer_id INTEGER REFERENCES producers(id),
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(100) NOT NULL,
  sort_order  INTEGER DEFAULT 0,
  active      BOOLEAN DEFAULT true,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  -- exactly one parent
  CONSTRAINT categories_parent_check CHECK (
    (store_id IS NOT NULL AND producer_id IS NULL) OR
    (store_id IS NULL AND producer_id IS NOT NULL)
  )
);

CREATE INDEX idx_categories_store ON product_categories(store_id) WHERE store_id IS NOT NULL;
CREATE INDEX idx_categories_producer ON product_categories(producer_id) WHERE producer_id IS NOT NULL;

-- ============================================================
-- PRODUCTS (partnered store and producer catalog)
-- ============================================================
CREATE TABLE products (
  id                  SERIAL PRIMARY KEY,
  store_id            INTEGER REFERENCES stores(id),
  producer_id         INTEGER REFERENCES producers(id),
  category_id         INTEGER REFERENCES product_categories(id),
  name                VARCHAR(200) NOT NULL,
  description         VARCHAR(1000),
  ingredients         VARCHAR(500),
  allergens           TEXT[],                -- array: ['wheat','dairy','nuts']
  photo_key           VARCHAR(255) NOT NULL, -- S3 key, required
  price               DECIMAL(8,2) NOT NULL,
  unit_label          VARCHAR(50),           -- 'lb','oz','dozen', NULL if sold per unit
  available           BOOLEAN DEFAULT true,
  sort_order          INTEGER DEFAULT 0,
  food_sovereignty    BOOLEAN DEFAULT false,  -- true for Track 2 products
  active              BOOLEAN DEFAULT true,
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW(),
  -- exactly one parent
  CONSTRAINT products_parent_check CHECK (
    (store_id IS NOT NULL AND producer_id IS NULL) OR
    (store_id IS NULL AND producer_id IS NOT NULL)
  )
);

CREATE INDEX idx_products_store ON products(store_id, active, available) WHERE store_id IS NOT NULL;
CREATE INDEX idx_products_producer ON products(producer_id, active, available) WHERE producer_id IS NOT NULL;
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- ============================================================
-- CART ITEMS (unified cart for both catalog and shopping list items)
-- ============================================================
CREATE TABLE cart_items (
  id                        SERIAL PRIMARY KEY,
  customer_id               INTEGER NOT NULL REFERENCES customers(id),
  store_id                  INTEGER NOT NULL REFERENCES stores(id),
  item_type                 VARCHAR(20) NOT NULL,  -- 'catalog' or 'shopping_list'

  -- Catalog items (partnered stores)
  product_id                INTEGER REFERENCES products(id),
  unit_price                DECIMAL(8,2),

  -- Shopping list items (unpartnered stores)
  batch_participant_id      INTEGER REFERENCES batch_participants(id),
  estimated_price           DECIMAL(8,2),
  substitution_preference   VARCHAR(20),  -- 'similar_ok','closest_match','skip'
  driver_notes              VARCHAR(300),

  -- Shared fields
  product_name              VARCHAR(200) NOT NULL,  -- denormalized for display
  quantity                  INTEGER NOT NULL DEFAULT 1,
  sort_order                INTEGER DEFAULT 0,

  created_at                TIMESTAMPTZ DEFAULT NOW(),
  updated_at                TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cart_items_customer ON cart_items(customer_id);
CREATE INDEX idx_cart_items_store ON cart_items(customer_id, store_id);

-- ============================================================
-- SHOPPING LIST ITEMS (working list before submission, linked to batch)
-- ============================================================
CREATE TABLE shopping_list_items (
  id                        SERIAL PRIMARY KEY,
  batch_participant_id      INTEGER NOT NULL REFERENCES batch_participants(id),
  customer_id               INTEGER NOT NULL REFERENCES customers(id),
  store_id                  INTEGER NOT NULL REFERENCES stores(id),
  description               VARCHAR(200) NOT NULL,
  quantity                  INTEGER NOT NULL DEFAULT 1,
  substitution_preference   VARCHAR(20) NOT NULL DEFAULT 'similar_ok',
  driver_notes              VARCHAR(300),
  estimated_price           DECIMAL(8,2),
  sort_order                INTEGER DEFAULT 0,
  created_at                TIMESTAMPTZ DEFAULT NOW(),
  updated_at                TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_shopping_list_batch ON shopping_list_items(batch_participant_id);
CREATE INDEX idx_shopping_list_customer ON shopping_list_items(customer_id, store_id);

-- ============================================================
-- SAVED LIST TEMPLATES
-- ============================================================
CREATE TABLE saved_list_templates (
  id              SERIAL PRIMARY KEY,
  customer_id     INTEGER NOT NULL REFERENCES customers(id),
  store_id        INTEGER NOT NULL REFERENCES stores(id),
  name            VARCHAR(50) NOT NULL,
  is_last_order   BOOLEAN DEFAULT false,  -- auto-generated from last completed order
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_saved_templates_customer ON saved_list_templates(customer_id, store_id);

-- ============================================================
-- SAVED LIST TEMPLATE ITEMS
-- ============================================================
CREATE TABLE saved_list_template_items (
  id                        SERIAL PRIMARY KEY,
  template_id               INTEGER NOT NULL REFERENCES saved_list_templates(id) ON DELETE CASCADE,
  description               VARCHAR(200) NOT NULL,
  quantity                  INTEGER NOT NULL DEFAULT 1,
  substitution_preference   VARCHAR(20) NOT NULL DEFAULT 'similar_ok',
  driver_notes              VARCHAR(300),
  estimated_price           DECIMAL(8,2),
  sort_order                INTEGER DEFAULT 0
);

CREATE INDEX idx_template_items ON saved_list_template_items(template_id);

-- ============================================================
-- ITEM ALIASES (quick-add learning + fuzzy matching)
-- ============================================================
CREATE TABLE item_aliases (
  id              SERIAL PRIMARY KEY,
  canonical_name  VARCHAR(200) NOT NULL,
  alias           VARCHAR(200) NOT NULL,
  store_id        INTEGER REFERENCES stores(id),  -- NULL = applies to all stores
  source          VARCHAR(20) DEFAULT 'organic',   -- 'preseed' or 'organic'
  usage_count     INTEGER DEFAULT 1,
  confirmed_count INTEGER DEFAULT 0,               -- driver confirmations
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_item_aliases_store ON item_aliases(store_id);
CREATE INDEX idx_item_aliases_canonical ON item_aliases(canonical_name);
CREATE INDEX idx_item_aliases_usage ON item_aliases(store_id, usage_count DESC);

-- ============================================================
-- QUICK ADD SUGGESTIONS (materialized per-store, refreshed weekly)
-- ============================================================
CREATE TABLE quick_add_suggestions (
  id                      SERIAL PRIMARY KEY,
  store_id                INTEGER REFERENCES stores(id),  -- NULL = default set
  label                   VARCHAR(50) NOT NULL,
  default_description     VARCHAR(200) NOT NULL,
  default_quantity        INTEGER DEFAULT 1,
  default_substitution    VARCHAR(20) DEFAULT 'similar_ok',
  source                  VARCHAR(20) DEFAULT 'preseed',  -- 'preseed' or 'learned'
  sort_order              INTEGER DEFAULT 0,
  active                  BOOLEAN DEFAULT true,
  created_at              TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_quick_add_store ON quick_add_suggestions(store_id, active);
```

### Tables Referenced (Defined Elsewhere)
- `customers`, `zones`, `stores`, `producers` — C-01
- `batch_participants`, `weekly_batches` — C-06
- `fee_tiers`, `service_fee_config`, `shopping_fee_config` — A-01 / C-06
- `feature_flags` (`on_demand_enabled`) — A-01
- `zone_delivery_days` — A-02

---

## Responsive Breakpoints

| Breakpoint | Product Grid Columns | Layout | Product Detail |
|---|---|---|---|
| < 768px | 2 | Full-screen, stacked | Full-screen page |
| 768px - 1024px | 3 | Full-screen, stacked | Modal overlay |
| > 1024px | 4 | Full-screen, stacked | Modal overlay |

| Breakpoint | Shopping List Layout | Quick Add | Item Entry Form |
|---|---|---|---|
| < 768px | Full-screen, single column | Horizontal scroll | Full-screen modal |
| 768px - 1024px | Two-column (list + sidebar) | Wrap rows | Inline or modal |
| > 1024px | Two-column (list + sidebar) | Wrap rows | Inline or modal |

---

## Open Questions

1. **Product image upload flow in the business dashboard.** Discussed at high level (camera capture, auto-crop). Full UX to be detailed in S-01 (Store Dashboard).
2. **Full-text search for large catalogs.** PostgreSQL `to_tsvector` is used for the search index. If stores grow past ~500 products, consider caching the product list on the frontend for instant client-side filtering. Unlikely at launch.
3. **Price-by-weight products.** Some products (deli meat, produce) are sold by weight, where the exact total depends on what the scale says. For partnered stores, price is listed as "$X.XX/lb" and the customer selects a weight estimate. Exact pricing determined at fulfillment. To be detailed in C-04 (checkout handles the adjustment).
4. **Multi-store cart (DoubleDash).** Deferred to a future feature. Data model supports it — `cart_items` already has `store_id` per item. The constraint is in the checkout flow and fee calculation, not the data.
5. **Quick-add refresh job.** The weekly cron that regenerates quick-add suggestions from order history. Simple: query top 20 most-ordered item descriptions per store from completed shopping lists, insert/update `quick_add_suggestions`. Detailed in A-03 (Scheduled Jobs).

---

## ECS Fargate Notes

- `GET /api/stores/:storeId/products` is the heaviest catalog endpoint. At < 200 products per store, return everything in one page and let the frontend filter by category and search client-side. No pagination needed at launch.
- Product images are the heaviest assets on this page. Use CloudFront with aggressive caching (24-hour TTL minimum). Product images rarely change.
- The shopping list builder is lightweight — all operations are single-row inserts/updates. No performance concerns.
- The `cart_items` table is queried on every product card render (to show `cartQuantity`). At < 50 items per cart, this is instant. The cart check is included in the products endpoint response to avoid a separate API call.

## Third-Party API Costs for This Feature

| Service | Operation | Cost | Monthly Estimate |
|---|---|---|---|
| S3 + CloudFront | Product images (serving) | $0.085/GB transfer | ~$0.50-2.00 |
| S3 | Product image storage | $0.023/GB/month | < $0.01 |
| **Total** | | | **~$0.50-2.00/month** |
