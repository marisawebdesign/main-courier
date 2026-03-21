# C-01: First-Time Onboarding & Signup

> **Spec Version:** 1.1
> **Status:** Complete — Ready for Development
> **Dependencies:** Google Maps API, Twilio Verify, S3, PostgreSQL with PostGIS
> **Relates to:** B-02 (Authentication), B-07 (Food Sovereignty Compliance), B-11 (Security)
> **Security:** All endpoints in this spec must comply with the 16 Global Security Standards defined in MASTER.md. Key standards for this flow: #2 (input validation), #3 (phone normalization), #10 (rate limiting), #14 (error handling).

---

## Overview

This spec covers the complete flow from a new visitor arriving on the homepage to landing on their authenticated customer home screen. The flow has 5 screens and should take under 3 minutes to complete. The design priority is zero friction — show value before asking for anything, collect only what's essential, and get the customer to their first order as fast as possible.

The target demographic is rural Maine residents, many of whom are older, on slower internet connections, and less comfortable with app-based signup flows. Every screen should be simple, fast-loading, and forgiving of mistakes.

---

## Flow Summary

```
Screen 1: Landing Page
  │ Customer enters address (or uses geolocation)
  │ No authentication required
  ▼
Screen 2: Service Area Preview
  │ Map view + list view of stores, producers, hubs
  │ ~50 seconds of browsing
  │ If no service area → Waitlist capture → END
  │ Tap "Sign up"
  ▼
Screen 3: Phone Verification
  │ Enter phone number → receive SMS code → enter code
  │ If returning user (phone exists in DB) → Home Screen → END
  ▼
Screen 4: Account Setup (new users only)
  │ Full name, full address, optional profile picture
  │ Address validated against service area
  │ Terms & buying club acceptance
  │ Create account
  ▼
Screen 5: Tutorial Prompt → Customer Home Screen
  │ "Want a quick tour?" → Yes (overlay tutorial) or Skip
  │ Customer is authenticated and ready to order
```

---

## Screen 1: Landing Page

### Purpose
Single entry point for all new visitors. Establishes what the app is and provides one clear action: enter your address to see what's available.

### UI Requirements
- Headline: app name and a single tagline explaining the service (e.g., "Groceries and local food, delivered.")
- One address input field — a standard text input, not a multi-field form. Google Places Autocomplete is optional but not required for MVP.
- A "Use my location" button that triggers the browser Geolocation API
- Minimal page weight — this is the first page users see on potentially slow connections. Target under 200KB total page weight including assets. No heavy images, no animations, no video.
- No navigation, no login link, no footer links. One screen, one action.
- If a referral code is present in the URL query string (`?ref=XXXXXX`), store it in React client state. Do not display a referral input field. Do not acknowledge the referral code visually — it's handled silently during account creation.

### Entry Points
Visitors arrive from four sources. The page handles all identically:
1. **Direct SMS link** from partner or group order notification
2. **Facebook share** from existing customer or community post
3. **Word of mouth** — typed URL into browser
4. **Referral link** — URL includes `?ref=XXXXXX` query parameter

### User Actions
1. **Types address and submits:** Frontend sends the raw address string to `POST /api/address/check`. Waits for response. On success, navigates to Screen 2 with the response data.
2. **Taps "Use my location":** Frontend calls `navigator.geolocation.getCurrentPosition()`. On success, sends lat/lng to `POST /api/address/check`. On failure (user denies permission or geolocation unavailable), show a gentle fallback message: "We couldn't get your location. Please type your address instead." Do not block the flow.

### Error States
- **Geolocation denied or unavailable:** Fall back to manual address entry. No error modal — just the text prompt.
- **Address check API failure (network error, timeout):** Show "Something went wrong. Please try again." with a retry button. Do not navigate away from the page.
- **Address check returns no zone match:** Navigate to Screen 2 anyway — Screen 2 handles the "no service area" state with the waitlist flow.

---

## Screen 2: Service Area Preview

### Purpose
Show the customer what's available in their area before asking them to sign up. This is the "aha moment" — they should understand the value in about 50 seconds and decide to sign up. This is not a full shopping experience. No prices, no delivery schedules, no fee breakdowns. Just: what's near you, it's real, there's stuff here.

### UI Requirements — Service Available State

**Map View (default):**
- A map centered on the customer's entered location
- Customer's location shown as a distinct pin or marker
- Store locations as pins (distinct icon/color from producers and hubs)
- Producer locations as pins (distinct icon/color)
- Hub locations as pins (distinct icon/color)
- A radius outline or shaded area showing the service zone boundary
- Map should be interactive (pan, zoom) but not overly featured — a simple Google Maps embed or Mapbox GL JS instance
- Tapping a pin shows a small popup with: name, town, one-line description

**List View (toggle):**
- Toggle switch or tab to switch between map and list
- Simple scrollable list of all locations grouped by type:
  - **Stores:** name, town, distance from customer (e.g., "Walmart Supercenter · Houlton · 35 mi"), one-line description
  - **Local Producers:** name, town, distance, one-line description
  - **Pickup Hubs:** name, distance (e.g., "Topsfield General Store · 1.2 mi from you")
- Distances calculated from the customer's entered address using the lat/lng returned by the address check API

**Sign Up Button:**
- Fixed at the bottom of the screen (sticky footer)
- Clear call to action: "Sign up to start ordering" or equivalent
- Tapping navigates to Screen 3

### UI Requirements — No Service Area State

If the address check API returns `zone: null` (no matching zone), replace the entire preview with a waitlist capture:

- Message: "We're not in your area yet — but we're growing."
- Optionally show how many other people in their area are on the waitlist (if you have the data): "X people near you have signed up to be notified."
- Phone number input field
- "Notify me when available" button
- On submit: `POST /api/waitlist` with phone, raw address, lat/lng
- After submission: confirmation message "We'll text you when we expand to your area. Thanks!" No further navigation — this is a dead end by design.

### Data Available to This Screen
The `POST /api/address/check` response from Screen 1 provides everything this screen needs:

```json
{
  "zone": {
    "id": "zone_houlton_north",
    "name": "Houlton North Corridor"
  },
  "customer_location": {
    "lat": 45.6912,
    "lng": -67.8834,
    "formatted_address": "42 Route 1, Weston, ME 04424"
  },
  "stores": [
    {
      "id": 1,
      "name": "Walmart Supercenter",
      "town": "Houlton",
      "description": "Groceries, household, pharmacy",
      "lat": 46.1281,
      "lng": -67.8401,
      "distance_miles": 35.2
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
      "distance_miles": 3.4
    }
  ],
  "hubs": [
    {
      "id": 1,
      "name": "Topsfield General Store",
      "lat": 45.4132,
      "lng": -67.7234,
      "distance_miles": 1.2
    }
  ]
}
```

Or if no zone match:
```json
{
  "zone": null,
  "customer_location": {
    "lat": 44.8012,
    "lng": -68.1234,
    "formatted_address": "123 Main St, Somewhere, ME 04400"
  },
  "stores": [],
  "producers": [],
  "hubs": []
}
```

---

## Screen 3: Phone Verification

### Purpose
Authenticate the customer's identity using their phone number and a one-time SMS code. This screen also serves as the login screen for returning users — the flow is identical, with the backend determining whether the user is new or returning after code verification.

### UI Requirements

**Step 3a — Phone Number Entry:**
- Phone number input field with US format hint: `(207) ___-____`
- Country code defaulted to +1, not editable (US only for MVP)
- "Send me a code" button
- Input should accept various formats and normalize on submit: `2075550123`, `(207) 555-0123`, `207-555-0123` all resolve to the same number
- On submit: call `POST /api/auth/send-code` with normalized phone number
- While waiting for API response: disable the button, show a loading indicator
- On success: navigate to Step 3b
- On rate limit error (429): show "Too many attempts. Please wait a few minutes and try again."
- On other error: show "Something went wrong. Please try again."

**Step 3b — Code Entry:**
- Display which phone number the code was sent to: "Enter the code we texted to (207) 555-0123"
- Six individual digit input fields in a row
- Auto-focus the first field on page load
- Auto-advance: when a digit is entered, focus moves to the next field automatically
- Auto-submit: when the 6th digit is entered, immediately call `POST /api/auth/verify-code` — do not require the user to tap a submit button
- Paste support: if the user pastes a 6-digit string, distribute the digits across all fields and auto-submit
- While verifying: show a loading indicator across the input fields
- On success (existing user): set session cookie, navigate to the customer home screen — skip Screens 4 and 5 entirely
- On success (new user): store the temporary token in client state, navigate to Screen 4
- On invalid code: show "That code didn't match. Please try again." Clear the fields and re-focus the first field. Do not navigate away.
- On expired code: show "That code has expired. We'll send you a new one." Automatically trigger a resend.

**Fallback Options (below the code input):**
- "Didn't get it?" label with two links:
  - "Resend code" — calls `POST /api/auth/send-code` again. Show "New code sent!" confirmation. Rate limited to 3 resends per session.
  - "Call me instead" — calls `POST /api/auth/send-code` with `channel: "call"` parameter. Twilio Verify calls the phone and reads the code aloud. Show "Calling you now..." confirmation.

### Returning User Detection
The `POST /api/auth/verify-code` response includes an `isNewUser` boolean:
- `isNewUser: false` → The response includes a `Set-Cookie` header with the session token. The frontend detects this, navigates directly to the customer home screen. The customer sees none of Screens 4 or 5.
- `isNewUser: true` → The response includes a `tempToken` in the JSON body (not a cookie). The frontend stores this in React state and uses it to authorize the account creation request in Screen 4.

---

## Screen 4: Account Setup

### Purpose
Collect the remaining required information to create the customer account: full name, full address (validated against the service area), optional profile picture, and terms acceptance.

### UI Requirements

**Form Fields:**
- First name (required, text input, max 50 characters)
- Last name (required, text input, max 50 characters)
- Street address (required, text input, max 200 characters)
- Town/City (required, text input, max 100 characters)
- State (required, dropdown, defaulted to "Maine" — include all US states but default to ME)
- ZIP code (required, text input, 5-digit US zip, validated format on the frontend)
- Profile picture (optional) — a file upload button or camera capture button. On selection, show a square crop preview. Frontend crops/resizes the image to a maximum of 400x400 pixels and compresses to JPEG at 80% quality before upload. This keeps the file under 100KB, which matters on slow connections.

**Terms Acceptance:**
- A single checkbox (required): "I agree to the Terms of Service and Buying Club Membership"
- "Terms of Service" and "Buying Club Membership" are each links — tapping either opens the respective document in a new browser tab or a modal. These are static HTML pages hosted on S3.
- The checkbox must be checked before the "Create my account" button is enabled. If the user taps the button without checking, highlight the checkbox with an error state.

**Submit Button:**
- "Create my account" button
- Disabled until: first name, last name, all address fields, and terms checkbox are filled/checked
- On tap: call `POST /api/auth/create-account` with all fields plus the temporary token from Screen 3

**Address Validation:**
- The backend re-geocodes the address and validates it falls within a served zone
- If the address is outside all zones: return a 400 error. The frontend shows an inline error below the address fields: "This address is outside our current delivery area. Please check your address or contact us for help." The user can edit and resubmit. Do not navigate away.
- If the address is within a zone: account creation proceeds normally

**Profile Picture Upload:**
- The account creation API response includes a pre-signed S3 upload URL if a profile picture was indicated
- Frontend uploads the cropped/compressed image directly to S3 using the pre-signed URL after the account is created — this is an asynchronous follow-up, not a blocking step. If the upload fails, the account still exists — the profile picture can be added later in settings.
- Flow: create account (without picture) → receive session + pre-signed URL → upload picture to S3 → PATCH customer record with the S3 key. The user sees Screen 5 immediately after account creation — they don't wait for the upload.

**Error States:**
- **Network error on submit:** "Something went wrong. Please try again." Retry button.
- **Temporary token expired (15 minutes since verification):** "Your verification has expired. Please verify your phone number again." Navigate back to Screen 3.
- **Phone number already exists (race condition — someone else created an account with this number between verify and create):** "An account with this phone number already exists. Please sign in." Navigate to Screen 3.
- **Address outside service area:** Inline error as described above. Do not navigate away.

---

## Screen 5: Tutorial Prompt & Home Screen

### Purpose
Transition the newly created customer to their authenticated home screen and optionally walk them through key features.

### UI Requirements

**Tutorial Prompt:**
- Appears as a modal or overlay immediately after account creation, before they interact with the home screen
- Simple message: "Want a quick tour of how the app works?"
- Two buttons: "Show me around" and "Skip for now"
- No aggressive upsell, no guilt trip for skipping

**If they accept the tutorial:**
- A lightweight tooltip-style walkthrough overlay on the home screen
- 4-5 steps maximum, each highlighting a key area:
  1. "This is where you browse stores and local producers"
  2. "Add items to your cart from any source"
  3. "Your delivery day and cutoff time are shown here"
  4. "Check your order status and history here"
- Each step has a "Next" button and a "Skip tour" option
- Tutorial completion state stored on the customer record (`tutorial_completed: true`) so it never shows again

**If they skip:**
- Close the modal, they're on the home screen
- Store `tutorial_completed: true` on the customer record so the prompt never shows again
- Contextual first-time tooltips appear once per section when they first interact with that area (separate from the full tutorial — these appear regardless of tutorial choice, once each, then never again)

---

## Backend Specification

### API Endpoints

---

#### `POST /api/address/check`

**Authentication:** None (public endpoint)

**Rate Limiting:** 10 requests per IP address per minute. Enforced by WAF rate-based rule and/or application-level check in PostgreSQL.

**Purpose:** Geocode an address or coordinates and return available stores, producers, and hubs in the matching zone.

**Request Body:**
```json
{
  "address": "42 Route 1, Weston, ME 04424"
}
```
OR:
```json
{
  "lat": 45.6912,
  "lng": -67.8834
}
```
One of `address` or `lat`/`lng` is required. If both are provided, `address` takes precedence and is geocoded.

**Processing:**
1. If `address` is provided:
   a. Normalize the address string (trim whitespace, collapse multiple spaces)
   b. Check `geocode_cache` table for a matching entry (key: SHA256 hash of lowercased, trimmed address string). If found and `cached_at` is within 30 days, use cached lat/lng and skip the Google API call.
   c. If not cached: call Google Maps Geocoding API with the address string. Extract lat, lng, and formatted address from the response.
   d. Store the result in `geocode_cache`: address_hash, lat, lng, formatted_address, cached_at.
2. Using lat/lng (from geocode or from request), query for matching zones:
   ```sql
   SELECT id, name, center_lat, center_lng, radius_miles
   FROM zones
   WHERE active = true
   AND ST_DWithin(
     ST_SetSRID(ST_MakePoint(center_lng, center_lat), 4326)::geography,
     ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography,
     radius_miles * 1609.34
   )
   ORDER BY ST_Distance(
     ST_SetSRID(ST_MakePoint(center_lng, center_lat), 4326)::geography,
     ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography
   )
   LIMIT 1;
   ```
   ($1 = customer lng, $2 = customer lat. Note: PostGIS ST_MakePoint takes longitude first, then latitude.)
3. If a zone is found, query stores, producers, and hubs within that zone:
   ```sql
   SELECT id, name, description, town, lat, lng,
     ST_Distance(
       ST_SetSRID(ST_MakePoint(lng, lat), 4326)::geography,
       ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography
     ) / 1609.34 AS distance_miles
   FROM stores
   WHERE zone_id = $3 AND active = true
   ORDER BY distance_miles;
   ```
   (Same pattern for `producers` and `hubs` tables.)

**Response (zone found):**
```json
{
  "zone": {
    "id": 1,
    "name": "Houlton North Corridor"
  },
  "customer_location": {
    "lat": 45.6912,
    "lng": -67.8834,
    "formatted_address": "42 Route 1, Weston, ME 04424"
  },
  "stores": [
    {
      "id": 1,
      "name": "Walmart Supercenter",
      "town": "Houlton",
      "description": "Groceries, household, pharmacy",
      "lat": 46.1281,
      "lng": -67.8401,
      "distance_miles": 35.2
    }
  ],
  "producers": [],
  "hubs": []
}
```

**Response (no zone match):**
```json
{
  "zone": null,
  "customer_location": {
    "lat": 44.8012,
    "lng": -68.1234,
    "formatted_address": "123 Main St, Somewhere, ME 04400"
  },
  "stores": [],
  "producers": [],
  "hubs": []
}
```

**Error Responses:**
- `400` — Missing both `address` and `lat`/`lng`
- `400` — Google Geocoding returned no results (invalid address)
- `429` — Rate limited
- `500` — Google API failure or database error

---

#### `POST /api/waitlist`

**Authentication:** None (public endpoint)

**Rate Limiting:** 3 submissions per phone number per 24 hours. Checked against `waitlist` table.

**Purpose:** Capture contact info for customers outside the current service area.

**Request Body:**
```json
{
  "phone": "2075550123",
  "address": "123 Main St, Somewhere, ME 04400",
  "lat": 44.8012,
  "lng": -68.1234
}
```

**Processing:**
1. Normalize phone number to E.164 format: `+12075550123`
2. Validate phone format (must be 10-digit US number after normalization)
3. Check rate limit: count existing waitlist entries with this phone in the last 24 hours
4. Compute nearest zone (even though they're outside all zones, store the closest one for analytics):
   ```sql
   SELECT id,
     ST_Distance(
       ST_SetSRID(ST_MakePoint(center_lng, center_lat), 4326)::geography,
       ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography
     ) / 1609.34 AS distance_miles
   FROM zones WHERE active = true
   ORDER BY distance_miles LIMIT 1;
   ```
5. Insert into waitlist table

**Response:**
```json
{
  "success": true,
  "message": "You're on the list. We'll text you when we expand to your area."
}
```

**Error Responses:**
- `400` — Missing or invalid phone number
- `429` — Already submitted recently

---

#### `POST /api/auth/send-code`

**Authentication:** None (public endpoint)

**Rate Limiting:** 3 attempts per phone number per 10-minute window. Tracked in `rate_limits` table.

**Purpose:** Send a 6-digit verification code to the provided phone number via SMS (or voice call).

**Request Body:**
```json
{
  "phone": "2075550123",
  "channel": "sms"
}
```
`channel` is optional, defaults to `"sms"`. Accepted values: `"sms"`, `"call"`.

**Processing:**
1. Normalize phone number to E.164 format: `+12075550123`
2. Validate phone format
3. Check rate limit in `rate_limits` table:
   ```sql
   SELECT COUNT(*) FROM rate_limits
   WHERE identifier = $1
   AND action = 'send_code'
   AND created_at > NOW() - INTERVAL '10 minutes';
   ```
   If count >= 3, return 429.
4. Record this attempt:
   ```sql
   INSERT INTO rate_limits (identifier, action, created_at)
   VALUES ($1, 'send_code', NOW());
   ```
5. Call Twilio Verify API:
   ```javascript
   await twilioClient.verify.v2
     .services(TWILIO_VERIFY_SERVICE_SID)
     .verifications.create({
       to: normalizedPhone,
       channel: channel // 'sms' or 'call'
     });
   ```
6. Return success. **Do not reveal whether the phone number exists in the database.** The response is identical whether the number belongs to an existing customer or a new one.

**Response:**
```json
{
  "sent": true,
  "channel": "sms"
}
```

**Error Responses:**
- `400` — Missing or invalid phone number
- `400` — Invalid channel value
- `429` — Rate limited: `{ "error": "Too many attempts. Please wait a few minutes.", "code": "RATE_LIMITED" }`
- `502` — Twilio API failure

---

#### `POST /api/auth/verify-code`

**Authentication:** None (public endpoint)

**Rate Limiting:** 5 incorrect attempts per phone number per verification. After 5 failures, the pending verification is invalidated and a new code must be requested.

**Purpose:** Verify the 6-digit code and either authenticate an existing customer or issue a temporary token for new account creation.

**Request Body:**
```json
{
  "phone": "2075550123",
  "code": "847293"
}
```

**Processing:**
1. Normalize phone number to E.164 format
2. Check verification attempt rate limit:
   ```sql
   SELECT COUNT(*) FROM rate_limits
   WHERE identifier = $1
   AND action = 'verify_code'
   AND created_at > NOW() - INTERVAL '10 minutes';
   ```
   If count >= 5, return 429 with instruction to request a new code.
3. Call Twilio Verify API to check the code:
   ```javascript
   const check = await twilioClient.verify.v2
     .services(TWILIO_VERIFY_SERVICE_SID)
     .verificationChecks.create({
       to: normalizedPhone,
       code: code
     });
   ```
4. If `check.status !== 'approved'`:
   a. Record the failed attempt in `rate_limits`
   b. Return 401
5. If approved, check if customer exists:
   ```sql
   SELECT id, first_name, last_name, zone_id FROM customers
   WHERE phone = $1;
   ```
6. **Existing customer path:**
   a. Generate session token: `crypto.randomBytes(32).toString('hex')`
   b. Insert into sessions table:
      ```sql
      INSERT INTO sessions (token, customer_id, expires_at)
      VALUES ($1, $2, NOW() + INTERVAL '90 days');
      ```
   c. Set HTTP-only secure cookie with the session token
   d. Return `{ isNewUser: false, customer: { id, first_name, last_name, zone_id } }`
7. **New customer path:**
   a. Generate temporary token: `crypto.randomBytes(32).toString('hex')`
   b. Insert into pending_verifications table:
      ```sql
      INSERT INTO pending_verifications (phone, token, expires_at)
      VALUES ($1, $2, NOW() + INTERVAL '15 minutes');
      ```
   c. Return `{ isNewUser: true, tempToken: "<token>" }`
   d. **Do not set a session cookie.** The temporary token is returned in the response body only.

**Response (existing user):**
```json
{
  "isNewUser": false,
  "customer": {
    "id": 42,
    "firstName": "Martha",
    "lastName": "Williams",
    "zoneId": 1
  }
}
```
Plus `Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax; Max-Age=7776000; Path=/`

**Response (new user):**
```json
{
  "isNewUser": true,
  "tempToken": "a8f2c9e14b7d..."
}
```

**Error Responses:**
- `400` — Missing phone or code
- `401` — Invalid code: `{ "error": "Invalid code. Please try again.", "code": "INVALID_CODE" }`
- `429` — Too many failed attempts: `{ "error": "Too many incorrect attempts. Please request a new code.", "code": "VERIFY_RATE_LIMITED" }`
- `502` — Twilio API failure

---

#### `POST /api/auth/create-account`

**Authentication:** Temporary token (from verify-code response) required in the `Authorization` header: `Authorization: Bearer <tempToken>`

**Purpose:** Create a new customer account with full profile information.

**Request Body:**
```json
{
  "firstName": "Martha",
  "lastName": "Williams",
  "addressStreet": "42 Route 1",
  "addressTown": "Weston",
  "addressState": "ME",
  "addressZip": "04424",
  "hasProfilePicture": true,
  "termsAccepted": true,
  "referralCode": "XXXXXX"
}
```
`hasProfilePicture` is a boolean indicating whether the client intends to upload a profile picture after account creation. `referralCode` is optional — passed through from the URL query string captured on Screen 1.

**Input Validation (all server-side, do not trust client validation):**
- `firstName`: required, string, 1-50 characters, trimmed, stripped of HTML tags
- `lastName`: required, string, 1-50 characters, trimmed, stripped of HTML tags
- `addressStreet`: required, string, 1-200 characters, trimmed
- `addressTown`: required, string, 1-100 characters, trimmed
- `addressState`: required, string, must be a valid 2-letter US state abbreviation
- `addressZip`: required, string, must match regex `^\d{5}$`
- `termsAccepted`: required, must be `true`. If `false` or missing, return 400.
- `referralCode`: optional, string, 6-8 alphanumeric characters if present

**Processing:**
1. Validate the temporary token:
   ```sql
   SELECT phone FROM pending_verifications
   WHERE token = $1
   AND expires_at > NOW()
   AND used = false;
   ```
   If no match: return 401 (expired or invalid token).
2. Validate all input fields (see validation rules above). Return 400 with specific field errors for any failures.
3. Check that no customer exists with this phone number (race condition guard):
   ```sql
   SELECT id FROM customers WHERE phone = $1;
   ```
   If found: mark the pending verification as used, return 409 Conflict.
4. Geocode the address via Google Maps API (or geocode cache):
   a. Construct address string: `"{addressStreet}, {addressTown}, {addressState} {addressZip}"`
   b. Check `geocode_cache` for a match. If miss, call Google Geocoding API.
   c. Extract lat, lng, formatted_address.
   d. Cache the result.
5. Validate the geocoded coordinates fall within an active zone (same PostGIS query as address/check endpoint). If no zone match: return 400 with error code `OUTSIDE_SERVICE_AREA`.
6. Calculate cluster assignment within the zone:
   ```sql
   SELECT id FROM clusters
   WHERE zone_id = $1
   ORDER BY ST_Distance(
     ST_SetSRID(ST_MakePoint(center_lng, center_lat), 4326)::geography,
     ST_SetSRID(ST_MakePoint($2, $3), 4326)::geography
   )
   LIMIT 1;
   ```
   (Cluster assignment is nullable — if no clusters are defined for the zone, leave it null.)
7. Generate a unique referral code for this customer: 6 uppercase alphanumeric characters. Generate, check uniqueness against `customers.referral_code_own`, retry if collision (unlikely with 36^6 = 2.1 billion possibilities).
8. If `referralCode` was provided, validate it:
   ```sql
   SELECT id FROM customers WHERE referral_code_own = $1;
   ```
   If valid, store the referrer's customer ID. If invalid, ignore silently (don't fail account creation over a bad referral code).
9. **In a single database transaction:**
   a. Insert the customer record
   b. Mark the pending verification as used: `UPDATE pending_verifications SET used = true WHERE token = $1`
   c. Create a session token and insert into sessions table
10. Set the session cookie on the response.
11. If `hasProfilePicture` is true, generate a pre-signed S3 PUT URL:
    ```javascript
    const uploadUrl = await s3.getSignedUrlPromise('putObject', {
      Bucket: PROFILE_BUCKET,
      Key: `profiles/${customerId}.jpg`,
      ContentType: 'image/jpeg',
      Expires: 300 // 5 minutes
    });
    ```
    Include this URL in the response.

**Response:**
```json
{
  "customer": {
    "id": 42,
    "phone": "+12075550123",
    "firstName": "Martha",
    "lastName": "Williams",
    "addressStreet": "42 Route 1",
    "addressTown": "Weston",
    "addressState": "ME",
    "addressZip": "04424",
    "lat": 45.6912,
    "lng": -67.8834,
    "zoneId": 1,
    "zoneName": "Houlton North Corridor",
    "clusterId": 3,
    "referralCodeOwn": "WES7MK",
    "tutorialCompleted": false,
    "createdAt": "2026-03-20T14:32:00Z"
  },
  "profileUploadUrl": "https://s3.amazonaws.com/..."
}
```
Plus `Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax; Max-Age=7776000; Path=/`

**Error Responses:**
- `400` — Validation errors: `{ "error": "Validation failed", "code": "VALIDATION_ERROR", "fields": { "firstName": "Required", "addressZip": "Must be 5 digits" } }`
- `400` — Address outside service area: `{ "error": "This address is outside our current delivery area.", "code": "OUTSIDE_SERVICE_AREA" }`
- `401` — Temp token invalid or expired: `{ "error": "Verification expired. Please verify your phone number again.", "code": "TOKEN_EXPIRED" }`
- `409` — Account already exists for this phone: `{ "error": "An account with this phone number already exists.", "code": "ACCOUNT_EXISTS" }`
- `500` — Database or Google API failure

---

#### `PATCH /api/customers/me`

**Authentication:** Session cookie required

**Purpose:** Update customer profile fields after account creation. Used for: profile picture key update after S3 upload, tutorial completion flag, and future profile edits.

**Request Body (partial — only include fields being updated):**
```json
{
  "profilePictureKey": "profiles/42.jpg",
  "tutorialCompleted": true
}
```

**Processing:**
1. Validate session token from cookie via auth middleware (see Authentication Middleware below)
2. Validate provided fields against allowed update fields. Reject any attempt to update phone, zone_id, or buying_club fields through this endpoint.
3. Update the customer record:
   ```sql
   UPDATE customers
   SET profile_picture_key = COALESCE($1, profile_picture_key),
       tutorial_completed = COALESCE($2, tutorial_completed),
       updated_at = NOW()
   WHERE id = $3
   RETURNING *;
   ```

**Response:** Updated customer object.

**Error Responses:**
- `401` — Not authenticated
- `400` — Invalid fields

---

### Authentication Middleware

Every endpoint except the four public ones (`address/check`, `waitlist`, `auth/send-code`, `auth/verify-code`) must pass through authentication middleware.

```javascript
async function requireAuth(req, res, next) {
  const token = req.cookies.session;

  if (!token) {
    return res.status(401).json({
      error: 'Authentication required.',
      code: 'NOT_AUTHENTICATED'
    });
  }

  const result = await db.query(
    `SELECT s.customer_id, s.expires_at, c.first_name, c.last_name, c.zone_id
     FROM sessions s
     JOIN customers c ON c.id = s.customer_id
     WHERE s.token = $1
     AND s.expires_at > NOW()`,
    [token]
  );

  if (!result.rows.length) {
    // Clear the invalid/expired cookie
    res.clearCookie('session');
    return res.status(401).json({
      error: 'Session expired. Please sign in again.',
      code: 'SESSION_EXPIRED'
    });
  }

  // Attach customer info to the request for downstream handlers
  req.customerId = result.rows[0].customer_id;
  req.customerName = result.rows[0].first_name;
  req.zoneId = result.rows[0].zone_id;

  // Optionally update last_accessed_at asynchronously (non-blocking)
  db.query(
    'UPDATE sessions SET last_accessed_at = NOW() WHERE token = $1',
    [token]
  ).catch(() => {}); // Fire and forget — don't block the request

  next();
}
```

---

### Request Logging Middleware

Applied to all requests. Produces one JSON log line per request that CloudWatch can index and search. See MASTER.md Developer Notes → "Logging: What to Log and What Never to Log" for the full policy.

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

Register this middleware **before** your route handlers so it captures all requests. Register the global error handler **after** all route handlers so it catches unhandled errors.

Applied to all responses. See Global Security Standards in MASTER.md for the full rationale behind each header.

```javascript
function securityHeaders(req, res, next) {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  res.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=(self)');
  // NOTE: Content-Security-Policy is intentionally omitted at MVP.
  // CSP requires whitelisting every third-party domain and breaks silently
  // when misconfigured. Add it once all third-party integrations are finalized.
  // See Global Security Standards #5 in MASTER.md for details.
  next();
}
```

---

### CORS Configuration

```javascript
const corsOptions = {
  // IMPORTANT: Read from Secrets Manager, not hardcoded.
  // Production: 'https://app.maincourier.com'
  // Development: 'http://localhost:3000'
  origin: secrets.FRONTEND_ORIGIN,
  credentials: true, // Required for session cookies to be sent cross-origin
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400 // Cache preflight response for 24 hours
};
```

**Do not use wildcard (`*`) for origin.** The CORS origin must be the exact CloudFront domain in production. The value is configured once per environment in Secrets Manager and never needs to change unless you change your domain.

---

### Global Error Handler

This must be the **last middleware registered** in your Express/Fastify app. It catches any unhandled error from any endpoint and returns a safe generic message to the client while logging full details to CloudWatch. This is Security Standard #14 from MASTER.md.

```javascript
app.use((err, req, res, next) => {
  // Log the full error to CloudWatch — this is what you search when debugging
  console.error(JSON.stringify({
    timestamp: new Date().toISOString(),
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
    customerId: req.customerId || 'unauthenticated',
    // Redact sensitive fields from the request body
    body: req.body ? {
      ...req.body,
      code: req.body.code ? '[REDACTED]' : undefined,
      token: req.body.token ? '[REDACTED]' : undefined
    } : undefined
  }));

  // Return a generic message to the client — never expose internals
  if (!res.headersSent) {
    const status = err.status || err.statusCode || 500;
    res.status(status).json({
      error: err.clientMessage || 'Something went wrong. Please try again.',
      code: err.code || 'INTERNAL_ERROR'
    });
  }
});
```

**Usage in endpoint handlers:** When you want to return a specific user-facing error (like "Invalid code" or "Address outside service area"), throw an error with `clientMessage` and `code` properties set. When an unexpected error occurs (database down, Twilio timeout), let it bubble up to this handler unmodified — the customer sees the generic message and you see the full details in CloudWatch.

---

### Database Schema (Tables for This Flow)

```sql
-- Enable PostGIS extension (run once on database setup)
CREATE EXTENSION IF NOT EXISTS postgis;

-- ============================================================
-- CUSTOMERS
-- ============================================================
CREATE TABLE customers (
  id                        SERIAL PRIMARY KEY,
  phone                     VARCHAR(15) UNIQUE NOT NULL, -- E.164 format: +12075550123
  first_name                VARCHAR(50) NOT NULL,
  last_name                 VARCHAR(50) NOT NULL,
  address_street            VARCHAR(200) NOT NULL,
  address_town              VARCHAR(100) NOT NULL,
  address_state             VARCHAR(2) NOT NULL DEFAULT 'ME',
  address_zip               VARCHAR(5) NOT NULL,
  lat                       DECIMAL(9,6) NOT NULL,
  lng                       DECIMAL(9,6) NOT NULL,
  zone_id                   INTEGER REFERENCES zones(id),
  cluster_id                INTEGER REFERENCES clusters(id),
  default_delivery_type     VARCHAR(10), -- 'door' or 'hub', set later during first order
  default_hub_id            INTEGER REFERENCES hubs(id),
  profile_picture_key       VARCHAR(255),
  buying_club_accepted      BOOLEAN NOT NULL DEFAULT false,
  buying_club_accepted_at   TIMESTAMPTZ,
  buying_club_terms_version VARCHAR(10),
  referral_code_used        VARCHAR(10),
  referral_code_own         VARCHAR(10) UNIQUE NOT NULL,
  referred_by_customer_id   INTEGER REFERENCES customers(id),
  tutorial_completed        BOOLEAN NOT NULL DEFAULT false,
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_phone ON customers(phone);
CREATE INDEX idx_customers_zone ON customers(zone_id);
CREATE INDEX idx_customers_cluster ON customers(cluster_id);
CREATE INDEX idx_customers_referral_code ON customers(referral_code_own);

-- ============================================================
-- SESSIONS
-- ============================================================
CREATE TABLE sessions (
  id                SERIAL PRIMARY KEY,
  token             VARCHAR(64) UNIQUE NOT NULL, -- crypto.randomBytes(32).toString('hex')
  customer_id       INTEGER NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at        TIMESTAMPTZ NOT NULL,
  last_accessed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_token ON sessions(token);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
CREATE INDEX idx_sessions_customer ON sessions(customer_id);

-- ============================================================
-- PENDING VERIFICATIONS
-- ============================================================
CREATE TABLE pending_verifications (
  id          SERIAL PRIMARY KEY,
  phone       VARCHAR(15) NOT NULL, -- E.164 format
  token       VARCHAR(64) UNIQUE NOT NULL,
  used        BOOLEAN NOT NULL DEFAULT false,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMPTZ NOT NULL -- 15 minutes from creation
);

CREATE INDEX idx_pending_token ON pending_verifications(token);
CREATE INDEX idx_pending_phone ON pending_verifications(phone);
CREATE INDEX idx_pending_expires ON pending_verifications(expires_at);

-- ============================================================
-- RATE LIMITS
-- ============================================================
CREATE TABLE rate_limits (
  id          SERIAL PRIMARY KEY,
  identifier  VARCHAR(50) NOT NULL, -- phone number or IP address
  action      VARCHAR(30) NOT NULL, -- 'send_code', 'verify_code', 'address_check', 'waitlist'
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_limits_lookup ON rate_limits(identifier, action, created_at);

-- ============================================================
-- WAITLIST
-- ============================================================
CREATE TABLE waitlist (
  id              SERIAL PRIMARY KEY,
  phone           VARCHAR(15) NOT NULL, -- E.164 format
  address_raw     VARCHAR(300),
  lat             DECIMAL(9,6),
  lng             DECIMAL(9,6),
  nearest_zone_id INTEGER REFERENCES zones(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_waitlist_phone ON waitlist(phone);

-- ============================================================
-- GEOCODE CACHE
-- ============================================================
CREATE TABLE geocode_cache (
  address_hash    VARCHAR(64) PRIMARY KEY, -- SHA256 of normalized address string
  lat             DECIMAL(9,6) NOT NULL,
  lng             DECIMAL(9,6) NOT NULL,
  formatted       VARCHAR(300),
  cached_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- ZONES (referenced by this flow, defined in detail in A-02)
-- ============================================================
CREATE TABLE zones (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(100) NOT NULL,
  center_lat      DECIMAL(9,6) NOT NULL,
  center_lng      DECIMAL(9,6) NOT NULL,
  radius_miles    DECIMAL(6,2) NOT NULL,
  boundary_geojson JSONB, -- optional complex boundary shape
  active          BOOLEAN NOT NULL DEFAULT true,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- CLUSTERS (referenced by this flow, defined in detail in A-02)
-- ============================================================
CREATE TABLE clusters (
  id          SERIAL PRIMARY KEY,
  zone_id     INTEGER NOT NULL REFERENCES zones(id),
  name        VARCHAR(100),
  center_lat  DECIMAL(9,6) NOT NULL,
  center_lng  DECIMAL(9,6) NOT NULL,
  radius_miles DECIMAL(6,2),
  active      BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clusters_zone ON clusters(zone_id);

-- ============================================================
-- STORES (referenced by this flow, defined in detail in S-01)
-- ============================================================
CREATE TABLE stores (
  id          SERIAL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL,
  description VARCHAR(300),
  address     VARCHAR(200),
  town        VARCHAR(100),
  lat         DECIMAL(9,6) NOT NULL,
  lng         DECIMAL(9,6) NOT NULL,
  zone_id     INTEGER NOT NULL REFERENCES zones(id),
  store_type  VARCHAR(50),
  active      BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stores_zone ON stores(zone_id);

-- ============================================================
-- PRODUCERS (referenced by this flow, defined in detail in A-04)
-- ============================================================
CREATE TABLE producers (
  id                      SERIAL PRIMARY KEY,
  name                    VARCHAR(100) NOT NULL,
  description             VARCHAR(300),
  town                    VARCHAR(100),
  lat                     DECIMAL(9,6) NOT NULL,
  lng                     DECIMAL(9,6) NOT NULL,
  zone_id                 INTEGER NOT NULL REFERENCES zones(id),
  sovereignty_track       VARCHAR(20) NOT NULL, -- 'track1_licensed' or 'track2_sovereignty'
  municipality_verified   BOOLEAN NOT NULL DEFAULT false,
  active                  BOOLEAN NOT NULL DEFAULT true,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_producers_zone ON producers(zone_id);

-- ============================================================
-- HUBS (referenced by this flow, defined in detail in A-02)
-- ============================================================
CREATE TABLE hubs (
  id                  SERIAL PRIMARY KEY,
  name                VARCHAR(100) NOT NULL,
  address             VARCHAR(200),
  lat                 DECIMAL(9,6) NOT NULL,
  lng                 DECIMAL(9,6) NOT NULL,
  zone_id             INTEGER NOT NULL REFERENCES zones(id),
  max_orders_per_day  INTEGER,
  has_refrigeration   BOOLEAN NOT NULL DEFAULT false,
  contact_name        VARCHAR(100),
  contact_phone       VARCHAR(15),
  active              BOOLEAN NOT NULL DEFAULT true,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_hubs_zone ON hubs(zone_id);

-- ============================================================
-- AUTO-UPDATE updated_at TRIGGER
-- ============================================================
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customers_updated_at
  BEFORE UPDATE ON customers
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- CLEANUP JOBS (run nightly via ECS scheduled task or in-app cron)
-- ============================================================
-- Delete expired sessions
-- DELETE FROM sessions WHERE expires_at < NOW();

-- Delete expired/used pending verifications
-- DELETE FROM pending_verifications WHERE expires_at < NOW() OR used = true;

-- Delete rate limit records older than 1 hour
-- DELETE FROM rate_limits WHERE created_at < NOW() - INTERVAL '1 hour';

-- Delete geocode cache entries older than 30 days
-- DELETE FROM geocode_cache WHERE cached_at < NOW() - INTERVAL '30 days';
```

---

### Scheduled Cleanup Job

Run nightly (or every 6 hours) as an ECS scheduled task or as a cron job within the API container:

```sql
-- Expired sessions
DELETE FROM sessions WHERE expires_at < NOW();

-- Used or expired pending verifications
DELETE FROM pending_verifications WHERE expires_at < NOW() OR used = true;

-- Old rate limit records (older than 1 hour)
DELETE FROM rate_limits WHERE created_at < NOW() - INTERVAL '1 hour';

-- Stale geocode cache (older than 30 days)
DELETE FROM geocode_cache WHERE cached_at < NOW() - INTERVAL '30 days';
```

Implementation: either schedule an ECS task (Fargate) that runs a Node.js script executing these queries, or use a lightweight cron library (like `node-cron`) inside the API container itself that triggers at a configured interval. The in-container approach is simpler and avoids spinning up a separate Fargate task.

---

### ECS Fargate Configuration Notes for This Flow

**Task Definition:**
- Image: pulled from ECR (`your-account.dkr.ecr.us-east-1.amazonaws.com/your-api:latest`)
- CPU: 256 (0.25 vCPU)
- Memory: 512 (0.5 GB)
- Port mappings: container port 3000 (or whatever your Express/Fastify app listens on)
- Log configuration: awslogs driver → CloudWatch log group `/ecs/your-api`

**Task Role (IAM):**
- `secretsmanager:GetSecretValue` — read Twilio, Stripe, Google Maps, DB connection secrets
- `s3:PutObject` on the profile picture bucket (for generating pre-signed URLs)
- `s3:GetObject` on the profile picture bucket (for generating pre-signed read URLs)

**Execution Role (IAM):**
- `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` — pull image from ECR
- `logs:CreateLogStream`, `logs:PutLogEvents` — write to CloudWatch

**Service Configuration:**
- Desired count: 1
- Launch type: FARGATE
- Platform version: LATEST
- Health check: ALB health check on `/api/health` endpoint (return 200 with `{ "status": "ok" }`)
- Deployment: rolling update, minimum healthy percent 100%, maximum percent 200% (starts new task before stopping old one)

**Networking:**
- Fargate task in a **public subnet** with `assignPublicIp: ENABLED`
- No NAT gateway required
- Security group inbound: port 3000 from ALB security group only
- Security group outbound: port 443 to 0.0.0.0/0 (Twilio, Google Maps, Stripe, Secrets Manager), port 5432 to RDS security group

**ALB Configuration:**
- Internet-facing, in public subnet
- HTTPS listener (443) with ACM certificate, forwards to target group on port 3000
- HTTP listener (80) redirects to HTTPS
- Target group: IP type (for Fargate awsvpc), health check on `/api/health`, healthy threshold 2, interval 30s
- WAF attached with rate-based rule: 100 requests per 5 minutes per IP (adjustable)

**RDS Configuration:**
- db.t3.micro, PostgreSQL 15+
- Private subnet, security group allows inbound 5432 only from Fargate task security group
- Encryption at rest enabled (AWS KMS, no extra cost)
- Automated backups enabled, 7-day retention
- Deletion protection enabled
- Storage: 20 GB gp3 (sufficient for years at this volume)
- PostGIS extension enabled

**Secrets Manager:**
- Secret: `danforth-delivers/api` containing JSON:
  ```json
  {
    "DB_HOST": "your-rds-endpoint",
    "DB_PORT": "5432",
    "DB_NAME": "danforth",
    "DB_USER": "api_user",
    "DB_PASSWORD": "...",
    "TWILIO_ACCOUNT_SID": "...",
    "TWILIO_AUTH_TOKEN": "...",
    "TWILIO_VERIFY_SERVICE_SID": "...",
    "STRIPE_SECRET_KEY": "...",
    "GOOGLE_MAPS_API_KEY": "...",
    "SESSION_SECRET": "...",
    "FRONTEND_ORIGIN": "https://app.maincourier.com",
    "S3_PROFILE_BUCKET": "danforth-delivers-profiles"
  }
  ```
- Read by the container at startup, cached in memory for the lifetime of the task

---

### Phone Number Normalization Utility

Used by every endpoint that receives a phone number:

```javascript
function normalizePhone(raw) {
  // Strip everything except digits
  const digits = raw.replace(/\D/g, '');

  // Handle various input formats
  if (digits.length === 10) {
    return `+1${digits}`;
  }
  if (digits.length === 11 && digits.startsWith('1')) {
    return `+${digits}`;
  }

  throw new ValidationError('Invalid phone number format. Please enter a 10-digit US phone number.');
}
```

All database storage and Twilio API calls use the E.164 normalized format. The frontend can display the number in a human-friendly format `(207) 555-0123` but the backend always works with `+12075550123`.

---

### Third-Party API Cost Summary for This Flow

| Service | Operation | Cost | Monthly Estimate (50 signups + 100 previews) |
|---------|-----------|------|----------------------------------------------|
| Twilio Verify | Send + check code | $0.05 per verification | $2.50 |
| Google Maps Geocoding | Geocode an address | $0.005 per request | ~$0.75 (with caching) |
| S3 | Profile picture storage | $0.023 per GB | < $0.01 |
| S3 | Pre-signed URL generation | Free | $0 |
| **Total** | | | **~$3.25/month** |
