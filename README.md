# Income

An experiment in earning a small fee across a high volume of digital
transactions — legally, and only with explicit opt-in from every party
involved.

## Current sub-project: embeddable checkout widget

The first piece being built is a drop-in checkout widget that merchants
embed on their own sites. It's built on top of [Stripe
Connect](https://stripe.com/connect), so:

- Stripe (a licensed payment processor) is the only party that ever
  moves money.
- Merchants explicitly opt in by connecting their Stripe account via
  OAuth, agreeing to the platform fee as part of that connection.
- The platform earns a small `application_fee_amount` on each
  transaction that flows through the widget — the same mechanism
  Shopify and other platforms use on top of Stripe Connect.

See the full design in
[docs/superpowers/specs/2026-09-19-checkout-widget-design.md](docs/superpowers/specs/2026-09-19-checkout-widget-design.md).

The MVP runs entirely in Stripe **test mode** — no real money moves
until there's a reason to go live.

## Planned sub-projects (later, each with its own design)

- Fraud/error detection add-on
- Currency conversion / cross-border routing

## Status

Design spec approved. Implementation plan and build in progress.
