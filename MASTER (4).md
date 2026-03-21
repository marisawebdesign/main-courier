# MainCourier — Master Project Document

> **Status:** Planning & Pre-Development
> **Last Updated:** 2026-03-21
> **Stack:** Next.js (S3/CloudFront) · Node.js API (ECS Fargate) · PostgreSQL (RDS) · Twilio · Stripe · Google Maps API
> **Audit Framework:** [AUDIT-FRAMEWORK.md](AUDIT-FRAMEWORK.md) — Reference this document when auditing any spec or code in this project.

---

## Project Overview

A delivery web application serving a 50-mile radius around Danforth, Maine (pop. ~590, Washington County). The platform connects underserved rural households with grocery stores in hub towns (Houlton, Calais, Lincoln), local cottage food producers operating under Maine's Food Sovereignty Act, and community food surplus/rescue programs.

The platform serves four user roles through a single Next.js application with route-protected sections:
- **Customer** — Orders groceries and local food for delivery or hub pickup
- **Store** — Manages product catalog, receives and fulfills orders
- **Driver** — Manages pickup, delivery routes, substitutions, and status updates
- **Developer/Admin** — System configuration, zone management, analytics, compliance

---

## Infrastructure Summary

| Component | Service | Details |
|-----------|---------|---------|
| Frontend | Next.js → S3 + CloudFront | Static export, custom domain via Route 53, ACM SSL cert |
| Backend API | ECS Fargate (single task) | Node.js, 0.25 vCPU / 0.5 GB RAM, behind ALB with ACM cert |
| Database | RDS PostgreSQL (db.t3.micro) | PostGIS extension for geospatial queries |
| File Storage | S3 | Product images, profile pictures, delivery photos, store flyers |
| Notifications | Twilio (all SMS — transactional and marketing) | SMS ordering, verifications, delivery updates, marketing broadcasts |
| Payments | Stripe | Card payments, future subscription billing, producer payouts |
| Maps & Geocoding | Google Maps API | Address geocoding, distance calculation, route sequencing |
| Secrets | AWS Secrets Manager | All API keys, DB credentials, third-party tokens |
| Monitoring | CloudWatch | Logs, metrics, alarms |
| DNS & SSL | Route 53 + ACM | Domain management and certificate provisioning |
| Container Registry | ECR | Docker images for the API service |

### Networking Architecture

```
Internet
  │
  ├─── CloudFront ──── S3 (Next.js static frontend)
  │
  └─── ALB (public subnet, ACM SSL termination)
         │
         └─── ECS Fargate Task (public subnet, public IP)
                │
                ├─── RDS PostgreSQL (private subnet, SG locked to Fargate only)
                ├─── S3 (file storage, via IAM role)
                ├─── Twilio API (outbound HTTPS)
                ├─── Stripe API (outbound HTTPS)
                └─── Google Maps API (outbound HTTPS)
```

- Fargate task runs in a **public subnet with a public IP** to eliminate the $32/month NAT gateway cost
- RDS runs in a **private subnet**, security group allows inbound 5432 only from the Fargate task security group
- ALB handles HTTPS termination; Fargate container receives plain HTTP
- WAF rate-based rule on the ALB ($6/month) provides blanket DDoS and abuse protection

### Global Security Standards

These apply to ALL features and endpoints across the entire application. Every item here is either a one-time setup or a code pattern — none require ongoing maintenance or admin configuration.

1. **All database queries use parameterized statements** — never string concatenation, including within ORMs and query builders. No exceptions.
2. **All user inputs validated server-side and rejected if invalid** — enforce generous length limits (100 chars for names, 300 chars for addresses), type checking, format validation. Do not attempt to sanitize bad input into good input — reject it with a clear 400 error and specific field-level messages. Never use `dangerouslySetInnerHTML` or build raw HTML strings anywhere in the codebase.
3. **Phone numbers stored in E.164 format** (`+12075550123`) — normalized and validated on input before any database write or API call. The `normalizePhone()` utility defined in C-01 must be used everywhere a phone number is received.
4. **Session cookies set with** `httpOnly: true`, `secure: true`, `sameSite: 'lax'`. No session data in `localStorage` or `sessionStorage`. `sameSite: 'lax'` (not `strict`) is required so that customers clicking SMS links arrive authenticated.
5. **Security headers on all API responses** — applied via a global middleware that runs on every response:
   - `X-Content-Type-Options: nosniff`
   - `X-Frame-Options: DENY`
   - `X-XSS-Protection: 1; mode=block`
   - `Strict-Transport-Security: max-age=31536000; includeSubDomains`
   - `Referrer-Policy: strict-origin-when-cross-origin`
   - `Permissions-Policy: camera=(), microphone=(), geolocation=(self)`
   - **Content-Security-Policy: Skip at MVP.** CSP requires whitelisting every third-party domain (Google Maps, Stripe.js, etc.) and breaks silently when misconfigured. React's built-in JSX escaping and input validation provide sufficient XSS protection for launch. Add CSP once the app is stable and all third-party integrations are finalized.
6. **CORS origin read from Secrets Manager** — production locked to the exact CloudFront domain, development uses `localhost`. Configured once per environment via the `FRONTEND_ORIGIN` secret. Never use wildcard (`*`) origins.
7. **All secrets stored in AWS Secrets Manager** — never in code, Dockerfiles, ECS task definition environment variables, or committed to git (even private repos). The Fargate task reads from Secrets Manager at startup and holds values in memory only. Local development uses `.env` files listed in `.gitignore` and never committed. **The following are secrets:** all API keys, all tokens, database credentials, session signing secrets, and any connection strings.
8. **RDS encryption at rest enabled, automated backups with 7-day retention, deletion protection on.** These are checkboxes during RDS setup. Encryption uses AWS KMS, costs nothing extra, and means customer data is encrypted on disk.
9. **CloudWatch alarms** — 5 alarms only, all emailing you and your partner via a single SNS topic. Only alert on things that require human action:
   - Fargate task stopped (the app is down — ECS will auto-restart, but you should know)
   - API 5xx error rate > 10% over 5 minutes (something is seriously broken)
   - RDS CPU > 90% sustained for 10 minutes (database is struggling)
   - RDS storage > 80% (running out of space — rare but catastrophic if ignored)
   - ALB unhealthy host count > 0 for 5 minutes (API isn't responding to health checks)
   - **No Twilio alarms** — SMS delivery failures are often carrier-side and not actionable. Log Twilio delivery statuses and review periodically if customers report missing messages.
10. **WAF rate-based rule on ALB as the primary rate limiter** ($6/month, configured once). Application-level rate limiting on auth endpoints only, backed by the PostgreSQL `rate_limits` table with automated hourly cleanup. WAF handles 90% of abuse scenarios with zero application code.
11. **Logging standards** — log every request with: timestamp, endpoint path, customer ID (if authenticated), and outcome (success or HTTP status code). Log errors with the full stack trace to CloudWatch. **Never log:** full session tokens, credit card numbers, SMS verification codes, or any full secret values. Log the first 8 characters of tokens if needed for debugging correlation.
12. **Customer data deletion** — manual SQL runbook documented in this project (see below). Orders anonymized (PII stripped but financial data retained), customer record deleted, sessions cascaded. No self-service deletion feature at MVP. This will be a rare event — maybe a few times per year.
13. **HTTPS everywhere, no exceptions** — ALB redirects all HTTP to HTTPS, CloudFront serves HTTPS only, all third-party API calls (Twilio, Stripe, Google Maps) are over HTTPS, no mixed content.
14. **Error responses never expose internals** — a single global error handler catches all unhandled errors and returns `{ "error": "Something went wrong. Please try again.", "code": "INTERNAL_ERROR" }` to the client. Full stack traces, SQL errors, file paths, and third-party API error details are logged to CloudWatch only. The customer sees a generic message. This is critical for both security and for reducing confused support contacts.
15. **No sensitive data in URLs** — session tokens in cookies only, temporary tokens in request bodies only, PII never in query parameters or URL paths. The only exception is the referral code in URL query strings (`?ref=XXXXXX`), which is non-sensitive by design.
16. **GitHub Dependabot enabled** for automated dependency security pull requests. Manual `npm audit` run monthly as a health check. No build-blocking audit gate — `npm audit` produces too many false positives to be a reliable deploy gate and will create unnecessary maintenance friction.

### Customer Data Deletion Runbook

For the rare case when a customer requests their data be deleted. Run manually via a database client connected to RDS:

```sql
-- Customer data deletion runbook
-- Replace CUSTOMER_ID with the actual customer ID
-- Verify the correct customer before running

BEGIN;

-- Remove active sessions
DELETE FROM sessions WHERE customer_id = CUSTOMER_ID;

-- Anonymize order records (keep for financial records, strip PII)
UPDATE orders SET
  customer_name = 'Deleted Customer',
  delivery_address = 'Deleted',
  delivery_town = 'Deleted',
  delivery_zip = 'Deleted',
  phone = 'Deleted'
WHERE customer_id = CUSTOMER_ID;

-- Remove the customer record (cascades to any remaining FK references)
DELETE FROM customers WHERE id = CUSTOMER_ID;

COMMIT;
```

### Scheduled Maintenance Jobs

These run as ECS scheduled tasks or in-app cron (node-cron):

| Job | Schedule | Description |
|-----|----------|-------------|
| Session cleanup | Nightly 3:00 AM EST | Delete expired rows from `sessions` table |
| Pending verification cleanup | Nightly 3:00 AM EST | Delete expired/used rows from `pending_verifications` table |
| Geocode cache cleanup | Weekly Sunday 3:00 AM | Delete rows older than 30 days from `geocode_cache` table |
| Rate limit cleanup | Hourly | Delete expired rows from `rate_limits` table |

---

## Feature Documents

Each feature is documented in its own file with full frontend and backend specifications. These documents are written to be consumed by Claude Code during development.

### Customer Interface

| # | Feature | Document | Status |
|---|---------|----------|--------|
| 1 | First-Time Onboarding & Signup | [specs/C-01-onboarding.md](specs/C-01-onboarding.md) | ✅ Specified (v1) |
| 2 | Home Screen & Navigation | [specs/C-02-home-navigation.md](specs/C-02-home-navigation.md) | ✅ Specified (v1) |
| 3 | Product Browsing & Shopping Lists | [specs/C-03-product-browsing.md](specs/C-03-product-browsing.md) | ✅ Specified (v1) |
| 4 | Cart & Checkout | [specs/C-04-cart-checkout.md](specs/C-04-cart-checkout.md) | ✅ Specified (v1) |
| 5 | Order History, Smart Reorder & Post-Delivery | [specs/C-05-orders-reorder.md](specs/C-05-orders-reorder.md) | ✅ Specified (v1) |
| 6 | Community & Route Building | [specs/C-06-community-routes.md](specs/C-06-community-routes.md) | ✅ Specified (v1) |
| 7 | Delivery Tracking & Communication | features/07-delivery-tracking.md | 🔲 Not started |
| 8 | Rescue Bags & Surplus | features/08-rescue-bags.md | 🔲 Not started |
| 9 | Buying Club & Producer Profiles | features/09-buying-club.md | 🔲 Not started |
| 10 | Account & Settings | features/10-account-settings.md | 🔲 Not started |
| 11 | Accessibility & Low-Bandwidth | features/11-accessibility.md | 🔲 Not started |
| 12 | Community & Social Features | features/12-community.md | 🔲 Not started |

### Store Interface

| # | Feature | Document | Status |
|---|---------|----------|--------|
| 13 | Store Onboarding & Profile | features/13-store-onboarding.md | 🔲 Not started |
| 14 | Product Catalog Management | features/14-store-catalog.md | 🔲 Not started |
| 15 | Order Receiving & Fulfillment | features/15-store-orders.md | 🔲 Not started |
| 16 | Substitution & Stock Communication | features/16-store-substitutions.md | 🔲 Not started |
| 17 | Rescue Bag / Surplus Management | features/17-store-surplus.md | 🔲 Not started |
| 18 | Store Reporting & Analytics | features/18-store-reporting.md | 🔲 Not started |
| 19 | Store Promotions & Flyers | features/19-store-promotions.md | 🔲 Not started |
| 20 | Store Communication & Support | features/20-store-communication.md | 🔲 Not started |

### Driver Interface

| # | Feature | Document | Status |
|---|---------|----------|--------|
| 21 | Daily Route Dashboard | features/21-driver-dashboard.md | 🔲 Not started |
| 22 | Pickup Checklist & Store Workflow | features/22-driver-pickup.md | 🔲 Not started |
| 23 | Substitution Management | features/23-driver-substitutions.md | 🔲 Not started |
| 24 | Delivery Route & Navigation | features/24-driver-navigation.md | 🔲 Not started |
| 25 | Order Status Updates | features/25-driver-status.md | 🔲 Not started |
| 26 | Cottage Food Pickup Integration | features/26-driver-cottage-pickup.md | 🔲 Not started |
| 27 | Cash & Payment Handling | features/27-driver-cash.md | 🔲 Not started |
| 28 | Hub Drop-off Management | features/28-driver-hubs.md | 🔲 Not started |
| 29 | Issue Reporting & Exceptions | features/29-driver-issues.md | 🔲 Not started |
| 30 | End-of-Day Summary & Mileage | features/30-driver-summary.md | 🔲 Not started |

### Developer / Admin Interface

| # | Feature | Document | Status |
|---|---------|----------|--------|
| 31 | System Configuration & Feature Flags | features/31-admin-config.md | 🔲 Not started |
| 32 | Zone & Route Management | features/32-admin-zones.md | 🔲 Not started |
| 33 | User & Account Management | features/33-admin-users.md | 🔲 Not started |
| 34 | Producer & Food Sovereignty Admin | features/34-admin-producers.md | 🔲 Not started |
| 35 | Store Partnership Management | features/35-admin-stores.md | 🔲 Not started |
| 36 | Financial Dashboard & Reporting | features/36-admin-financials.md | 🔲 Not started |
| 37 | Analytics & Business Intelligence | features/37-admin-analytics.md | 🔲 Not started |
| 38 | Notification & Communication Mgmt | features/38-admin-notifications.md | 🔲 Not started |
| 39 | Monitoring & System Health | features/39-admin-monitoring.md | 🔲 Not started |
| 40 | Deployment & Infrastructure | features/40-admin-deployment.md | 🔲 Not started |

### Backend Systems

| # | System | Document | Status |
|---|--------|----------|--------|
| 41 | Database Schema (Complete) | features/41-database-schema.md | 🔲 Not started |
| 42 | Authentication & Authorization | features/42-auth-system.md | 🔲 Not started |
| 43 | Order Processing Pipeline | features/43-order-pipeline.md | 🔲 Not started |
| 44 | Twilio Integration Layer | features/44-twilio-integration.md | 🔲 Not started |
| 45 | Payment & Financial Processing | features/45-payment-processing.md | 🔲 Not started |
| 46 | Route Planning & Logistics Engine | features/46-route-planning.md | 🔲 Not started |
| 47 | Food Sovereignty Compliance | features/47-sovereignty-compliance.md | 🔲 Not started |
| 48 | Notification & Scheduling Engine | features/48-notification-engine.md | 🔲 Not started |
| 49 | Product & Catalog Management | features/49-catalog-system.md | 🔲 Not started |
| 50 | Analytics & Reporting Pipeline | features/50-analytics-pipeline.md | 🔲 Not started |

### Cross-Cutting Concerns

| # | Concern | Document | Status |
|---|---------|----------|--------|
| 51 | Security & Data Privacy | features/51-security.md | 🔲 Not started |
| 52 | Error Handling & Edge Cases | features/52-error-handling.md | 🔲 Not started |
| 53 | Offline & Degraded Mode | features/53-offline-mode.md | 🔲 Not started |
| 54 | Legal & Compliance | features/54-legal-compliance.md | 🔲 Not started |
| 55 | Testing & QA Strategy | features/55-testing.md | 🔲 Not started |
| 56 | Launch & Rollout Plan | features/56-launch-plan.md | 🔲 Not started |

---

## Shared Database Tables

Tables referenced across multiple features. Full column definitions in [features/41-database-schema.md](features/41-database-schema.md).

### Core Tables
- `customers` — Customer profiles, addresses, zone assignments, buying club status
- `sessions` — Active authentication sessions
- `pending_verifications` — Temporary tokens between phone verify and account creation
- `zones` — Delivery zone definitions with geospatial boundaries
- `clusters` — Neighborhood clusters within zones for group ordering
- `stores` — Partner store profiles and configurations (includes `partnership_tier`: partnered or unpartnered)
- `producers` — Cottage food producer profiles and sovereignty track
- `hubs` — Community pickup hub locations and capacity
- `products` — Product catalog (partnered store items and cottage food items only)
- `orders` — Order headers with status, totals, delivery info
- `order_items` — Individual line items (catalog items for partnered stores, freeform shopping list items for unpartnered)
- `deliveries` — Delivery route assignments and tracking
- `waitlist` — Addresses outside the current service area
- `surprise_bags` — Rescue/surplus food bags from stores and producers

### Batch Route Tables
- `route_interests` — Customer interest signals for store routes (community-driven route building)
- `weekly_batches` — One batch per store per zone per week (the core scheduling unit)
- `batch_participants` — Customer commitment to a specific weekly batch
- `route_history` — Route maturity tracking for admin (consecutive weeks, avg participants, maturity stage)
- `shopping_fee_config` — Personal shopping fee settings per unpartnered store (base fee, floor minimum)

### Bulk Special Tables
- `bulk_specials` — One-time distribution routes from partnered vendors (product, quantity, pricing, radius, cutoff)
- `bulk_special_participants` — Customer commitments to bulk specials (quantity, fees, status)

### Product & Cart Tables
- `product_categories` — Per-store/producer product categories managed by the business
- `products` — Product catalog with photos, pricing, allergens (partnered stores and producers only)
- `cart_items` — Unified cart supporting both catalog items (partnered) and shopping list items (unpartnered)
- `shopping_list_items` — Working shopping list items linked to batch participations (unpartnered stores)
- `saved_list_templates` — Customer-saved shopping list templates per store
- `saved_list_template_items` — Items within saved templates
- `item_aliases` — Fuzzy matching dictionary for shopping list deduplication (moved from Batch Route Tables)
- `quick_add_suggestions` — Per-store quick-add button data, refreshed weekly from order history

### Order & Payment Tables
- `orders` — Order headers with status, pricing, auth/capture amounts, delivery details, payment method (includes `delivered_at` for time-gated actions)
- `order_items` — Line items per order (catalog or shopping list type, with fulfillment fields)
- `order_flags` — Admin review queue for payment issues, negative ratings, and customer-reported issues
- `cash_verification_log` — Audit trail for admin cash payment verification decisions
- `delivery_ratings` — Customer thumbs up/down per delivery (one per order)
- `order_issues` — Customer issue reports with affected items, photos, resolution tracking

### Configuration Tables
- `fee_tiers` — Delivery and hub fee schedules per zone
- `group_discount_tiers` — Neighbor group discount rules
- `service_fee_config` — Service fee percentages and caps per zone
- `feature_flags` — Runtime feature toggles
- `notification_templates` — SMS/email message templates
- `sovereignty_municipalities` — Towns with food sovereignty ordinances
- `product_categories` — Allowed product types per sovereignty track
- `system_config` — Key-value pairs for global settings

### Utility Tables
- `rate_limits` — Rate limiting counters for API abuse prevention
- `geocode_cache` — Cached Google Maps geocoding results (30-day TTL)
- `referral_credits` — Referral tracking and credit ledger

---

## Monthly Cost Estimate

| Service | Monthly Cost |
|---------|-------------|
| S3 (frontend + files) | $1–3 |
| CloudFront (CDN) | $1–2 |
| Route 53 (DNS) | $0.50 |
| ACM (SSL) | Free |
| RDS PostgreSQL (db.t3.micro) | $15–20 |
| ECS Fargate (0.25 vCPU / 0.5 GB) | $9–13 |
| ALB | $16–18 |
| WAF (1 rate-based rule) | $6 |
| ECR (container storage) | $1 |
| Secrets Manager | $0.40 |
| CloudWatch | $0 (free tier) |
| Google Maps API | $0 (under $200/mo free credit) |
| Stripe (2.9% + $0.30 on ~$2K/mo) | $88–120 |
| Twilio (transactional SMS) | $15–25 |
| Domain registration | $1 (amortized) |
| **Total** | **$154–210/month** |

---

## Development Conventions for Claude Code

When implementing any feature document:

1. **Read this master document first** for infrastructure context, security standards, and shared table definitions
2. **Read the specific feature document** for detailed frontend and backend specifications
3. **Use parameterized queries for ALL database operations** — no exceptions
4. **Normalize phone numbers to E.164 before any database write or comparison**
5. **All API endpoints that require authentication must use the `requireAuth` middleware** defined in [specs/C-01-onboarding.md](specs/C-01-onboarding.md)
6. **All user-facing error messages must be generic** — never expose database errors, stack traces, or internal state. Use the global error handler pattern defined in the security standards.
7. **All config values that might change must be read from config tables at runtime** — never hardcode fees, zone boundaries, templates, or feature states. If the partner might ever say "can we change X," then X belongs in a config table editable through the admin interface, not in code.
8. **File uploads go through S3 pre-signed URLs** — never route binary data through the API container
9. **Third-party API keys are read from Secrets Manager at container startup** — cached in memory for the container lifetime. Never from environment variables in task definitions, .env files in production, or hardcoded values.
10. **Every API endpoint must validate all inputs server-side and reject invalid input** — do not attempt to sanitize bad input into good input. Return a 400 with specific field-level error messages.
11. **The global error handler must be the last middleware registered** — it catches everything that wasn't handled by specific endpoint error responses and returns a safe generic message to the client while logging full details to CloudWatch.
12. **Never use `dangerouslySetInnerHTML`** in React components. Never build raw HTML strings on the server for client rendering.

---

## UI/UX Design Standards

### Visual Design

The MainCourier design language is clean, sharp, and professional. It should feel trustworthy and simple, not playful or trendy.

**Color palette (starting point, expandable):**
- Primary: Opaque blue (#1A56DB or similar strong, trustworthy blue). Used for CTAs, active states, links, and key accents.
- Primary hover/active: Slightly darker shade of primary blue
- Background: White (#FFFFFF) for content areas, light gray (#F9FAFB) for page backgrounds
- Text: Near-black (#111827) for body text, medium gray (#6B7280) for secondary text
- Success: Green (#059669) for delivered, confirmed, positive states
- Warning: Amber (#D97706) for pending, attention-needed states
- Error: Red (#DC2626) for failures, declines, critical issues
- Border/divider: Light gray (#E5E7EB)
- Card backgrounds: White with subtle border or light shadow

**Typography:**
- Use the system font stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`. No custom font loading (saves bandwidth on slow rural connections).
- Body text: 16px minimum on mobile
- Headings: Bold weight, same font family, sized proportionally
- No italic for emphasis. Use bold or color instead.

**Components:**
- Buttons: Solid fill with primary blue for primary actions, outlined/ghost for secondary actions. Rounded corners (4-6px radius), not pill-shaped.
- Cards: White background, subtle border (1px #E5E7EB) or light box-shadow. No heavy drop shadows.
- Form inputs: Clean bordered rectangles with clear labels above the field (not floating labels, not placeholder-only).
- Status indicators: Small colored dot or pill + text label. NEVER color alone.

**No emojis in the UI.** Status indicators, store type icons, delivery type icons, and all other visual markers must use simple text labels, SVG icons, or colored indicators. Emojis render inconsistently across devices and look unprofessional. Where the spec mockups show emojis (e.g., "🏪", "🌾", "📦", "✅"), replace with appropriate SVG icons or text labels during implementation.

**No decorative elements.** No gradients, no background patterns, no illustrations on functional pages. The interface is a tool, not a brochure.

**Whitespace is the primary design tool.** Generous padding, clear visual hierarchy through spacing and typography weight, not through color or decoration.

### Writing & Copy Standards

All customer-facing text in the application must follow these rules:

**No AI-generated syntax.** Specifically:
- No em dashes. Use commas, periods, or parentheses instead.
- No "not X, but Y" constructions. Rephrase directly: say what it IS, not what it isn't then is.
- No "It's worth noting that..." or "Importantly,..." or "Interestingly,..." throat-clearing phrases.
- No semicolons in customer-facing copy. Use two sentences.
- No exclamation marks except on genuine celebration moments (order confirmed, delivery complete). One per screen maximum.

**Be direct.** Say what the user needs to know in the fewest clear words.
- Good: "Your card will be held for $45.49 until your order is fulfilled."
- Bad: "It's important to note that a temporary authorization hold of $45.49 will be placed on your card, which will be adjusted to the final amount once your order has been successfully fulfilled."

**Use plain language.** The target audience is rural Maine, median age 48, mixed tech literacy.
- Good: "Pay with cash at delivery"
- Bad: "Utilize our cash-on-delivery payment modality"
- Good: "Order by Wednesday 8pm for Thursday delivery"
- Bad: "Submit your order prior to the weekly batch cutoff deadline"

**Error messages are helpful and specific.**
- Good: "Your card was declined. Try a different card."
- Bad: "An error occurred while processing your payment request."
- Good: "Add at least one item to your list before submitting."
- Bad: "Validation error: shopping list items array must not be empty."

**Confirmation messages are brief.**
- Good: "Order placed. Your card is held for $45.49."
- Bad: "Congratulations! Your order has been successfully placed! A temporary hold of $45.49 has been applied to your payment method. You will receive a notification when your order is on its way!"

These writing standards apply to ALL customer-facing text: buttons, labels, error messages, confirmation screens, SMS notifications, empty states, tooltips, and any other text the customer reads. They do NOT apply to admin dashboard text or developer documentation.

When Claude Code generates UI copy, it must follow these rules. If a spec document contains copy that violates these rules, the implementation should correct it.

---

## Developer Notes

> These notes are written for you, the developer, to provide context and practical guidance that doesn't fit neatly into spec documents. Read these before you start building — they'll save you time and prevent common mistakes.

### How to Debug Customer Issues

When a customer reports a problem ("my order didn't go through," "I never got my verification code," "the app just shows an error"), here's the process:

1. Your partner asks the customer: **"When did you try?"** and **"What were you trying to do?"** — you need an approximate time and the action they were attempting.
2. Open CloudWatch Logs for the `/ecs/your-api` log group.
3. Filter by the approximate time window. If you know their phone number or customer ID, add that as a search filter.
4. Every API request is logged with: timestamp, endpoint path, customer ID (if authenticated), and outcome. Find the matching request.
5. If it was an error, the full stack trace is in the log entry. This tells you exactly what went wrong — database error, third-party API timeout, validation failure, etc.
6. Fix the issue, or if it was transient (network timeout, Twilio hiccup), tell the customer to try again.

**You do not need correlation IDs, distributed tracing, or any fancy observability tooling at your scale.** At 300 orders/month, CloudWatch log search by time and customer ID is sufficient. You can add correlation IDs later if the log volume ever makes this approach insufficient — but at your volume it won't.

### How to Handle "It Works on My Machine" Situations

Your local development environment and the Fargate production environment will have subtle differences. The most common gotchas:

- **Database connections:** Locally you connect to PostgreSQL on `localhost:5432`. In Fargate, the connection string comes from Secrets Manager and points to the RDS endpoint in a private subnet. Make sure your code reads the connection config from environment/secrets, never hardcoded.
- **File uploads:** Locally you might save files to disk for testing. In Fargate, the filesystem is ephemeral — anything written to disk disappears when the task restarts. All persistent files must go to S3.
- **Environment variables:** Locally you use a `.env` file. In production, everything comes from Secrets Manager. Make sure your app startup code handles both — check Secrets Manager first, fall back to environment variables for local development.
- **Time zones:** Always use UTC in the database (`TIMESTAMPTZ` columns, which the schema already specifies). Convert to Eastern time only in the frontend when displaying to users. Never store local times in the database.

### How Config Tables Prevent Code Deploys

One of the most important architectural decisions in this project is that **business rules live in database config tables, not in code.** Here's why this matters practically:

When your partner says "can we lower the delivery fee for orders over $100?" — if that fee tier is hardcoded, you need to: change the code, test it, build a new Docker image, push to ECR, update the ECS service, and wait for the rolling deployment. That's 15-30 minutes of your time and a brief service interruption.

When that fee tier is in a `fee_tiers` config table with an admin interface, your partner opens the admin panel, changes the number, and hits save. It's live in seconds. You're not involved at all.

**Rule of thumb:** Any number, threshold, text template, schedule, or on/off toggle that a non-developer might ever want to change should be in a config table. The code reads these tables at runtime and behaves accordingly. The code itself should only change when you're adding new features or fixing bugs — never for business rule adjustments.

Config tables that are defined in this project:
- `fee_tiers` — delivery and hub fee schedules per zone
- `group_discount_tiers` — neighbor group discount rules
- `service_fee_config` — service fee percentages and caps per zone
- `feature_flags` — runtime feature toggles (rescue bags on/off, new zone enabled, etc.)
- `notification_templates` — SMS and email message templates with variable placeholders
- `sovereignty_municipalities` — towns with food sovereignty ordinances
- `product_categories` — allowed product types per sovereignty track
- `system_config` — key-value pairs for global settings (current terms version, etc.)

### How Session Auth Works (The Full Picture)

The auth system uses phone number + SMS code (via Twilio Verify) instead of passwords. Here's the mental model:

1. **Customer identity = their phone number.** There is no username or email.
2. **Authentication = proving they have access to that phone.** They receive a one-time code via SMS and type it in.
3. **Session = a long random token stored in a cookie.** After verification, the server generates a 64-character hex token (`crypto.randomBytes(32).toString('hex')`), stores it in the `sessions` table linked to the customer ID, and sets it as an HTTP-only secure cookie with a 90-day expiry.
4. **Every subsequent request:** The browser automatically sends the cookie. The server's `requireAuth` middleware reads the token from the cookie, looks it up in the `sessions` table, confirms it hasn't expired, and attaches the customer ID to the request object. If the token is missing, invalid, or expired, the middleware returns a 401 and the frontend redirects to the phone verification screen.
5. **There is no "login" vs. "signup" distinction from the user's perspective.** They enter their phone number and a code. The server checks if the phone exists in the `customers` table — if yes, it's a returning user and they get a session. If no, it's a new user and they continue to the account setup screen.
6. **Session persistence:** The cookie persists for 90 days in the browser. If the customer "installs" the PWA to their home screen, the browser context (including cookies) persists across app opens. They'll stay logged in for 90 days without ever re-entering a code.
7. **When to re-verify:** After 90 days, if they switch devices, or if they clear their browser data. The experience is: enter phone number → get code → enter code → back in with all their data intact. Takes 15 seconds.

### Logging: What to Log and What Never to Log

**Always log (on every request):**
- Timestamp (ISO 8601, UTC)
- HTTP method and endpoint path (`POST /api/auth/verify-code`)
- Customer ID if authenticated (from the session middleware), or `"unauthenticated"`
- Response status code (200, 400, 401, 500, etc.)
- Response time in milliseconds

**Log on errors (in addition to the above):**
- Full error message
- Full stack trace
- Request body (with sensitive fields redacted — see below)

**Never log, under any circumstances:**
- Full session tokens (log first 8 characters only if needed: `a8f2c9e1...`)
- SMS verification codes
- Stripe card numbers or payment tokens
- Full passwords (you don't have these, but state the rule anyway)
- Any value from Secrets Manager

**Redact in request body logs:**
- The `code` field in verify-code requests
- Any field named `password`, `token`, `secret`, or `key`

A simple logging middleware example:
```javascript
function requestLogger(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      method: req.method,
      path: req.path,
      customerId: req.customerId || 'unauthenticated',
      status: res.statusCode,
      ms: Date.now() - start
    }));
  });
  next();
}
```

This produces one JSON log line per request that CloudWatch can index and search. When something goes wrong, you filter by time range and status code (e.g., all 500s in the last hour) and you'll find the problem.

### Understanding Fargate (For Someone New To It)

ECS Fargate runs your Docker container without you managing any servers. Here's the mental model:

- **Your code** runs inside a Docker container. You build the container image locally, push it to ECR (Amazon's container registry), and tell ECS "run this image."
- **A Task** is one running copy of your container. You run exactly 1 task. It's your API server.
- **A Service** is the thing that keeps your task running. If the task crashes, the service starts a new one automatically. If you deploy a new version, the service starts the new task, waits for it to be healthy, then stops the old one — zero downtime.
- **You never SSH into anything.** If you need to inspect the running container, use ECS Exec (`aws ecs execute-command`) which gives you a shell inside the container.
- **The container is ephemeral.** If it restarts, anything written to the local filesystem is gone. All persistent data goes to RDS (database) or S3 (files). This is why Secrets Manager is important — you can't store config files on the container's disk.
- **Cost:** ~$9-13/month for one always-on task at 0.25 vCPU / 0.5 GB RAM. This is fixed whether you serve 10 requests or 10,000 per day.

### Deploy Process (Once Set Up)

```bash
# 1. Build the Docker image
docker build -t your-api .

# 2. Tag it for ECR
docker tag your-api:latest YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/your-api:latest

# 3. Push to ECR
docker push YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/your-api:latest

# 4. Update the ECS service to pull the new image
aws ecs update-service --cluster your-cluster --service your-api --force-new-deployment

# 5. Watch the deployment (optional)
aws ecs wait services-stable --cluster your-cluster --services your-api
```

The ECS service handles the rest — starts a new task with the new image, waits for the ALB health check to pass, then stops the old task. Total deploy time: 2-3 minutes. Zero downtime.

### When Things Go Wrong at 10pm

Realistic scenarios and what to do:

1. **"The app won't load"** — Check if the Fargate task is running (`aws ecs describe-services`). If it stopped, ECS should auto-restart it within 60 seconds. If it keeps crashing, check CloudWatch logs for the startup error — likely a bad secret value, database connection failure, or a code bug in the latest deploy.

2. **"I can't log in / verification code never arrives"** — Check Twilio's dashboard (console.twilio.com) for delivery status. If Twilio is down, there's nothing you can do — tell the customer to try again in a few minutes. If Twilio shows the message was delivered but the customer says they didn't get it, the carrier may have filtered it as spam. Have them try the "Call me instead" option.

3. **"Orders aren't saving"** — Check CloudWatch logs for 500 errors on the order endpoints. Most likely a database issue — RDS out of storage, connection pool exhausted, or a schema mismatch after a bad migration. Check the RDS CloudWatch metrics for CPU and storage.

4. **"Everything is slow"** — Check RDS CPU in CloudWatch. If it's spiking, look for slow queries (enable RDS Performance Insights, free for db.t3.micro). Most likely a missing index — find the slow query and add the index. If RDS CPU is fine, check if the Fargate task is at memory or CPU limits.

5. **"I deployed and now everything is broken"** — Roll back by redeploying the previous Docker image tag. Tag your images with version numbers (not just `latest`) so you can always go back: `docker tag your-api:latest your-api:v1.2.3`
