# Halal Aura — Premium Halal Fragrance E-Commerce Platform

> A live, UK-based premium halal fragrance store serving UK customers, with international fulfilment built in — built and operated end-to-end as a three-service system (storefront, admin, API).

---

**Author:** Md. Mahamudul Hasan Pavel · GitHub [@mahmud035](https://github.com/mahmud035) · [LinkedIn](https://www.linkedin.com/in/mahmud035/) · [Portfolio](https://mahamudulhasan.vercel.app/)

**Role:** Sole full-stack developer — architecture, delivery, and production operations.

**Status:** Live in production, processing real orders.

### Live surfaces

| Surface    | URL                                           | Notes                                |
| ---------- | --------------------------------------------- | ------------------------------------ |
| Storefront | **https://halalaura.co.uk**                   | Public shopfront                     |
| Admin      | **https://admin.halalaura.co.uk**             | Operations dashboard _(login-gated)_ |
| API health | **https://api.halalaura.co.uk/api/v1/health** | Monitored production endpoint        |

The API exposes a health endpoint that reports process liveness — proof the backend is running and monitorable:

```json
{
  "success": true,
  "message": "Halal Aura API is running",
  "timestamp": "2026-07-02T02:09:26.452Z"
}
```

> **On the source code.** Halal Aura is a commissioned commercial platform; its application source is the client's property and is kept private out of respect for that ownership. This README documents the engineering — the architecture, the hard decisions, and how the system behaved under real production pressure — so the work can be evaluated without exposing client IP.

---

## Highlights at a glance

- **A forced payment-processor migration was contained to a driver swap.** When the original processor was shut down, the storefront stayed fully operational — browsing, cart, wishlist, and accounts never went down; only the payment path was affected. A new live provider was integrated and taking payments **within 4 days**, with the **order schema and business logic untouched**.
- **One payment abstraction absorbed two provider changes** (Stripe → Mollie → Square). Adding a provider is a new _driver_, not a rewrite — provider-specific quirks are quarantined behind a narrow contract.
- **Payment processing is idempotent by design** — engineered to prevent duplicate charges and duplicate confirmation emails under webhook replay and concurrent delivery, before any incident occurred.
- **Accessibility 100 across all 26 storefront routes** (lab Lighthouse), with **SEO 100** and **Best Practices 100** on the primary route types.
- **Feature-driven architecture across three codebases** with a strict per-feature module pattern and compile-time contract enforcement between frontend types and backend responses.
- **Money is handled as integer pence with merchant-favourable rounding**, applied consistently across API, storefront, and admin.

---

## Overview

Halal Aura is a premium halal fragrance e-commerce platform for the UK market, built for both domestic and international fulfilment via Royal Mail. It is a working commercial business with live customers and real orders, not a demo. The entire system — a customer storefront, an operations admin panel, and the API that backs both — was designed, built, deployed, and is operated by a single developer.

---

## System architecture

Three independently deployed services share one API and one database. The two frontends are separate applications with separate hosting, separate auth surfaces, and separate design systems, but they consume the same domain contracts.

```
        ┌──────────────────────────┐        ┌──────────────────────────┐
        │        STOREFRONT        │        │          ADMIN           │
        │   halalaura.co.uk        │        │  admin.halalaura.co.uk   │
        │  React 19 · Vite · TW4   │        │  React 19 · Radix · TW4  │
        │  (Vercel)                │        │  (Vercel, login-gated)   │
        └────────────┬─────────────┘        └────────────┬─────────────┘
                     │  HTTPS · HTTP-only cookies (.halalaura.co.uk)     │
                     └──────────────────┬───────────────────────────────┘
                                        ▼
                        ┌───────────────────────────────┐
                        │              API               │
                        │      api.halalaura.co.uk       │
                        │   Node · Express 5 · TS        │
                        │   Feature modules · Zod         │
                        │   (Railway)                    │
                        └───────────────┬───────────────┘
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
             ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
             │ MongoDB Atlas│   │  Payment     │   │  Resend /    │
             │ (Mongoose 9) │   │  providers   │   │  Cloudinary  │
             └──────────────┘   │ Square·Mollie│   └──────────────┘
                                └──────────────┘
```

**Data flow.** Both frontends talk to a single versioned REST API. All server state on the clients is managed through TanStack Query; the API owns all business logic and database access. Payment providers reach the API asynchronously via webhooks; the storefront confirmation page reconciles final state by polling the API, which in turn queries the live provider — so the customer sees a correct result even when a webhook is delayed.

**1:1 feature mirroring.** The API is organised into feature modules (`auth`, `product`, `category`, `collection`, `cart`, `wishlist`, `order`, `payment`, `contact`, `newsletter`, `upload`, `address-lookup`, `user`, `admin`). Each frontend mirrors the domains it needs one-to-one. API response shapes define the frontend types, so a contract change surfaces as a **TypeScript compile error**, not a runtime surprise.

**Auth model.** Authentication uses **JWT in HTTP-only cookies** (not bearer tokens), scoped to `.halalaura.co.uk` so the cookie is shared across the storefront, admin, and API subdomains. Google OAuth is verified server-side. Guests can shop and check out without an account.

**Hosting split.** Both React apps deploy to Vercel; the API and its worker concerns deploy to Railway; data lives in MongoDB Atlas.

---

## Tech stack

| Layer             | Stack (pinned)                                                                                                                                                                |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **API**           | Node ≥20 · Express **5.2.1** · TypeScript **5.9.3** · Mongoose **9.0.2** · Zod **4.2.1**                                                                                      |
| **Auth**          | `jsonwebtoken` 9 (HTTP-only cookies) · `google-auth-library` 10 · `bcrypt`                                                                                                    |
| **Payments**      | Square **40** (live) · Mollie **4.5** (integrated, feature-flagged off) · Stripe **20** (dependency + enum value retained for historical orders; no active driver registered) |
| **Infra & ops**   | Helmet 8 · `express-rate-limit` 8 · Winston 3.19 (+ daily-rotate) · Railway                                                                                                   |
| **Media & email** | Cloudinary 2.8 + Sharp 0.34 + Multer 2.0 · Resend 6.6                                                                                                                         |
| **Storefront**    | React **19.2** · Vite **7** · Tailwind **4.1** · TanStack Query **5.90** · React Router 7 · React Hook Form 7.69 + Zod 4 · `motion` 12 · React Compiler · Vercel              |
| **Admin**         | React **19.2** · Vite **7** · Tailwind **4.1** · Radix UI primitives · TanStack Table 8.21 · Recharts 3.6 · Vercel                                                            |
| **Data**          | MongoDB Atlas                                                                                                                                                                 |

---

## Engineering deep-dives

### 6.1 Provider-agnostic multi-provider payment system

**Challenge.** Payments are the highest-risk surface of any store: they touch money, they run through third parties you do not control, and a processor relationship can end without warning. Coupling order and checkout logic directly to one provider's SDK is the default — and it is a trap, because the day the provider goes away, the whole checkout goes with it.

**Approach.** Every provider is hidden behind a deliberately narrow contract. Orders persist a `paymentProvider` discriminator and a `paymentProviderId`; a dispatcher resolves the correct driver at runtime — off the customer's choice for a new order, or off the persisted provider for refunds, retries, and webhook routing on existing orders. The contract covers exactly what any hosted-checkout provider must do, and nothing more:

```ts
interface PaymentDriver {
  readonly providerName: PaymentProvider;
  createCheckout(input): Promise<{ checkoutUrl; paymentProviderId }>;
  verifyPayment(id): Promise<{ status; raw }>;
  handleWebhook(input): Promise<{ paymentProviderId; status }>;
  createRetryCheckout(input): Promise<{ checkoutUrl; paymentProviderId }>;
  createRefund(input): Promise<{ refundId; amount; status }>;
}
```

Controllers and the order service call **orchestrators** (`initiatePayment`, `refundOrder`, `verifyAndParseWebhook`, `handlePaymentResult`, `resolveProvider`, `getEnabledProviders`) — never a driver directly. Each driver maps its provider's native statuses to one normalised enum, so the rest of the system never sees a provider-specific status string. Providers are toggled on and off through an enabled-providers config, not a code deploy.

**Impact.** This abstraction absorbed **two** provider changes — Stripe → Mollie → Square — with the **order schema and business logic untouched**. When the original processor was force-closed, the storefront stayed up in full; only the payment step was affected, and a new live provider was taking real payments **within 4 days**. Adding that provider was a driver addition plus a config flag. The historical Stripe orders still resolve correctly through the persisted discriminator, even though Stripe has no active driver.

### 6.2 Two webhook models behind one contract

**Challenge.** Payment providers do not agree on how they tell you a payment succeeded. A push provider sends the full authoritative payload and expects you to verify a signature. A pull provider sends only an identifier and expects you to call back for the truth. Leaking that difference into shared code means every future provider reopens the checkout core.

**Approach.** Both models live entirely inside their driver:

| Provider                | Webhook model | What the driver does                                                                                                                                                                  |
| ----------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Square** (live)       | **Push**      | Verifies an **HMAC-SHA256** signature over the raw body + notification URL, then treats the verified payload's payment object as authoritative.                                       |
| **Mollie** (integrated) | **Pull**      | Payload carries only a payment ID; the driver **re-fetches** the live payment from the provider API — that fetch _is_ the security boundary, since only the server holds the API key. |

The controller receives the raw request bytes and headers, hands them to `verifyAndParseWebhook`, and gets back the same normalised `{ paymentProviderId, status }` tuple regardless of provider. It responds `200` immediately, then applies the result to the order out of band. Because the confirmation redirect can beat the webhook, the storefront also polls a verification endpoint with a **per-provider polling window** (a longer window for Square, whose sandbox settles more slowly) so the customer always lands on a correct final state.

**Impact.** The two most divergent integration styles in payments — signature-verified push and blind-ID pull — coexist without a single `if (provider === …)` branch in the checkout core. Signature verification, ID re-fetching, and status mapping are all quarantined in the driver that owns them.

### 6.3 Idempotent payment processing

**Challenge.** Webhooks are delivered _at least once_. Providers replay them, retry them, and occasionally deliver two in parallel. A naïve success handler double-deducts stock, double-counts analytics, and — worst for a real customer — sends two confirmation emails or double-processes a charge.

**Approach.** The payment-success path is defended at several layers:

- A **fast idempotency guard**: if the order is already `PAID` and analytics are already processed, the handler returns immediately.
- A **MongoDB transaction with bounded retry** (3 attempts, short backoff) for the money-touching work, which re-reads the order _inside_ the transaction and **double-checks the `PAID` status** — so a competing webhook that already settled the order causes a clean abort rather than a second stock deduction.
- Stock is deducted with atomic conditional updates; analytics carry their own `analyticsProcessed` flag so a handler that crashed mid-way can resume without redoing completed work.
- Confirmation email uses a **send-first-then-mark** pattern (`confirmationEmailSent` / `confirmationEmailSentAt`): the email is sent, _then_ the flag is set, and the flag is reset if sending fails. This deliberately favours a rare duplicate email over a missed one — the correct trade-off for a purchase receipt.

**Impact.** The system is engineered to prevent duplicate charges and duplicate confirmation emails under webhook replay and concurrent delivery. This was built as proactive hardening — the correctness guarantees were designed in before any incident, not bolted on after one.

### 6.4 Feature-driven architecture across three codebases

**Challenge.** One developer maintaining three applications cannot afford ambiguity about where code lives or hidden coupling between features. Debugging has to be boring.

**Approach.** The API enforces a strict per-feature module split — a route file wires endpoints and validation only, a controller never touches the database, a service owns all logic and data access, Zod schemas validate input, a Mongoose model defines the schema, and interfaces define the types. Each frontend mirrors the backend domains one-to-one, with a hard `features/` boundary and **no cross-feature imports** — shared logic is promoted to shared UI, hooks, or utils before it can leak. Because frontend types mirror API responses, a contract change breaks the build at `tsc` rather than in front of a customer.

**Impact.** A predictable, uniform shape across 14 API modules and two frontends means any feature can be located, reasoned about, and changed in isolation — the property that makes a solo-owned, multi-service system sustainable in production.

### 6.5 Money & checkout correctness

**Challenge.** Floating-point money and inconsistent discount rules are how e-commerce quietly loses (or gives away) real money.

**Approach.** Prices are stored and computed as **integer pence** everywhere, converted for display only. Discounts round with **`Math.floor`** — merchant-favourable, so the store never gifts fractional pennies. The **free-shipping threshold is evaluated against the pre-discount subtotal**, and that rule is applied identically across the API, the storefront, and the admin — never re-derived per surface. Checkout supports guests, so a purchase never requires account creation, and phone/postcode validation was widened to accept international addresses for cross-border fulfilment. Every input is validated in depth — Zod at the edge, then the service layer.

**Impact.** Monetary calculations are exact, consistent across all three applications, and biased in the business's favour by design, with a checkout that does not shed customers at a mandatory sign-up wall.

### 6.6 Quality & resilience

**Challenge.** A public storefront is judged on accessibility, discoverability, and speed; a production API is judged on whether it stays correct under abuse and stays observable.

**Approach & evidence.**

- **Accessibility & standards (storefront, lab Lighthouse).** Captured live on the three primary route types — home, product detail, and paginated shop:

  | Route type       | Accessibility | Best Practices | SEO | Performance |
  | ---------------- | ------------- | -------------- | --- | ----------- |
  | Home             | 100           | 100            | 100 | 86–91       |
  | Product detail   | 100           | 100            | 100 | 86–91       |
  | Shop (paginated) | 100           | 100            | 100 | 86–91       |

  Accessibility 100 holds across **all 26 storefront routes**. SEO 100 reflects a structured-data / metadata layer built for crawlers and social sharing.

- **Admin (lab Lighthouse).** Performance **96** on login and **88–92** on the data-dense dashboard, Accessibility **100**, Best Practices **96–100**. (SEO is not tracked for a login-gated panel.)

- **Core Web Vitals — desktop field data.** **LCP 1.94s · INP 88ms · CLS 0.04**, all within Google's "Good" thresholds. Mobile **LCP remains an active optimization target**.

- **API resilience.** Rate limits were tightened to production-grade thresholds; a subtle guest-session bug that produced spurious `401`s was fixed with a non-HTTP-only session-signal cookie; lean-query paths were hardened against null-driven crashes. A monitored health endpoint (shown above) reports liveness.

**Impact.** The customer-facing surface meets a top accessibility and SEO bar with desktop performance in Google's "Good" range, and the API is hardened and observable in production — with mobile performance flagged as ongoing work rather than overclaimed.

---

## Selected engineering decisions

| Decision                                             | Rationale                                                                                                                                                           |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Provider-agnostic payment dispatcher**             | Prevents processor lock-in; a forced migration becomes a driver swap, not a rewrite. Proven across two provider changes with zero schema impact.                    |
| **Wrap, don't delete (migration pattern)**           | Legacy support is retained without an active driver (Stripe dependency + enum value kept) so historical orders keep resolving and nothing is destructively removed. |
| **Store money as integer pence**                     | Eliminates floating-point drift; one representation across all three apps.                                                                                          |
| **`Math.floor` on discounts**                        | Merchant-favourable rounding; never gifts fractional pennies.                                                                                                       |
| **Free-shipping threshold on pre-discount subtotal** | Single, consistent rule applied identically across API, storefront, and admin.                                                                                      |
| **Frontend types mirror API responses**              | Contract drift fails at compile time, not at runtime in front of a customer.                                                                                        |
| **HTTP-only, domain-scoped auth cookies**            | Shared session across subdomains; not exposed to JavaScript.                                                                                                        |
| **Defence-in-depth validation (Zod → service)**      | Every input validated at more than one layer.                                                                                                                       |
| **Send-email-then-mark idempotency**                 | Favours a rare duplicate receipt over a missed one — the correct trade-off for purchase confirmations.                                                              |

---

## Scope of ownership

One developer owned the platform end to end: system architecture, the backend API, both frontend applications (storefront and admin), the payment integrations and migrations, deployment across Vercel and Railway, and live production operations. There was no team to absorb a bad abstraction or a fragile deploy — every decision in this document had to hold up in production, alone.

---

## Appendix A — Resume-ready bullets

- Architected and shipped a **provider-agnostic payment layer** that absorbed **two processor migrations (Stripe → Mollie → Square)** with **zero order-schema or business-logic changes**, containing a forced processor shutdown to a **4-day** driver swap while the storefront stayed fully operational.
- Designed a single payment-driver contract that hides **two incompatible webhook models** — Square's HMAC-SHA256-verified push and Mollie's ID-only pull-and-refetch — behind one normalised interface, with no provider branching in the checkout core.
- Engineered **idempotent payment processing** using a retrying MongoDB transaction, an in-transaction `PAID` double-check, and a send-then-mark email pattern to prevent duplicate charges and duplicate confirmation emails under webhook replay and concurrent delivery.
- Built a **feature-driven architecture across three codebases** (an Express 5 / Mongoose 9 API and two React 19 frontends) with a strict per-feature module split and **compile-time contract enforcement** between frontend types and API responses.
- Achieved **Accessibility 100 across all 26 storefront routes** with **SEO 100 and Best Practices 100** on primary route types (lab Lighthouse); desktop Core Web Vitals in Google's "Good" range (**LCP 1.94s, INP 88ms, CLS 0.04**).
- Implemented **exact money handling in integer pence** with merchant-favourable `Math.floor` rounding and a pre-discount free-shipping rule applied consistently across API, storefront, and admin.
- Hardened the production API — **tightened rate limits**, resolved a guest-session `401` bug, and hardened lean-query paths against null crashes — behind an **HTTP-only, domain-scoped cookie** auth model with server-verified Google OAuth and guest checkout.
- Delivered and now operate the platform solo — **architecture, backend, two frontends, payments, deployment (Vercel + Railway), and live operations** — for a store processing real orders, with international fulfilment built in.

---

## Appendix B — LinkedIn / portfolio blurbs

**Headline (2–3 sentences).**
Sole full-stack developer of Halal Aura, a live UK premium halal fragrance platform — a three-service system (storefront, admin, API) I architected, built, and operate end to end. Its provider-agnostic payment layer absorbed two processor migrations with zero schema changes and contained a forced processor shutdown to a 4-day driver swap.

**About / portfolio (4–6 sentences).**
Halal Aura is a production e-commerce platform I designed, built, and run single-handedly: a customer storefront, an operations admin panel, and the API that backs both, deployed across Vercel and Railway on a MongoDB Atlas backend. The engineering centre of gravity is a provider-agnostic payment system — a narrow driver contract that hides each processor's quirks (HMAC-verified push webhooks for one, ID-only pull-and-refetch for another) and let me migrate providers twice, and survive a forced shutdown, without touching the order schema. Payment processing is idempotent by design, engineered up front to prevent duplicate charges and confirmation emails under webhook replay. The system is built feature-first across all three codebases, with frontend types mirroring API responses so contract changes fail at compile time. The storefront scores Accessibility 100 across all 26 routes with SEO 100 and desktop Core Web Vitals in Google's "Good" range, while money is handled exactly in integer pence with merchant-favourable rounding.
