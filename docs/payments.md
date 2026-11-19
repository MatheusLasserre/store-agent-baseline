# Payments: Mercado Pago Checkout Pro

The customer pays on Mercado Pago's own page (Pix, boleto, card, Mercado Pago
balance) and comes back. This app never sees card data and never renders a
payment form.

Read Mercado Pago's current Checkout Pro docs before writing or changing
this code: <https://www.mercadopago.com.br/developers/pt/docs/checkout-pro/overview>.
Field names and endpoints below were correct when this was written; the
official docs win where they differ, and this file gets corrected.

All of this lives in `src/modules/checkout/` and
`src/routes/api/mercado-pago.webhook.ts`, calls Mercado Pago with `fetch`
(no SDK; the app runs on Workers), and reads its two secrets from
`env.server.ts`.

## The standard for this code

This is the one part of the store where a mistake costs the owner real
money or gives a product away. Work on it with double the care of anything
else:

- Any change to checkout, the webhook, order statuses, stock or ebook
  downloads is "a money change". It gets the full
  [money test suite](testing.md#the-money-suite) and a complete
  [dry run](#dry-run), every time, however small the change looks.
- A money change qualifies for a second opinion
  ([PERSONA.md](../PERSONA.md#when-you-need-a-second-opinion)). Use it when
  the change touches how an order becomes paid, refunded or delivered.
- Never weaken or skip a money test to get a change through, and never
  "temporarily" bypass a check to debug in production.
- When in doubt between charging or delivering wrongly and stopping, stop:
  leave the order as it is, flag it for the owner, and say so.

## What must never happen

Each line is an invariant with at least one test that tries to break it.

**Nothing is delivered without payment**

1. An order is `paid` only through the one function that applies a verified
   payment. No other code path, admin screen, script or return URL sets it.
2. Every delivery checks the order's status in the database at that moment:
   each ebook download, each time. A link that worked yesterday stops
   working the moment the order is refunded.
3. A payment counts only if all of these hold, checked against what Mercado
   Pago's API returns (never the notification body): status approved; the
   `external_reference` is an existing order; the currency is BRL; the
   amount equals the order's total exactly; the payment was received by the
   store's own Mercado Pago account; and it's a live payment when running
   in production (a test payment can never pay a real order, and the
   reverse).
4. An order's items and total are fixed when the checkout is created. A
   changed cart is a new order. Totals are computed on the server from the
   database, with quantities validated as positive whole numbers within a
   sane maximum, and a total of zero or less refused.
5. The webhook does nothing for a request whose signature is missing,
   malformed, wrong or older than a few minutes, and it does that before
   touching the database.

**Nothing is processed twice**

6. A payment is applied at most once: a unique constraint on the Mercado
   Pago payment id, and the status change, the stock change and the
   idempotency record in one transaction that locks the order row. Two
   deliveries of the same notification arriving at the same instant give
   the same result as one.
7. Stock goes down exactly once per paid order and comes back exactly once
   if a paid, unshipped order is refunded.
8. Each email is sent at most once per order and kind, tracked in the
   database, and sent after the transaction commits. A failed email never
   undoes a payment, and a retried webhook never sends it again.

**The owner doesn't lose money**

9. Statuses only move forward along the allowed transitions (below). A
   late "pending" or "in process" notification never un-pays an order, and
   notifications arriving out of order end in the right state.
10. A refund or chargeback immediately cuts off ebook downloads and, if a
    printed order hasn't shipped, pulls it from the to-ship list. If it has
    shipped, it's flagged.
11. Money the store shouldn't keep is never silent. A second approved
    payment for the same order, an amount above the total, or a payment for
    a cancelled order is recorded and flagged so the owner can refund it;
    an amount below the total never marks the order paid and is flagged.
12. A paid order is never lost. If the webhook can't finish its work (the
    database is down, an unexpected error), it answers with an error so
    Mercado Pago retries; it answers 200 only after the change is
    committed. Missed notifications are caught by
    [reconciliation](#reconciliation).
13. Checkouts expire: the preference carries an expiry, and a pending order
    is cancelled after it, so an old link can't be paid at an old price
    weeks later.
14. Production and test credentials never mix. The health check reports
    which mode the store is in, and a production deploy with test
    credentials (or the reverse) is treated as a failed deploy.

Every flagged order shows first in the admin and sends the owner an email
that says, in plain pt-BR, what happened and what to do.

## Order statuses

| From | To | Caused by |
| --- | --- | --- |
| `pending` | `paid` | A verified approved payment |
| `pending` | `cancelled` | Expiry, or a verified cancelled or rejected payment once the checkout has expired |
| `paid` | `shipped` | The owner, in the admin (printed items only) |
| `paid` / `shipped` | `completed` | Ebook-only orders at once; printed orders when the owner marks them delivered |
| `paid` / `shipped` / `completed` | `refunded` | A verified refund or chargeback |

No other transition exists. The transition table lives in one place in the
code, the database function enforces it, and anything else is rejected and
logged. `cancelled` and `refunded` are final: a payment that arrives for a
final order is recorded and flagged (invariant 11), not applied.

## Reconciliation

Webhooks can be delayed or lost, so the webhook is not the only path to the
truth; it's the fastest one. Both of these go through the same
apply-payment function and the same checks:

- **On the order page:** while an order is `pending`, loading its page asks
  Mercado Pago for payments with that `external_reference` (rate-limited),
  so a customer who paid isn't left waiting on a lost notification.
- **Daily, by Cron Trigger:** compare the last several days of payments in
  Mercado Pago with the orders. Approved there and not paid here: apply it.
  Paid here and not approved there, or any amount that doesn't match: flag
  it loudly. The job reports its counts in the log, and a mismatch emails
  the owner.

## Dry run

Almost every case can be exercised without real money, and must be. The
store has a payment simulator from the day checkout is built:

- **A local fake Mercado Pago** (a small HTTP fake for development, MSW in
  tests) that creates preferences, serves payments in any state, and can
  return errors, delays and malformed responses.
- **`npm run pay:sim -- <scenario>`** runs a whole purchase locally against
  the local database: creates an order through the real server functions,
  plays the scenario's notifications at the real webhook route, signed with
  the test secret, and then reads the database back and checks the
  invariants. It prints one line per check and fails loudly.
- **Scenarios**, at least: approved (card); Pix pending then approved;
  boleto approved days later; rejected; the same notification three times;
  two identical notifications in parallel; refund before and after
  shipping; chargeback; refund notification arriving before the approval;
  a late "pending" after approval; amount lower and higher than the total;
  wrong currency; a payment for an unknown, cancelled or already paid
  order; a second payment for the same order; a test payment against
  production mode; missing, wrong and stale signatures; the database
  failing mid-webhook then recovering on retry; the email service down; a
  notification that never arrives (reconciliation picks it up); payment
  after the checkout expired; stock gone by the time payment arrives.
- **`npm run pay:sim -- all`** runs every scenario. It's part of the checks
  for every money change.

After the dry run, before a money change is called done:

1. In Mercado Pago's test mode, with test buyers: one approved purchase and
   one refund, end to end on the deployed store.
2. At launch, and after any change to how an order becomes paid: one real
   low-value purchase and its refund in production, approved by Product in
   their own Mercado Pago app.
3. Read the order back in the database and in Mercado Pago after each, and
   confirm they agree.

Say exactly which of these three levels you ran. A dry run is evidence
about the store's logic, not about Mercado Pago's real behavior.

## Creating a checkout

After the order is saved as `pending`, the server creates a payment
preference (`POST https://api.mercadopago.com/checkout/preferences`,
`Authorization: Bearer <access token>`), with:

- `items`: title, quantity and `unit_price` from the database (Mercado Pago
  takes reais as a decimal number: convert from cents here, in one helper,
  and nowhere else), `currency_id: "BRL"`. Shipping is one more item or the
  preference's shipping cost.
- `external_reference`: the order id. This is how a payment finds its order.
- `back_urls`: success, pending and failure all point at the order page
  (`/pedido/<token>`), with `auto_return: "approved"`.
- `notification_url`: the webhook route's public HTTPS address.
- An expiry (`expiration_date_to`), matching when the pending order is
  cancelled.
- `payer`: the customer's name and email.

The server stores the preference id on the order and redirects the customer
to the preference's `init_point`.

## The webhook

`POST /api/mercado-pago/webhook` is a server route. It is the only place an
order becomes paid. In order:

0. **Limit the request** before anything else: POST only, a small maximum
   body size, and a rate limit.
1. **Check the signature.** The `x-signature` header carries `ts` and `v1`.
   Build the manifest Mercado Pago documents (from the `data.id` query
   parameter, the `x-request-id` header and `ts`), compute HMAC-SHA256 with
   `MERCADO_PAGO_WEBHOOK_SECRET` using Web Crypto, and compare with `v1` in
   constant time. Reject a `ts` more than a few minutes old. On any failure
   answer 401 and do nothing else.
2. **Ignore what isn't a payment** (`type` other than `payment`): answer 200.
3. **Fetch the payment from Mercado Pago**
   (`GET https://api.mercadopago.com/v1/payments/<data.id>`). Never trust the
   notification body for status or amount.
4. **Match it to the order** by `external_reference` and run every check of
   invariant 3. A mismatch flags the order for the owner and does not mark
   it paid.
5. **Apply the status, idempotently.** One Postgres function, in one
   transaction: record the notification in `webhook_event` (unique on the
   payment id and status; a repeat changes nothing), set the order's status
   (`approved` → `paid`; `refunded` or `charged_back` → `refunded`;
   `cancelled` or `rejected` leave a `pending` order as it is until it
   expires), and take stock when it becomes paid.
6. **Answer 200 only after the transaction commits**, and quickly: Mercado
   Pago retries when it gets no 200/201 within about 22 seconds. If the
   work failed, answer 500 so it retries. Send the emails after responding
   (`waitUntil`), from the database's record of what is still unsent.

Mercado Pago may send the same notification several times, and out of
order. Running the handler twice for one payment must leave the same result
as running it once.

## Rules

- The return page never marks anything paid. It reads the order.
- Never log the access token, the webhook secret, the signature header or
  the payer's details.
- Test with Mercado Pago's test credentials and test buyers. Test payments
  don't send real notifications: the webhook is exercised with the
  dashboard's simulator and with the tests in this repo, which sign a sample
  body with a test secret.
- Every invariant above has tests, and every scenario has a dry run
  ([testing.md](testing.md#the-money-suite)).
- Pending orders past their checkout's expiry are marked `cancelled` by a
  daily Cron Trigger. The expiry is long enough for a boleto to be paid and
  confirmed (several days). That job changes status only; it deletes
  nothing.
