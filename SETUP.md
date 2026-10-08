# H2K TRAVEL — Booking Platform Setup

**Status today:** the website (design, search, packages, AI assistant, accounts UI, checkout, payment UI) is built. Every external provider is implemented **behind an interface and waiting for credentials**. Nothing is faked — no live availability, no PNRs, no payment success.

- Razorpay integration — implemented, waiting for production credentials
- Stripe integration — implemented, waiting for production credentials
- Email (Resend) — implemented, waiting for `RESEND_API_KEY`
- Flight provider — implemented, waiting for provider credentials
- Hotel provider — implemented, waiting for provider credentials
- Database & auth (Supabase) — implemented, waiting for project keys

---

## The one rule that matters

```
REAL DATA → SERVER VERIFICATION → PAYMENT VERIFICATION → PROVIDER CONFIRMATION → BOOKING CONFIRMED
```

Never skip a step. A booking is only `CONFIRMED` when the airline/hotel/package provider confirms it — **not** when payment succeeds, and never because the browser said so.

Secrets live in server environment variables only. If a key can be read in the browser's Network tab, it is compromised.

---

## 1. Supabase (database + auth)

1. Create a project at supabase.com → copy Project URL, `anon` key, `service_role` key.
2. Put them in `.env.local` (see `.env.example`). `service_role` is **server-only**.
3. Create the tables: `users`, `travellers`, `destinations`, `packages`, `package_dates`, `bookings`, `booking_items`, `booking_travellers`, `flights`, `flight_bookings`, `hotels`, `hotel_rooms`, `hotel_bookings`, `payments`, `refunds`, `cancellations`, `notifications`, `email_logs`, `audit_logs`, `provider_transactions`.
4. **Enable Row Level Security on every customer-facing table.** Policy shape: `auth.uid() = user_id`. Without this, changing `/my-trips/H2K-123` in the URL exposes another customer's booking.
5. Enable Auth → Email + password, and switch on email confirmation.

`bookings.booking_type` is one of: `PACKAGE`, `FLIGHT_ONLY`, `HOTEL_ONLY`, `PACKAGE_FLIGHT`, `PACKAGE_HOTEL`, `PACKAGE_FLIGHT_HOTEL`.

`booking_status`: `DRAFT`, `PENDING`, `PAYMENT_PENDING`, `PAYMENT_RECEIVED`, `PROCESSING`, `AVAILABILITY_CHECKING`, `CONFIRMED`, `PARTIALLY_CONFIRMED`, `BOOKING_REQUIRES_REVIEW`, `FAILED`, `CANCELLED`, `REFUND_PENDING`, `REFUNDED`, `COMPLETED`.

`payment_status`: `PENDING`, `AUTHORIZED`, `PAID`, `FAILED`, `CANCELLED`, `REFUND_PENDING`, `PARTIALLY_REFUNDED`, `REFUNDED`, `VERIFICATION_FAILED`.

---

## 2. Razorpay (India payments)

1. Sign up at razorpay.com and complete business KYC — this takes days and cannot be skipped.
2. Test mode first: copy Key ID + Key Secret from the dashboard.
3. Implement server-side: `createRazorpayOrder`, `verifyRazorpayPayment`, `handleRazorpayWebhook`, `refundRazorpayPayment`, `getRazorpayPaymentStatus`.
4. Verify the payment signature server-side with `RAZORPAY_KEY_SECRET`. Only then set `payment_status = PAID`.
5. Add the webhook in the dashboard → `https://YOURDOMAIN.com/api/webhooks/razorpay`, events: `payment.captured`, `payment.failed`, `refund.processed`.
6. Store the Razorpay event id and mark it processed — **webhooks must be idempotent** (the same event will arrive twice).

Never collect card, CVV, UPI PIN or netbanking credentials in your own form. Use Razorpay Checkout.

## 3. Stripe (international payments)

Same shape: complete business verification, then implement `createStripeCheckout`, `verifyStripePayment`, `handleStripeWebhook`, `refundStripePayment`, `getStripePaymentStatus`. Webhook → `https://YOURDOMAIN.com/api/webhooks/stripe`, verified with `STRIPE_WEBHOOK_SECRET`. Use Stripe Checkout or the Payment Element — never raw card fields.

**Gateway choice is a server decision**, based on currency and configuration, not on a country field the customer typed.

---

## 4. Resend (email)

1. Create a Resend account, verify your sending domain (SPF + DKIM), get `RESEND_API_KEY`.
2. Implement: `sendBookingConfirmation`, `sendPaymentConfirmation`, `sendFlightConfirmation`, `sendHotelConfirmation`, `sendCancellationEmail`, `sendRefundEmail`, `sendAdminBookingNotification`.
3. Customer subject: `H2K TRAVEL — Booking Confirmed | H2K-XXXXXX`. Admin copy goes to `h2ktravel89@gmail.com` with no payment-card data.
4. Log sends in `email_logs` and retry failures. **If email fails, the booking stays confirmed** — never cancel a booking because an email bounced.
5. Only send an email for a state that actually happened.

---

## 5. Flight provider (Amadeus or Duffel)

1. Register for developer access, start with sandbox credentials.
2. Implement `flightService`: `searchFlights`, `getFlightDetails`, `recheckPrice`, `createBooking`, `cancelBooking`.
3. **Price recheck before payment is mandatory** — availability, fare, taxes, baggage, fare rules. If the price moved, stop checkout and show the customer old vs new before charging.
4. Display only provider values: flight numbers, times, baggage, and the real PNR. Never generate a PNR.
5. Collect passport/DOB/gender **only** when the provider requires it for that itinerary.

## 6. Hotel provider (Amadeus or Hotelbeds)

1. Register, start in sandbox.
2. Implement `hotelService`: `searchHotels`, `getHotelDetails`, `getRooms`, `recheckPrice`, `createBooking`, `cancelBooking`.
3. Recheck room availability, price, taxes, meal plan, guest capacity and cancellation rules before payment.
4. Show only provider-supplied ratings, amenities and the real confirmation number. Never invent one.
5. Do not scrape Booking.com, Agoda, Expedia or hotel sites — use the licensed API only.

---

## 7. Server responsibilities (never move these to the browser)

Create payment · verify payment · create booking · check availability · recheck price · compute the final amount · validate coupons · confirm with provider · handle refunds · send email · process webhooks · authorise every booking read.

The frontend is never the source of truth for price, discount, status or refund amount.

Additional protections: rate limiting, input validation, secure cookies, session expiry, CSRF where applicable, audit logging, idempotency keys on pay/booking calls, and booking locks to stop a double-click creating two orders.

---

## 8. Webhook URLs

| Provider | URL |
|---|---|
| Razorpay | `https://YOURDOMAIN.com/api/webhooks/razorpay` |
| Stripe | `https://YOURDOMAIN.com/api/webhooks/stripe` |
| Flights | `https://YOURDOMAIN.com/api/webhooks/flights` |
| Hotels | `https://YOURDOMAIN.com/api/webhooks/hotels` |

Verify the signature, check the event id against `provider_transactions`, and return success without reprocessing if it's a duplicate.

---

## 9. Edge cases to build deliberately

- **Payment succeeded, provider booking failed** → `PAYMENT_RECEIVED` + `BOOKING_REQUIRES_REVIEW`, notify H2K, never show "Confirmed". Staff reconcile or refund.
- **Price changed before payment** → halt, show old vs new, let the customer accept.
- **Room/flight gone** → clear message plus "Search other options" and "Contact H2K TRAVEL".
- **Payment failed** → no booking confirmed; don't claim "no money was charged" unless the gateway confirms it.
- **Booking expired** → release any held inventory.
- **Session expired** → say so and let the customer restart.

---

## 10. Going live

1. Test the whole flow in sandbox: package, flight-only, hotel-only, and the combined types.
2. Test duplicate pay clicks, duplicate webhooks, price change, provider failure, cancellation and refund.
3. Test on mobile — checkout, date picker, traveller selector, payment.
4. Publish Terms, Privacy, Cancellation and Refund policies (get them reviewed; the placeholders in the site are not legal advice).
5. Set `H2K_TEST_MODE=false`, swap sandbox keys for production keys in your host's environment settings, point webhooks at the live domain, serve over HTTPS.
6. Never commit real secrets. Rotate any key that has ever been pasted into chat, a repo, or a frontend file.


---

## Live availability — provider status and endpoints

Switch providers with one constant each; the UI does not change.

| Module | Constant / env | Today | Live options |
|---|---|---|---|
| Flights | `FLIGHT_PROVIDER` | `none` — no live data | `amadeus`, `duffel`, `travelport`, `sabre` |
| Hotels | `HOTEL_PROVIDER` | `mock` — labelled DEMO DATA | `hotelbeds`, `amadeus`, `expedia` |
| Payments | `paymentService` | not connected | Razorpay (India), Stripe (international) |
| Email | `emailService` | not connected | Resend / SendGrid |

Both services expose the same three canonical methods — `search()`, `revalidate()`, `book()` — so the booking flow is provider-independent.

### Server routes (the browser never calls a provider)

```
POST /api/flights/search        → Amadeus  GET  /v2/shopping/flight-offers
                                  Duffel   POST /air/offer_requests
POST /api/flights/revalidate    → mandatory before payment
POST /api/flights/book          → only after verified payment

POST /api/hotels/search         → Hotelbeds POST /hotel-api/1.0/hotels
                                  Amadeus   GET  /v3/shopping/hotel-offers
POST /api/hotels/revalidate     → mandatory before payment
POST /api/hotel-bookings/create → only after verified payment
```

### Normalised shapes

Flight offer: `{ airline, flightNumber, departureAirport, arrivalAirport, departureTime, arrivalTime, duration, stops, cabinClass, baggage, price, currency, fareRules, offerId }`

Hotel: `{ hotelName, location, starRating, rating, images, amenities, rooms: [{ roomType, occupancy, price, currency, cancellationPolicy, mealPlan }] }`

Normalising in the server route is what makes swapping suppliers cheap — the UI reads one shape regardless of provider.

### Booking state machine

`SEARCHED → SELECTED → PRICE_VERIFIED → PAYMENT_PENDING → PAYMENT_SUCCESS → BOOKING_PENDING → BOOKING_CONFIRMED | BOOKING_FAILED`

**Payment success ≠ booking confirmed.** Confirm with the provider after verified payment; if the provider fails, hold at `PAYMENT_SUCCESS` + `BOOKING_REQUIRES_REVIEW`, notify H2K, and never display "Confirmed".

### Where to sign up

- Flights — [developers.amadeus.com](https://developers.amadeus.com) or [duffel.com](https://duffel.com)
- Hotels — Hotelbeds, or Amadeus Hotels (same Amadeus account)
- Payments — razorpay.com, stripe.com
- Email — resend.com

Start on sandbox credentials with `H2K_TEST_MODE=true`, then swap to production keys in your host's environment settings. Without provider keys the site is UI plus clearly-labelled demo data — that is the honest state, not a bug.


---

## Modes, capabilities and error codes

### One switch controls data honesty

`H2K_CONFIG.travelDataMode` (server: `TRAVEL_DATA_MODE`)

- `demo` — clearly-labelled demo data is permitted. Every result carries a visible **DEMO DATA — NOT LIVE** banner.
- `live` — only real provider data may be displayed. **A missing credential never falls back to demo data** — the UI shows "Live availability is not configured." instead.

### Search access ≠ booking access

Each service carries two independent flags, because providers grant them separately: a sandbox may return live fares months before it permits issuing tickets.

```
flightService.searchEnabled  / flightService.bookingEnabled
hotelService.searchEnabled   / hotelService.bookingEnabled
```

Never infer one from the other. Booking UI stays disabled until `bookingEnabled` is true, even with live search working.

### Provider health

`GET /api/health/providers` returns provider name, `configured`, `searchEnabled`, `bookingEnabled` and status per module — never credentials. The account area surfaces this as a **Travel Inventory Status** panel with a LIVE/DEMO badge.

### Standardised error codes

`LIVE_PROVIDER_UNAVAILABLE`, `PROVIDER_NOT_CONFIGURED`, `NO_FLIGHTS_FOUND`, `NO_HOTELS_FOUND`, `OFFER_EXPIRED`, `PRICE_CHANGED`, `ROOM_UNAVAILABLE`, `PAYMENT_FAILED`, `BOOKING_FAILED`, `PROVIDER_TIMEOUT`, `PROVIDER_ERROR`, `INVALID_SEARCH`, `AUTHENTICATION_REQUIRED` — each maps to customer-facing copy in `ERR`.

### Flight offer expiry

Normalised offers carry `expiresAt`. Re-verify before payment; if expired, show "Your selected flight offer has expired. Please search again." and never charge an expired offer.

### Central config

Brand, owner, phone, email and Instagram live in `H2K_CONFIG` — one place, not repeated per component.

### Requires a commercial agreement (not just an API key)

Flight ticketing (IATA/consolidator arrangement), hotel reservation rights, Razorpay/Stripe merchant KYC, and hotel content licensing for images. Live *search* is usually available on sandbox terms much sooner.

### Recommended build order

One flight provider → one hotel provider → Supabase → Razorpay + Stripe → Resend → My Trips/Admin. Do not integrate five providers at once.


---

## SMS OTP + email verification

### The flow

**New customer:** mobile → SMS OTP → mobile verified → full name + email → *separate* email OTP → email verified → account created → welcome email → signed in.

**Existing customer:** mobile → SMS OTP → signed in. No password, no email, no registration form.

Mobile OTP verifies the phone; email OTP verifies the email. They are two independent verifications and never share a code.

### Endpoints (all server-side)

```
POST /api/auth/send-mobile-otp    POST /api/auth/verify-mobile-otp
POST /api/auth/send-email-otp     POST /api/auth/verify-email-otp
POST /api/auth/login              POST /api/auth/register
```

`verify-mobile-otp` returns `{ verified, customerExists }` and nothing else. No response, log or email ever contains the code.

### Provider setup (India)

1. Create an **MSG91** or **2Factor** account.
2. Register a **sender ID** (6 alphanumeric characters) and get it approved.
3. Create and register a **DLT template** — Indian regulation requires the message body be pre-approved. Example: *"H2K Travel verification code is ###. Valid for 5 minutes. Do not share this code with anyone."*
4. Copy the API/auth key, sender ID and template ID into your server environment (never the repo).
5. For email codes and the welcome mail, set `RESEND_API_KEY`, `EMAIL_FROM` and `EMAIL_REPLY_TO`.

Swapping provider means implementing one interface — `sendOtp(phoneE164, otp)` — and changing `SMS_PROVIDER`. Nothing else in auth changes.

### Security rules the server must enforce

- Codes generated with a **cryptographically secure RNG** (never `Math.random`), stored **hashed**, single-use, invalidated on resend.
- **5-minute expiry**, **5 attempts** max, **30s resend cooldown**; 3 requests / 10 min and 10 / hour per number, plus IP limiting — every OTP costs money.
- Phone numbers normalised to **E.164** (`+919358514323`) before any lookup, with a database uniqueness constraint so one person can't create two accounts.
- Account created only when **both** mobile and email are verified.
- A provider send failure must **not** mark the OTP as sent.
- Session via secure, HttpOnly, SameSite cookies over HTTPS.

### Testing locally

Set `OTP_PROVIDER_MODE=development` to print codes to the **server console only** — never a response body, never enabled in production. Then switch to `production` with real credentials.


---

## OTP backend — Twilio Verify (current implementation)

The site's auth screen no longer manages codes. It relays to the Express backend, and **Twilio Verify** owns generation, delivery, retries and expiry.

### Wiring

`API_BASE` in the page is `""` — same origin, because Express serves the frontend too, so there is no CORS step. If you host the page separately, set `API_BASE` to the backend URL and add that origin to `ALLOWED_ORIGINS`.

```
POST /send-otp    { phone, channel }  → triggers an SMS (or email) code
POST /verify-otp  { phone, code }     → { status: "approved" }
```

Numbers are converted to **E.164** before every call: a 10-digit Indian number becomes `+91XXXXXXXXXX`; anything typed with a `+` passes through.

### Run it

```
npm install
cp .env.example .env     # add TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN,
                         # TWILIO_VERIFY_SERVICE_SID
npm start                # http://localhost:3000
```

Create the **Verify Service** in the Twilio console first — the SID it gives you is `TWILIO_VERIFY_SERVICE_SID`.

### Still to add server-side

1. **`POST /lookup-customer` { phone } → { exists }** — the page calls this to decide sign-in vs registration. Until it exists, every verified number is treated as new and goes through name + email.
2. **Email codes** — enable the email channel on the Verify Service (or send them with Resend) so the new-customer email step can verify.
3. **Session + customer record** — persist the verified customer and issue a secure HttpOnly cookie; the page currently just flips to signed-in.
4. **Welcome email** — after both verifications, via Resend.

### India note

Twilio Verify delivers to Indian numbers, but Indian regulation (**DLT**) requires a registered sender and template for reliable delivery — expect to complete Twilio's India onboarding, or use MSG91/2Factor, which are built around it. The `smsService` abstraction means switching is a small change, not a rewrite.

### What the page will not do

No code is generated, stored or validated in the browser. Rate limiting lives on the server (5 sends / 10 verifies per 15 min per IP). A send failure shows an error and does not advance the step, so a failed SMS can never look like a delivered one.


---

## Deployment — one app, H2K homepage as the landing page

The travel site is the landing page; the Express app that holds the Twilio
Verify routes serves it. There is no separate login page — the OTP screen
lives inside the homepage, so `login.html` can be deleted.

### Folder layout

```
server.js
package.json
.env                     TWILIO_* , RAZORPAY_* , RESEND_API_KEY …
public/
  index.html             ← copy of "H2K Travel Home.dc.html"
  support.js
  image-slot.js
  h2k-logo.jpg
```

Rename the design file to `index.html` when you copy it, or add an explicit
route. All four files must sit together — the page loads `support.js`,
`image-slot.js` and the logo relatively.

### server.js

```js
app.use(express.static("public"));          // serves the homepage + assets
app.post("/send-otp", …);                   // Twilio Verify
app.post("/verify-otp", …);
app.post("/lookup-customer", …);            // still to add — see below
app.get("*", (req, res) =>                  // any unknown path → homepage
  res.sendFile(path.join(__dirname, "public/index.html")));
```

Keep the static middleware **above** the catch-all, or assets get swallowed.

### Why no CORS

Page and API share an origin, so `API_BASE` stays `""` and `ALLOWED_ORIGINS`
is only needed if you later host the frontend elsewhere.

### Run

```
npm install
npm start          # http://localhost:3000 → H2K Travel homepage
```

Sign In opens the OTP screen in place. No page navigation, no second auth
implementation to keep in sync.

### Remaining server work

- `POST /lookup-customer { phone } → { exists }` — tells sign-in from registration.
- Customer record + secure HttpOnly session cookie after verification.
- Welcome email via Resend once mobile and email are both verified.
- Razorpay order creation and **server-side** signature verification before any booking is marked paid.

---

## Support details used across the site

H2K TRAVEL · Shubham Kumar · +91-9358514323 · h2ktravel89@gmail.com · [@h2ktravel](https://www.instagram.com/h2ktravel/)

*Go Further. Feel More.*
