# MainCourier — Audit Framework

> **Purpose:** This document is a reference for any Claude instance to use when auditing spec documents, generated code, architectural decisions, or system configuration for the MainCourier project. When the project owner says "audit this," "review this," "check this," or anything similar — read this framework first, then apply every relevant section to the document or code being reviewed.
> **Tone:** Be blunt. Flag problems clearly. Do not soften findings to be polite. The project owner would rather hear about a problem now than discover it in production with real customers and real money.
> **Output format:** For each audit, produce findings organized by severity: CRITICAL (must fix before shipping), WARNING (should fix, risk if you don't), NOTE (improvement opportunity, not blocking), and ASK JOE (architectural or infrastructure decisions that warrant a second opinion from a senior engineer).

---

## Context About the Project Owner

The project owner is not a developer. They have a basic understanding of how code works but cannot read or write code independently. Every line of code in this project will be written by AI (Claude Code or similar). The project owner will handle direct inputs that AI cannot do — AWS console configuration, Stripe dashboard setup, domain registration, third-party account creation, etc.

The project owner has access to Joe, a senior software engineer at Indeed, who can provide expert guidance on architecture and infrastructure decisions. Joe's time is valuable — only escalate questions that are genuinely non-obvious, involve significant cost or reliability tradeoffs, or where Claude is uncertain. DO NOT escalate basic or well-established patterns. See the "Ask Joe" section below for escalation criteria.

This means:

1. **Nothing can be implied.** If a spec says "handle authentication" but doesn't specify HOW, Claude Code will make assumptions that may be wrong. Every spec must be explicit enough that an AI coding agent can implement it without ambiguity.
2. **Debugging will be AI-assisted.** When something breaks, the project owner will describe the symptoms to Claude and ask for help. The system must produce clear, readable error logs. Cryptic failures are unacceptable.
3. **The project owner cannot manually intervene in code.** If a bug requires "just change line 47 in auth.js," the owner needs Claude to do it. Every fix is a conversation. Minimize the need for fixes by getting the spec right.
4. **Configuration should be in the database, not in code.** Every value that might need to change (fee percentages, thresholds, radius limits, buffer amounts) should be in a config table editable through the admin dashboard. A code deploy to change a fee percentage is a failure of system design.
5. **The project owner's partner runs the day-to-day business.** She needs to understand the admin dashboard without technical training. Admin interfaces must be self-explanatory.
6. **All code will be implemented by Claude Code.** The specs must be complete enough that Claude Code can build the entire system from them, including connecting backend services, database setup, and deployment. Steps that require the project owner to do something manually (AWS console, domain setup, Stripe dashboard) must be called out explicitly as manual steps.

---

## Audit Category 1: Security

### 1.1 Authentication & Session Management

Check for:

- [ ] **Session tokens are cryptographically random.** Must use `crypto.randomBytes(32)` or equivalent. Not UUIDs, not sequential IDs, not timestamps.
- [ ] **Session tokens are in HTTP-only, Secure, SameSite cookies.** Never in localStorage, never in URL parameters, never in response bodies that JavaScript can read.
- [ ] **Session expiry is defined.** Every session must have a maximum lifetime and an inactivity timeout.
- [ ] **Phone verification codes have a short TTL.** 10 minutes maximum. Codes must be single-use (marked as used after verification, not deleted — deletion creates a race condition).
- [ ] **Rate limiting exists on all auth endpoints.** Phone verify, login, code submission. Without rate limiting, an attacker can brute-force 6-digit codes (1 million possibilities).
- [ ] **No sensitive data in URLs.** Session tokens, phone numbers, customer IDs — none of these belong in query parameters or URL paths. URLs are logged by proxies, CDNs, and browsers.

### 1.2 Payment Security

Check for:

- [ ] **Card data never touches the server.** Stripe Elements handles card input on the frontend. The backend only receives a Stripe `paymentMethodId` — never a card number, CVV, or expiry.
- [ ] **Stripe secret key is in AWS Secrets Manager.** Not in environment variables in code, not in a `.env` file committed to git, not hardcoded anywhere.
- [ ] **Stripe publishable key is the ONLY key exposed to the frontend.** It's public by design, but confirm the secret key isn't accidentally bundled.
- [ ] **Idempotency keys on all Stripe API calls.** Every PaymentIntent create, capture, and refund must have a unique idempotency key derived from the order ID. Without this, network retries can create duplicate charges.
- [ ] **Auth hold amounts are calculated server-side.** The frontend displays the amount but never computes it. A client-side calculation can be tampered with.
- [ ] **Fee calculations are server-side only.** Delivery fees, service fees, shopping fees — all computed by the API. The frontend renders them but doesn't calculate them.
- [ ] **Cash payment is gated by server-side verification.** The `cash_verified` flag is checked on the server, not trusted from the client. A modified client could claim cash verification.

### 1.3 Data Protection

Check for:

- [ ] **PII is minimized.** Only collect what's needed. Don't store data "just in case."
- [ ] **Database encryption at rest is enabled.** RDS PostgreSQL should have encryption enabled (AWS handles this, but verify it's configured).
- [ ] **No PII in logs.** CloudWatch logs should never contain phone numbers, addresses, card details, or full names. Log customer IDs, order IDs, and status codes — not personal data.
- [ ] **Error responses never expose internals.** SQL errors, stack traces, file paths, third-party API error details — none of these should reach the client. Return generic error messages. Log the details server-side.
- [ ] **The anonymous participant system in C-06 is truly anonymous.** The `/api/community/route/:storeId` endpoint must NEVER return customer IDs, names, or any identifying information in the participants array. Verify this at the code level — a developer might accidentally include the customer_id in a JOIN result.
- [ ] **Geocode data is fuzzy for community features.** Neighbor indicators on maps should use fuzzy area circles, not exact coordinates. A customer's exact location should only be visible to drivers and admin.

### 1.4 API Security

Check for:

- [ ] **Every endpoint checks authentication.** No public API endpoints except health checks. Every route handler must verify the session cookie before processing.
- [ ] **Authorization checks exist beyond authentication.** A logged-in customer should not be able to access another customer's orders, cart, or profile. Every query must filter by `customer_id` from the session — never trust a customer_id from the request body.
- [ ] **Admin endpoints are protected by admin role checks.** Not just "is authenticated" but "is authenticated AND is admin."
- [ ] **Driver endpoints are protected by driver role checks.** A customer session cannot access driver routes.
- [ ] **Store endpoints are protected by store role checks.** A customer cannot access the store dashboard.
- [ ] **Input validation exists on every endpoint.** String lengths, number ranges, enum values, required fields. Never trust client input. SQL injection, XSS, and command injection are all prevented by parameterized queries and input sanitization.
- [ ] **WAF is configured on the ALB.** AWS WAF with basic rules (rate limiting, SQL injection patterns, known bad IPs) provides a first line of defense.

### 1.5 Third-Party Key Management

Check for:

- [ ] **All API keys are in AWS Secrets Manager.** Stripe, Twilio, Google Maps — every key.
- [ ] **Google Maps API key is restricted.** In the Google Cloud Console, the key should be restricted to the CloudFront domain (HTTP referrer restriction). An unrestricted Maps key will be scraped and abused within hours.
- [ ] **Twilio credentials are not exposed to the frontend.** All SMS operations happen server-side.
- [ ] **No keys committed to git.** Check `.gitignore` for `.env` files. Check the git history for accidental key commits (they persist even after deletion).

---

## Audit Category 2: System Design & Reliability

### 2.1 Single Points of Failure

Check for:

- [ ] **Database backups are configured.** RDS automated backups with at least 7-day retention. Verify this is in the infrastructure setup steps.
- [ ] **What happens if the ECS Fargate task crashes?** There should be a health check configured on the ALB that restarts unhealthy tasks. ECS should be configured to maintain a desired count of 1 task (it auto-restarts on failure).
- [ ] **What happens if RDS goes down?** Multi-AZ is expensive at this scale, so single-AZ is acceptable — but the recovery plan must be documented. Point-in-time recovery from automated backups.
- [ ] **What happens if Stripe is down?** The checkout page should show "Payments are temporarily unavailable. Try again in a few minutes." No retry queue, no background processing. Simple is reliable.
- [ ] **What happens if Twilio is down?** SMS notifications fail silently (logged, not retried). The system continues to function — SMS is a notification, not a dependency for core functionality.
- [ ] **What happens if Google Maps API is down?** The map doesn't load, but the list view still works. Product browsing and checkout don't depend on Maps.

### 2.2 Data Integrity

Check for:

- [ ] **Foreign key constraints exist on all relationship columns.** Every `_id` column that references another table must have a `REFERENCES` constraint. Without FK constraints, orphaned records accumulate silently.
- [ ] **Unique constraints exist where business logic requires uniqueness.** One cart per customer per store. One batch participation per customer per batch. One route interest per customer per store. If the spec says "only one," there must be a `UNIQUE` constraint in the schema.
- [ ] **Check constraints exist for enum-like columns.** Status fields should have CHECK constraints limiting them to valid values. Without this, a typo in code creates an invalid status that breaks queries.
- [ ] **Decimal precision is appropriate for money.** All money columns should be `DECIMAL(8,2)` or `DECIMAL(6,2)`. Never use `FLOAT` or `REAL` for money — floating point arithmetic creates rounding errors.
- [ ] **Timestamps use TIMESTAMPTZ.** Not `TIMESTAMP` (which has no timezone) — always `TIMESTAMPTZ`. The server, database, and clients may be in different timezones. TIMESTAMPTZ stores everything in UTC and converts on retrieval.
- [ ] **Critical operations use database transactions.** Creating an order (insert order + insert items + clear cart + update participant status) must be atomic. If any step fails, everything rolls back. Check for `BEGIN/COMMIT/ROLLBACK` or ORM transaction wrappers.
- [ ] **Concurrent access is handled.** When two people join a batch simultaneously, participant_count must not be corrupted. Use `UPDATE ... SET count = count + 1` (atomic) not `SELECT count; UPDATE ... SET count = old_count + 1` (race condition). Check for any read-then-write patterns.

### 2.3 State Machine Validity

Check for:

- [ ] **Every status column has a defined set of valid transitions.** An order can go from `authorized` → `fulfilling` → `captured` → `delivered`, but NOT from `delivered` → `authorized`. Document valid transitions and enforce them in code.
- [ ] **Status transitions are checked server-side before applying.** The API must verify "is this transition valid from the current status?" before updating. A stale client could send a transition request based on an outdated status.
- [ ] **Terminal states cannot be exited.** `delivered`, `cancelled`, `refunded`, `complete` — once an entity reaches a terminal state, no further transitions are allowed.
- [ ] **Status changes are logged.** When a status changes, log the old status, new status, who/what triggered it, and when. This is essential for debugging "how did this order end up in this state?"

### 2.4 Scheduled Job Reliability

Check for:

- [ ] **Scheduled jobs are idempotent.** If the batch reformation job runs twice on the same Monday morning (because of a restart or clock skew), it should produce the same result — not duplicate batches.
- [ ] **Scheduled jobs log their execution.** Start time, end time, what they processed, any errors. Without this, a silently failing job is invisible.
- [ ] **Scheduled jobs have error handling.** If one participant's notification fails, the job should continue processing the remaining participants — not abort the entire run.
- [ ] **Job timing accounts for edge cases.** The batch cutoff auth job runs "1 hour before cutoff." What if the server was down at that time and comes back 30 minutes later? The job should check for any cutoffs that passed within the last 2 hours, not just exactly-on-time cutoffs.
- [ ] **Node-cron persistence.** Node-cron schedules are in-memory. If the ECS task restarts, cron jobs restart from scratch. This is fine as long as jobs are idempotent and check for missed runs on startup.

### 2.5 API Design Consistency

Check for:

- [ ] **Consistent error response format.** Every error from every endpoint should return the same JSON structure: `{ "error": "message", "code": "ERROR_CODE" }`. Inconsistent error formats break frontend error handling.
- [ ] **Consistent success response format.** Naming conventions (camelCase for JSON keys), pagination structure, timestamp formats (ISO 8601).
- [ ] **HTTP status codes are used correctly.** 200 for success, 201 for created, 400 for validation errors, 401 for unauthenticated, 403 for unauthorized, 404 for not found, 409 for conflicts, 500 for server errors. Don't return 200 with an error in the body.
- [ ] **Endpoints don't expose more data than needed.** An endpoint that returns a list of stores doesn't need to include each store's internal configuration. Return what the client needs, nothing more.
- [ ] **Pagination exists for list endpoints that could grow.** Product lists, order history, notification history. Even if there are only 5 items today, pagination prevents a future problem.

---

## Audit Category 3: Completeness & Ambiguity

### 3.1 Spec Completeness

Check for:

- [ ] **Every UI element has a defined behavior.** If the spec shows a button, what happens when you tap it? What happens if you tap it twice quickly? What happens if the network is slow?
- [ ] **Every API endpoint has a complete request/response contract.** Request body with field names, types, and validation rules. Response body with exact JSON structure. Error responses with specific codes.
- [ ] **Every database table has a complete schema.** Column names, types, constraints, indexes, and foreign keys. No "we'll figure out the schema later."
- [ ] **Empty states are defined.** What does the screen look like when there are no orders? No products? No batch routes? An undefined empty state becomes a blank screen or a crash.
- [ ] **Loading states are defined.** What does the user see while data is being fetched? A spinner? A skeleton screen? A blank page? This matters for perceived performance, especially on slow rural connections.
- [ ] **Error states are defined.** What does the user see when the API returns an error? When the network times out? When Stripe declines a card? Every failure mode needs a user-facing response.
- [ ] **Edge cases are addressed.** What if a customer adds 99 of the same item? What if a producer's product name is 200 characters long? What if the driver submits a receipt total of $0? What if someone's address is outside all defined zones? Specs should either handle these or explicitly state they're out of scope.

### 3.2 Ambiguous Language

Flag any instance of:

- **"Handle appropriately"** — How? What does "appropriate" mean in this context? This will be interpreted differently by every AI coding agent.
- **"Should be validated"** — What validation? What are the rules? What's the error message?
- **"Error handling"** without specifics — Which errors? What does the user see? What gets logged?
- **"Secure the endpoint"** — How? Auth check? Rate limit? Input validation? All three?
- **"Display relevant information"** — What information, exactly? In what format?
- **"Standard pagination"** — What page size? What's the max? Cursor-based or offset-based?
- **"Appropriate fee"** — Calculated how? From which table? With what formula?
- **"Notify the user"** — Via SMS? In-app? Both? What's the message text?
- **"Update the status"** — From what status to what status? Is the transition valid?
- **"Clean up"** — Delete? Archive? Mark inactive? When? How often?

Any of these phrases in a spec document is a bug waiting to happen. Flag them and demand specifics.

### 3.3 Cross-Document Consistency

Check for:

- [ ] **Table schemas match across specs.** If C-01 defines the `customers` table and C-04 adds columns to it, do the column names and types match? Are there conflicts?
- [ ] **API endpoints don't conflict.** If C-02 defines `GET /api/home` and C-06 defines `GET /api/community`, do they reference the same underlying data consistently?
- [ ] **Status values are consistent.** If C-04 defines order statuses and C-06 references them, are they using the same values? A status called `checked_out` in one spec and `order_placed` in another is a bug.
- [ ] **Feature flags are referenced consistently.** If C-02 checks `on_demand_enabled` and C-04 also checks it, they should be querying the same flag from the same table.
- [ ] **Fee calculations use the same formula everywhere.** If C-03 displays a fee estimate and C-04 charges based on it, they must use the same calculation. Different rounding or different fee tiers = customer sees one price, gets charged another.

### 3.4 Missing Specs

When auditing, also check whether the document REFERENCES features or behaviors that don't have their own spec yet. Flag these as dependencies that need to be written before implementation.

Common gaps:
- Admin dashboard (viewing orders, managing stores, resolving flags)
- Store/producer dashboard (product management, deal posting, order fulfillment)
- Driver app (route view, shopping checklist, receipt submission, delivery confirmation)
- Notification system (SMS sending, template management, delivery status updates)
- Refund/dispute resolution flow
- Account settings and profile management

---

## Audit Category 4: Maintainability & Debuggability

### 4.1 Logging

Check for:

- [ ] **Every API request is logged.** Method, path, customer_id (from session), response status code, response time in milliseconds. This is the minimum for debugging "what happened at 3pm when Martha's order failed."
- [ ] **Every state transition is logged.** Order status changes, batch status changes, participant status changes. Include before/after status, triggering action, and timestamp.
- [ ] **Every Stripe API call is logged.** PaymentIntent ID, operation (create/capture/refund), amount, success/failure, and Stripe's response code. Never log card numbers or full PaymentMethod details.
- [ ] **Every scheduled job execution is logged.** Job name, start time, end time, items processed, errors encountered.
- [ ] **Error logs include enough context to debug without reproducing.** Not just "Error: something went wrong" but "Error creating PaymentIntent for order 142, customer 37, store 3: Stripe returned card_declined (insufficient_funds)."
- [ ] **Logs are structured (JSON format).** Structured logs can be queried in CloudWatch Logs Insights. Unstructured string logs are nearly useless for debugging patterns.

### 4.2 Error Messages

Check for:

- [ ] **Customer-facing error messages are helpful but not technical.** "Your card was declined" not "Stripe API returned 402 with code card_declined on PaymentIntent pi_xxx."
- [ ] **Internal error messages include the entity ID.** "Order 142 capture failed" not "Capture failed." When your partner reports a problem, she needs to reference an order number.
- [ ] **Validation errors tell the user what to fix.** "Item description is required" not "Validation failed." "Maximum 200 characters" not "String too long."

### 4.3 Configuration Over Code

Check for:

- [ ] **Every business rule that might change is in a config table.** Fee percentages, buffer percentages, thresholds, minimum participants, dormancy weeks, grace periods, cash limits, delivery unlock thresholds, cutoff times. ALL of these should be in `system_config` or a dedicated config table.
- [ ] **Config values have sensible defaults.** If a config row is missing, the code should fall back to a hardcoded default — not crash. Document the defaults.
- [ ] **Config changes don't require a code deploy.** Changing a fee percentage should be: admin dashboard → update value → save. Not: edit code → commit → push → deploy → wait.
- [ ] **Feature flags control new features.** Any feature that might need to be turned off quickly (on-demand ordering, bulk specials, cash payments) should have a feature flag. Flipping a flag is faster than rolling back a deploy.

### 4.4 Database Design for Debugging

Check for:

- [ ] **Every table has `created_at` and `updated_at` columns.** "When was this record created?" and "When was it last changed?" are the first two questions in any debugging session.
- [ ] **Soft deletes where appropriate.** For records that might need to be recovered or audited (orders, customers, products), use an `active` flag instead of `DELETE`. For truly ephemeral data (sessions, rate limits), hard delete is fine.
- [ ] **Audit trail for sensitive operations.** Cash verification, no-contact unlock/revoke, admin flag resolution. Who did it, when, and why.
- [ ] **Indexes exist for common query patterns.** Every WHERE clause used in API endpoints should have a supporting index. Missing indexes = slow queries = timeouts = angry customers.

### 4.5 Code Organization

Check for:

- [ ] **One responsibility per module.** Auth logic in one file. Stripe logic in one file. Fee calculation in one file. Order creation in one file. If Stripe logic is scattered across 10 files, a Stripe API change means editing 10 files.
- [ ] **Database queries are centralized.** All SQL for a given table should be in a single data access file (e.g., `orders.repository.js`). No raw SQL scattered through route handlers.
- [ ] **Shared constants are defined once.** Status values, fee types, error codes — defined in a constants file, not string literals scattered through code. A typo in a string literal is a silent bug.
- [ ] **Environment-specific config is clearly separated.** Database URLs, API keys, feature flags — all from environment variables or Secrets Manager, never hardcoded for a specific environment.

---

## Audit Category 5: Operational Readiness

### 5.1 Deployment

Check for:

- [ ] **The deploy process is documented step by step.** For the project owner, who will be running deploys with Claude's help. "Push to main" → "ECR build" → "ECS service update" — each step explicit.
- [ ] **Rollback is possible.** ECS keeps previous task definitions. Rolling back to the last known good version should be a one-command operation.
- [ ] **Database migrations are safe.** Adding a column is safe. Removing a column can break the running code. Migrations should be additive only until the old code is fully replaced. Document the migration strategy.
- [ ] **No downtime deploys.** ECS rolling updates: new task starts, health check passes, old task stops. The service is never fully down during a deploy.

### 5.2 Monitoring & Alerting

Check for:

- [ ] **Health check endpoint exists.** `GET /health` returns 200 with a simple JSON body. The ALB uses this to determine if the task is healthy. Include a database connection check (try a simple query).
- [ ] **CloudWatch alarms are configured.** Minimum: 5xx error rate > threshold, response time > threshold, ECS task count = 0 (task died and didn't restart).
- [ ] **Stripe webhook monitoring.** If webhooks are configured, monitor for failures. A failing webhook means payment state may be out of sync.

### 5.3 Cost Awareness

Check for:

- [ ] **No surprise cost generators.** Google Maps API calls are metered. Twilio SMS costs per message. Stripe charges per transaction. Every third-party API call should be intentional and accounted for.
- [ ] **Caching exists for expensive operations.** Geocoding results cached in `geocode_cache` (30-day TTL). Don't re-geocode the same address on every request.
- [ ] **No N+1 query patterns.** Loading a list of 10 stores, then making a separate query for each store's batch state = 11 queries. One JOIN or batch query = 1 query. N+1 patterns kill performance and increase RDS costs under load.

### 5.4 Data Recovery

Check for:

- [ ] **Customer data deletion is documented.** GDPR-style data deletion runbook in MASTER.md. Manual SQL, but documented and tested.
- [ ] **Backup restoration is documented.** How to restore the database from an RDS automated backup. Step by step for the project owner to follow with Claude's guidance.
- [ ] **What data is lost if the ECS task dies mid-request?** Any in-memory state is lost. This should be fine — all state should be in the database or Stripe, not in memory. Verify no in-memory caches hold critical data.

---

## Audit Category 6: User Experience Sanity

### 6.1 Slow Connection Handling

The target population is in rural Maine. Internet connections are slow, inconsistent, and sometimes via cellular data only. Check for:

- [ ] **The frontend works on slow connections.** Static assets are on CloudFront (CDN). API responses are small. No multi-megabyte JavaScript bundles.
- [ ] **Buttons are disabled after tap to prevent double-submission.** "Place Order" tapped twice on a slow connection should not create two orders. The button should disable immediately and show a loading state.
- [ ] **Optimistic UI updates where appropriate.** Adding an item to cart should feel instant (update UI immediately, sync to server in background). But payment operations should NOT be optimistic — wait for server confirmation.
- [ ] **Timeout handling.** What happens if an API call takes 30 seconds? The user should see a "This is taking longer than expected. Please wait..." message, not a frozen screen.

### 6.2 Mobile-First Design

Most users will be on phones. Check for:

- [ ] **All touch targets are at least 44x44 pixels.** Small buttons are unusable on mobile, especially for older users.
- [ ] **Text is readable without zooming.** Minimum 16px base font size.
- [ ] **Forms are usable on mobile.** Appropriate input types (tel for phone numbers, email for email, number for quantities). Auto-focus on the first field. Keyboard doesn't obscure the submit button.
- [ ] **The bottom sheet / floating panel doesn't block critical content.** On small screens, a draggable bottom sheet can cover important information.

### 6.3 Accessibility Basics

Check for:

- [ ] **Color is not the only indicator.** Error states should have text labels, not just red color. Status should have text, not just colored dots.
- [ ] **Form fields have labels.** Not just placeholder text — actual labels that screen readers can identify.
- [ ] **Images have alt text.** Product photos, store photos, delivery photos.

---

## Audit Category 7: Cost Reasonableness

The goal is modern industry standard — not bargain-bin and not enterprise overkill. AWS costs should stay under $100/month at launch volume. Third-party API costs should be predictable and proportional to usage.

### 7.1 Infrastructure Cost

Check for:

- [ ] **No over-provisioned resources.** ECS Fargate at 0.25 vCPU / 0.5 GB is correct for this scale. If someone recommends scaling up "just in case," push back — scale when metrics justify it.
- [ ] **No unnecessary services running.** NAT Gateway ($32/month) was eliminated by design. If it reappears in infrastructure config, flag it. Same for any service that adds $20+/month without clear justification.
- [ ] **RDS instance size is appropriate.** db.t3.micro is correct for launch. Monitor and upgrade only when connection count or query latency warrants it.
- [ ] **S3 costs are negligible at this scale.** If S3 costs are projected above $5/month at launch, something is wrong (excessive storage or missing lifecycle rules).
- [ ] **CloudFront is serving static assets.** If the API is serving static files directly, that's wasted CPU and bandwidth.

### 7.2 Third-Party API Cost

Check for:

- [ ] **Twilio SMS costs are estimated per feature.** Every spec that sends SMS should estimate monthly message count and cost. Total Twilio cost should be projected.
- [ ] **Google Maps API calls are minimized.** Geocoding results are cached (30-day TTL). Map tile loads are the biggest cost — estimate monthly loads based on expected active users.
- [ ] **Stripe fees are the biggest variable cost.** At 2.9% + $0.30 per transaction, Stripe takes a significant cut. Verify the revenue model accounts for this. At $1,000 GMV, Stripe takes ~$30-40.
- [ ] **No unexpected API call patterns.** A bug that geocodes the same address on every page load, or sends an SMS on every cart update, can run up costs fast. Check for API calls inside loops or frequent-polling patterns.

### 7.3 Cost vs. Reliability Tradeoffs

Flag any decision where the cheaper option introduces meaningful reliability risk:

- Single-AZ RDS is fine for launch (saves ~$15/month vs. Multi-AZ) — but document the recovery plan.
- Single ECS task is fine for launch — but ECS should auto-restart on failure.
- No Redis/ElastiCache is fine — but note when in-memory caching becomes necessary.
- No separate staging environment is acceptable at launch — but flag when production testing becomes risky.

---

## Audit Category 8: Legal & Compliance

### 8.1 Food Sovereignty Compliance

Check for:

- [ ] **Producer is always the named seller.** The platform facilitates, it does not sell. This distinction must be maintained in all customer-facing text, receipts, and legal copy.
- [ ] **Buying club framing is consistent.** The platform is structured as a buying club / private arrangement under Maine statute. This must be reflected in terms of service, product detail views, and checkout confirmations.
- [ ] **Municipality verification for Track 2 producers.** The system must verify that a food sovereignty producer's municipality has an active food sovereignty ordinance. The `sovereignty_municipalities` table must be kept current.
- [ ] **Consumer acknowledgment.** The customer must acknowledge the direct transaction with the producer. This is a legal requirement, not optional UX copy.

### 8.2 Payment Compliance

Check for:

- [ ] **PCI compliance is handled by Stripe.** The platform never touches card data. Stripe Elements + Stripe.js handle PCI scope. Verify no custom card input fields exist outside of Stripe Elements.
- [ ] **Receipts and transaction records are retained.** For tax and dispute purposes. Order records should never be hard-deleted.
- [ ] **Sales tax handling is defined.** Maine food purchases may have specific tax rules. Define whether the platform collects tax, or whether this is the vendor's responsibility. This needs legal guidance — flag for the project owner to resolve.

### 8.3 Privacy

Check for:

- [ ] **Data collection disclosure.** What data is collected, why, and how it's used. This should be in the terms of service accepted at signup.
- [ ] **Data deletion capability.** The MASTER.md has a deletion runbook. Verify it covers all tables that store PII.
- [ ] **Third-party data sharing.** The platform shares addresses with drivers (necessary for delivery). This should be disclosed. Customer data should never be shared with stores beyond what's necessary for order fulfillment.

---

## Audit Category 9: Graceful Degradation

What works when something is partially broken? The system should degrade gracefully, not fail catastrophically.

### 9.1 Partial Failures

Check for:

- [ ] **Map fails to load → list view still works.** Product browsing, cart, and checkout don't depend on Google Maps.
- [ ] **SMS fails to send → order still processes.** SMS is a notification, not a gate. An order should never fail because Twilio returned an error.
- [ ] **One batch participant's auth fails → other participants are unaffected.** The batch cutoff auth job processes each participant independently.
- [ ] **CloudFront goes down → ?** The frontend is served from CloudFront. If it's down, the site is down. This is an acceptable single point of failure (CloudFront has 99.9% SLA). Document it.
- [ ] **Slow database queries → timeout handling.** API endpoints should have reasonable query timeouts. A slow query should return a 503, not hang indefinitely.

### 9.2 Stale Data Handling

Check for:

- [ ] **What if a product's price changes between cart add and checkout?** The cart should re-validate prices at checkout time, not trust the price from when the item was added.
- [ ] **What if a batch's participant count changes between cart display and order placement?** Fees should be recalculated at order creation, not trusted from the cart display.
- [ ] **What if a store is deactivated while a customer has items in cart?** The checkout should check store status and show a clear error: "This store is no longer available."

---

## Audit Category 10: Testing & Verification

Even though code is AI-generated, it should be testable and tested.

### 10.1 Test Coverage

Check for:

- [ ] **Critical paths have tests.** Order creation, payment auth/capture, fee calculation, batch cutoff job. These handle money and must be verified.
- [ ] **Fee calculation has unit tests with known inputs/outputs.** Given a cart with specific items, the fee should be exactly $X.XX. Rounding errors in fee calculations are real bugs that cost real money.
- [ ] **Status transitions have tests.** Verify that invalid transitions are rejected.
- [ ] **Auth/authorization has tests.** Verify that a customer cannot access another customer's data, that a non-admin cannot access admin endpoints, etc.

### 10.2 Manual Testing Checklist

Before launch, the project owner and partner should manually test:

- [ ] Signup flow end-to-end (phone verify, account creation, address entry)
- [ ] Browse a partnered store catalog, add items, checkout with a test card
- [ ] Build a shopping list for an unpartnered store, submit, verify it appears in the driver view
- [ ] Complete a batch route cycle: express interest → join batch → submit list → cutoff → auth → driver shops → capture → delivery
- [ ] Cash order end-to-end
- [ ] Admin: verify a customer for cash, toggle no-contact, view flagged orders
- [ ] Receive SMS notifications at each step
- [ ] Test on a real phone with cellular data (not just desktop with fast WiFi)

### 10.3 Stripe Test Mode

Check for:

- [ ] **All development and testing uses Stripe test mode keys.** Test keys start with `sk_test_` and `pk_test_`. Production keys start with `sk_live_` and `pk_live_`. These must NEVER be mixed.
- [ ] **Test card numbers are documented.** Stripe provides test cards: `4242424242424242` (success), `4000000000000002` (decline), etc. The project owner needs to know these for manual testing.
- [ ] **The switch to production keys is a documented manual step.** Not automated — the project owner consciously switches from test to live when ready.

---

## "Ask Joe" Escalation Criteria

The project owner has access to Joe, a senior software engineer at Indeed, for architectural guidance. Joe's time is valuable. Use the ASK JOE severity level in audit reports only when the decision meets ALL of these criteria:

### When to Escalate

1. **The decision has significant cost or reliability implications.** Not "should this variable be named X or Y" but "should we use connection pooling, and if so, which library?"
2. **Claude is uncertain between two legitimate approaches.** If the best practice is clear and well-established, just do it — don't ask Joe. If there are genuine tradeoffs that depend on experience and judgment, escalate.
3. **The decision is hard to reverse later.** Database schema choices, authentication architecture, deployment topology — things that are painful to change after launch. Easy-to-change decisions (UI layout, config values, copy text) are never worth escalating.
4. **The decision involves infrastructure Joe has direct experience with.** AWS architecture, container orchestration, database scaling, CI/CD — these are in Joe's wheelhouse and worth his input.

### When NOT to Escalate

- Standard patterns that any Node.js/PostgreSQL project uses (express middleware, parameterized queries, etc.)
- UI/UX decisions (these are product decisions, not engineering decisions)
- Business logic that's already been decided with the project owner
- Things that can be easily changed later (config values, feature flags, copy)
- Questions that a quick Google search or documentation read would answer

### How to Format an ASK JOE Finding

```
## ASK JOE — [Topic]

**Decision needed:** [What specifically needs to be decided]
**Context:** [Brief background — 2-3 sentences max]
**Option A:** [Description + pros/cons]
**Option B:** [Description + pros/cons]
**Claude's leaning:** [Which option and why, acknowledging uncertainty]
**What to ask Joe:** [The specific 1-2 sentence question for Joe]
```

Keep it concise. Joe should be able to read the finding and give an answer in under 2 minutes. If the question takes a paragraph to explain, simplify it or break it into smaller questions.

---

## How to Structure an Audit Report

When performing an audit, structure findings as:

```
# Audit Report: [Document Name]
Date: [date]
Auditor: Claude
Scope: [what was reviewed]

## CRITICAL — Must Fix Before Shipping

### [Finding title]
**Location:** [which section/endpoint/table]
**Issue:** [what's wrong]
**Risk:** [what happens if not fixed]
**Recommendation:** [specific fix]

## WARNING — Should Fix, Risk If You Don't

### [Finding title]
...

## NOTE — Improvement Opportunity

### [Finding title]
...

## ASK JOE — Architecture/Infrastructure Decisions

### [Topic]
(Use the format described in the Ask Joe section above)

## Positive Findings

### [What's done well]
Brief acknowledgment of things that are correctly implemented.
This helps the project owner know which parts they can trust.

## Summary

[X] critical, [Y] warnings, [Z] notes, [W] ask-joe items.
Overall assessment: [Ready / Needs Work / Major Issues]
```

---

## Special Audit Types

### Payment Audit

When specifically asked to audit payment-related code or specs:
- Trace the ENTIRE payment lifecycle: card entry → auth → hold → fulfillment → capture → customer notification
- Verify every Stripe API call has an idempotency key
- Verify no money-related calculation happens on the client
- Verify error handling for every Stripe failure mode (decline, network error, expired auth, partial capture)
- Verify cash flow is completely separated from card flow
- Check that the admin can see and resolve every type of payment flag

### Security Audit

When specifically asked to audit security:
- Attempt to find ways a malicious user could: access another user's data, bypass payment, modify fees, impersonate admin/driver/store, scrape personal information
- Check every API endpoint for auth AND authorization (not just "is logged in" but "is allowed to do this specific action")
- Verify no secrets in code, logs, or client-side bundles
- Check all user input paths for injection risks

### Pre-Implementation Audit

When auditing a spec BEFORE code is written:
- Focus on completeness and ambiguity (Category 3)
- Flag every place where Claude Code would need to make an assumption
- Verify the database schema is complete enough to support every endpoint described
- Check that all referenced tables/endpoints from other specs actually exist
- Verify error states, empty states, and loading states are defined

### Post-Implementation Audit

When auditing code AFTER it's been written:
- Focus on security (Category 1) and reliability (Category 2)
- Check that the code matches the spec — features described but not implemented are bugs
- Check that the code doesn't include features NOT in the spec — unapproved features are risk
- Run through the error handling paths — are they actually implemented or just TODO comments?
- Check that database indexes match the query patterns in the code
- Verify logging is actually present, not just described in the spec

---

## Reminder to the Auditor

You are the last line of defense before this code handles real money for real people in a small community. The project owner is trusting you because they cannot verify the code themselves. Be thorough. Be honest. If something feels wrong but you can't articulate exactly why, flag it anyway with your best explanation. A false alarm costs minutes. A missed bug costs trust, money, and potentially the business.
