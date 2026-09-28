# PLAY ticket buyer plan

Status: planning only. No buyer, scheduled task, or purchases have been created.

## Confirmed requirements

- Organiser: PLAY WEDNESDAYS, https://www.fatsoma.com/p/playwednesdays.
- Buy six tickets for the group on each weekly release.
- Spending limits: GBP 10 per ticket and GBP 60 total for all six, including all mandatory fees. Both limits must pass before payment; a cheaper ticket must not offset another ticket over its individual limit.
- User-specified release: Thursday at 10:00, Europe/London.
- Keep scheduling tied to UK local time across daylight-saving changes.
- Use Google Chrome as the default browser for research and the buyer.
- Run purchases automatically without the user being at their computer. Automatic payment is the intended final behaviour within an explicit spending limit, subject to checkout feasibility and any bank/site challenges.

## Findings from public pages, 28 September 2026

- PLAY's organiser page lists separate Wednesday events at Komedia, Bath. Discover each event's URL and date; do not hard-code one themed event forever.
- PLAY: MARIO KART, Wednesday 7 October, currently lists GENERAL RELEASE (BEFORE 11:30) opening Friday 2 October at 12:00pm, at GBP 7 plus GBP 1 booking fee.
- PLAY: PRIZE NIGHT, Wednesday 14 October, similarly lists public release on Friday 9 October at 12:00pm.
- The MARIO KART page exposes an Enter Promo Code control. Whether Thursday's release needs a code or another link is unconfirmed.
- The displayed ticket price would make six tickets GBP 48 before any further checkout-level charges. This is an observed price, not an approved spending limit.
- Fatsoma supports group orders, but organisers can limit quantities per order or ticket type. Six-ticket availability and the limit for this release remain unverified.
- Public-page extraction is not checkout verification. The browser tool reported that Chrome is unavailable, so quantity controls, basket reservation behaviour, and payment steps have not been inspected interactively.
- Fatsoma documents payment verification through card-issuer-controlled 3D Secure. A purchase may require user approval; neither a saved session nor a previously successful purchase guarantees unattended payment next week.

Sources:

- https://www.fatsoma.com/e/1rbmj6am/play-mario-kart
- https://www.fatsoma.com/e/jg0yax9a/play-prize-night
- https://support.fatsoma.com/en/articles/15381212-group-bookings-and-multiple-ticket-purchases
- https://www.fatsoma.com/policies/privacy
- https://support.fatsoma.com/en/articles/15376147-understanding-charges-on-your-fatsoma-order
- https://playwright.dev/docs/browsers

## Decisions still needed

1. Resolve the Thursday 10:00 vs Friday 12:00 difference. Identify any legitimate society/promo code or separate release link. Retain Thursday as the user's requested schedule until resolved; do not silently substitute Friday.
2. Confirm the acceptable ticket type and its entry deadline. Proposed default: exactly six acceptable tickets in one order; no partial purchase.
3. Confirm that each release targets the following Wednesday. Send an advance notice of the event title and date; matching events can proceed under the standing purchase rules without weekly approval. Support skipping a week and pausing future purchases.
4. Choose a hosting provider and hosting budget, plus a phone-accessible notification channel. Hosted operation is the recommended design; no server has been provisioned.
5. Verify how this event's web checkout accepts payment, including whether a saved payment method is available. Do not assume app payment features also work on the website. Arrange any initial login/payment setup before enabling automatic buying.

## Purchase limits

- Currency: GBP; represent money as integer pence.
- Quantity: exactly six tickets for one intended weekly event.
- Per-ticket ceiling: 1,000 pence, including ticket fees and an allocated share of any mandatory order fee. Allocate any rounding remainder conservatively so the full fee is counted.
- Order ceiling: 6,000 pence including all mandatory charges. Hosting costs are separate and not authorised by this ticket budget.
- Validate both ceilings against the final checkout immediately before payment. If either ceiling is exceeded, the currency differs, or the final price cannot be verified, stop and notify the user.
- Do not automatically increase the limits, purchase add-ons, or buy fewer tickets to fit the budget.

## Proposed implementation

Use TypeScript and Playwright with Google Chrome on an always-on hosted Linux x86-64 server, using a dedicated project browser profile. Explicitly select Playwright's `chrome` channel. Validate headless Chrome on the real checkout; if a visible session is needed, use Chrome on a server-side display with protected remote access for initial setup or exceptional recovery. The user's computer can be switched off.

Run a supervised service with a scheduler that understands Europe/London. Start the browser before release rather than relying on a machine starting at exactly 10:00. Keep configuration and a SQLite run ledger on persistent storage so restarts retain purchase history. Hosting is a separate recurring cost to choose before deployment. Local Chrome remains useful for development and rehearsal.

Store session state and any necessary secrets in protected persistent storage with encryption at rest and access limited to the service account. Keep them out of source control, logs, screenshots, and traces. Prefer a payment method retained by the checkout provider if supported; do not retain card security codes in the project.

Keep the design in five small components:

1. **Event discovery:** read the organiser's events, match the intended Wednesday and venue, and record the exact event ID/URL. Ambiguous matches stop the run.
2. **Session preparation:** establish the server's session during initial setup, reuse it, and check it the day before and just before release. Alert early if reauthentication is needed. Fatsoma documents Turnstile on sign-in, so support exceptional user intervention; do not promise that login can always renew unattended.
3. **Release watcher:** validate the selected release and its advertised opening time. Wait until sales open and use modest, bounded refreshes if needed. Preserve queue state if present, and back off on rate limits.
4. **Checkout:** select exactly six eligible tickets, verify availability and both the GBP 10 per-ticket and GBP 60 order limits including fees, and submit payment automatically under the configured standing purchase rules. Keep a dry-run mode for development. Read any actual reservation timer; do not assume selecting tickets reserves stock. If a bank/site challenge requires interaction, notify the user and stop when its deadline expires.
5. **Run ledger and status:** retain event ID, quantity, state, and confirmed order reference. Use a single-run lock and persist the payment-attempt state before submitting payment. Send the user a phone-accessible notification for success, failure, or required interaction. Provide a pause/skip control. Notification failure must never trigger another purchase.

## Weekly run

- Before release day: identify the intended event, resolve any release-time mismatch, validate the server's login session, and send advance notice. Normal matching runs need no weekly approval once purchase rules are configured.
- 09:55 Thursday: launch, validate configuration, login, connectivity, event, price constraints, and quantity limits.
- Before 10:00: keep the exact event page ready and show any issue requiring attention.
- 10:00: act as soon as the intended ticket release becomes available; select six and complete checkout automatically when payment can proceed without a challenge.
- On confirmed success: persist the order reference, prevent another purchase for that event, and send the user the event, quantity, total, and order confirmation status.
- On a bank or site challenge: alert the user promptly through the selected notification channel. If no response arrives before expiry, stop and report the purchase as incomplete; do not claim success.
- On timeout after payment submission: mark the result uncertain and reconcile with Fatsoma order history before any retry. Never treat a missing confirmation page as proof that payment failed.
- At the run deadline: stop retries and report the outcome. Do not make a surprise late purchase after a missed scheduled run.

The start and release times above remain provisional until the Thursday/public-release mismatch is resolved.

## Build sequence and acceptance criteria

1. **Inspect the actual release flow.** Establish access method, correct dates, six-ticket limit, entry conditions, checkout fees, session requirements, and reservation/payment behaviour.
2. **Build a manual-start rehearsal.** Discover and validate the event, reuse login, and produce a purchase summary. Use a local checkout simulation for repeatable purchase tests; avoid placing real orders or holding inventory during rehearsals.
3. **Build automatic checkout and recovery.** Test delayed release, fewer than six tickets, exact GBP 10/GBP 60 boundaries, a fee taking the total above GBP 60, one ticket above GBP 10 despite the order remaining under GBP 60, session expiry, wrong event, bank challenges, crashes immediately around payment submission, and ambiguous payment results. No case may cause an extra order.
4. **Add hosted weekly scheduling and notifications.** Verify UK daylight-saving transitions, duplicate launches, server restart/network failure, a missed release, and notification delivery failure. Confirm the service runs with the user's computer off. Provide a simple enable/disable control.
5. **Validate real checkout before relying on unattended purchases.** With the intended payment method configured and the user available for initial setup/any required authentication, verify the first real order's reference, date, quantity, and total. Subsequent normal runs target automatic completion without the user at their computer. A successful first run does not eliminate future bank challenges.

Success means one confirmed order containing six eligible tickets for the intended weekly event, with every ticket at most GBP 10 and the complete order at most GBP 60 including fees. Availability remains outside the buyer's control.
