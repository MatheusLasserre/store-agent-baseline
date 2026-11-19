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
- `payer`: the customer's name and email.

The server stores the preference id on the order and redirects the customer
to the preference's `init_point`.

## The webhook

`POST /api/mercado-pago/webhook` is a server route. It is the only place an
order becomes paid. In order:

1. **Check the signature.** The `x-signature` header carries `ts` and `v1`.
   Build the manifest Mercado Pago documents (from the `data.id` query
   parameter, the `x-request-id` header and `ts`), compute HMAC-SHA256 with
   `MERCADO_PAGO_WEBHOOK_SECRET` using Web Crypto, and compare with `v1` in
   constant time. On a mismatch answer 401 and do nothing else.
2. **Ignore what isn't a payment** (`type` other than `payment`): answer 200.
3. **Fetch the payment from Mercado Pago**
   (`GET https://api.mercadopago.com/v1/payments/<data.id>`). Never trust the
   notification body for status or amount.
4. **Match it to the order** by `external_reference`. Check that the
   currency is BRL and the paid amount equals the order's total. A mismatch
   flags the order for the owner and does not mark it paid.
5. **Apply the status, idempotently.** One Postgres function, in one
   transaction: record the notification in `webhook_event` (unique on the
   payment id and status; a repeat changes nothing), set the order's status
   (`approved` → `paid`; `refunded` or `charged_back` → `refunded`;
   `cancelled` or `rejected` leave a `pending` order as it is until it
   expires), and take stock when it becomes paid.
6. **Answer 200 quickly.** Mercado Pago retries when it gets no 200/201
   within about 22 seconds. Send the emails after responding (`waitUntil`).

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
- The signature check, the amount check and the idempotent status change
  each have unit tests ([testing.md](testing.md)).
- Pending orders older than a few days are marked `cancelled` by a daily
  Cron Trigger. That job changes status only; it deletes nothing.
