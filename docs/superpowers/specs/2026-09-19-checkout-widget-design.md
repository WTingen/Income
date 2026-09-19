# Embeddable Checkout Widget — Design Spec

Date: 2026-09-19

## Purpose

A small, embeddable checkout widget that any website can drop into their
pages to accept card payments. The platform earns a small per-transaction
fee on top of the normal Stripe processing fee, in exchange for providing
merchants an easier way to accept payments than integrating Stripe
directly.

This is the first sub-project of a larger idea (a business that earns a
small amount across a high volume of transactions). Fraud/error detection
and currency-conversion/routing are potential future sub-projects, each
with their own design and build cycle — they are explicitly out of scope
here.

## Why this is legally sound

The platform never takes custody of funds and never moves money on its
own authority. All money movement happens through Stripe, a licensed
payment processor, using Stripe Connect. Every merchant explicitly opts
in via OAuth, agreeing to the platform fee as part of connecting their
account. This mirrors how Shopify, Squarespace, and other platforms
built on Stripe Connect already operate.

## Non-goals (MVP)

- No real money movement — everything runs in Stripe **test mode**.
- No production merchant onboarding, legal ToS, or live keys.
- No fraud detection or currency conversion (future sub-projects).
- No platform-specific plugins (Shopify/WooCommerce apps) — the widget
  is a plain embeddable script that works on any site.

## Architecture

Monorepo with three packages:

1. **`widget/`** — Vanilla JS embeddable checkout form, built with Vite
   and shipped as a single `<script>` tag. Uses Stripe Elements for card
   input so the platform never handles raw card numbers (PCI scope stays
   minimal).
2. **`server/`** — Node.js + Express API. Responsibilities:
   - Stripe Connect OAuth flow (Express account type) for merchant
     onboarding.
   - Creates PaymentIntents on the merchant's connected account with
     `application_fee_amount` set to the platform's fee.
   - Verifies and processes Stripe webhooks (`payment_intent.succeeded`,
     `payment_intent.payment_failed`, `account.updated`).
   - Stores merchant + transaction records (SQLite for MVP — no need for
     a hosted DB yet).
3. **`dashboard/`** — React app (Vite) where a merchant:
   - Signs up (simple email/password or magic link).
   - Clicks "Connect with Stripe" (OAuth redirect).
   - Once connected, sees their embed snippet (`<script>` tag with their
     merchant ID baked in).
   - Sees a list of test transactions and fees collected.

## Data flow

1. Merchant signs up on the dashboard and connects their Stripe account
   via Connect OAuth. Server stores the returned `stripe_account_id`.
2. Merchant copies their embed snippet into their site's HTML.
3. A customer on the merchant's site triggers checkout; the widget loads
   and renders a Stripe Elements card form.
4. On submit, the widget calls `POST /api/checkout` on the server with
   the merchant ID and amount.
5. The server creates a PaymentIntent **on the merchant's connected
   account**, setting `application_fee_amount` to the platform's cut,
   and returns the `client_secret` to the widget.
6. The widget confirms the payment with Stripe.js using the
   `client_secret`.
7. Stripe settles the charge: the merchant's connected account receives
   the sale amount minus the platform fee; the platform's Stripe account
   receives the fee automatically. No invoicing or manual transfers.
8. A webhook (`payment_intent.succeeded`) confirms the result server-side
   and records the transaction for the dashboard.

## Error handling

- **Declined/failed payment**: widget surfaces the Stripe error message
  and allows retry; no transaction is recorded as successful.
- **Webhook verification**: all incoming webhooks are verified against
  the Stripe webhook signing secret; unverified requests are rejected.
- **Incomplete onboarding**: if a merchant's Connect account hasn't
  finished Stripe's onboarding requirements, the dashboard blocks
  generating a live embed snippet and shows what's missing.
- **Network/API errors**: widget shows a generic retry-safe error;
  no partial charges are possible since PaymentIntents are idempotent　
  per checkout attempt.

## Testing

- All development and demo happens in **Stripe test mode** with test
  card numbers (e.g. `4242 4242 4242 4242`).
- Stripe CLI (`stripe listen`) forwards webhooks to localhost during
  development.
- Manual end-to-end test: connect a test merchant account, embed the
  widget on a static test page, complete a test payment, confirm the
  application fee appears correctly split in the Stripe test dashboard.

## Tech stack

- **Widget**: Vanilla JS + Stripe.js/Elements, bundled with Vite.
- **Server**: Node.js, Express, Stripe Node SDK, SQLite (via
  `better-sqlite3`) for merchant/transaction records.
- **Dashboard**: React + Vite.
- **Package management**: npm workspaces (single monorepo, three
  packages).

## MVP definition of done

An end-to-end demo, entirely in Stripe test mode:

1. A merchant can sign up on the dashboard and connect a Stripe test
   account.
2. The merchant receives an embed snippet.
3. That snippet, dropped into a static HTML test page, renders a working
   checkout widget.
4. A test payment completes successfully.
5. The platform fee (`application_fee_amount`) is visibly applied and
   confirmed in the Stripe test dashboard and in the platform's own
   dashboard transaction list.

## Future sub-projects (explicitly deferred)

- Fraud/error detection add-on.
- Currency conversion / cross-border routing.
- Platform-specific plugins (Shopify, WooCommerce).
- Production onboarding, legal ToS, live-mode launch.
