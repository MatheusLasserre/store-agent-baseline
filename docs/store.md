# What the store does

This describes the baseline. The owner will ask for the store they want;
when a request changes any of this, update this file in the same change.

## Catalog

- One collection of books. No categories, tags, search engine or multiple
  storefronts: the home page lists the collection in the owner's order.
- A book is sold as an ebook, a printed book, or both. Each format has its
  own price. A printed format has stock; an ebook never runs out.
- Only published books appear. A printed format with no stock shows as
  "Esgotado" and can't be added to the cart.

## Cart

The cart lives in the customer's browser (local storage): a list of book
format ids and quantities, nothing else. It holds no prices. The cart page
and the checkout ask the server for current titles, prices and stock, so a
stale or edited cart can't change what is charged.

## Checkout

Customers buy without creating an account.

1. The customer gives name and email. If the cart has a printed book, also a
   phone and a delivery address in Brazil. Look the address up from the CEP
   (ViaCEP, a free HTTP service) and let the customer correct it.
2. The server recomputes everything from the database: items, prices, stock,
   shipping cost, total.
3. The server creates the order as `pending` with its items, then a Mercado
   Pago checkout for it, and sends the customer there
   ([payments.md](payments.md)).
4. Mercado Pago sends the customer back to the order page.

## Orders

- Every order has an unguessable access token (at least 128 random bits,
  from Web Crypto). The order page is `/pedido/<token>`; that link is in the
  customer's emails. An order id alone never opens an order.
- Statuses: `pending` → `paid` → `shipped` → `completed`, or `cancelled` /
  `refunded`. An order with only ebooks goes from `paid` straight to
  `completed`.
- Only the verified Mercado Pago webhook moves an order to `paid`. The page
  the customer returns to only displays what the database says, and shows
  "Aguardando confirmação do pagamento" while it's still `pending` (Pix and
  boleto can take a while).
- When an order becomes paid, in the same transaction: stock of its printed
  items goes down. Then the customer gets a confirmation email with the
  order link, and the owner gets a "novo pedido" email.
- A `pending` order holds no stock. If stock ran out by the time payment
  arrives, the order is still marked paid and flagged for the owner, who
  decides between restocking and refunding. Never silently drop a paid order.

## Ebooks

- A paid order's page shows a download button per ebook. The button calls a
  server function that checks the token and the paid status, then returns a
  signed URL valid for a few minutes.
- Downloads are counted per order item and limited (default: 5). Past the
  limit the page says to contact the store.
- No DRM, watermarking or reader app unless the owner asks for one.

## Shipping

- Brazil only. The baseline is a table the owner edits in the admin
  (`shipping_rate`): a price per region, by state or CEP range, plus an
  optional "frete grátis acima de" amount. The server picks the rate from
  the CEP.
- An order with only ebooks has no address and no shipping.
- The owner ships the parcel themselves, then marks the order as shipped in
  the admin and may enter a tracking code. The customer gets an email with it.
- Live carrier quotes and label printing (Correios, Melhor Envio) are not in
  the baseline. Build them only when the owner asks; they need an account
  from the technical helper.

## Admin

`/admin`, for signed-in admins only, in pt-BR, usable on a phone:

- Books: create, edit, publish, reorder; upload a cover and an ebook file;
  set each format's price and stock.
- Orders: list with status filter, open one, mark shipped with a tracking
  code, see flagged orders first.
- Shipping rates.

Refunds are made by the owner in the Mercado Pago dashboard; the webhook then
updates the order.

## Email

Sent through Resend's HTTP API from the server, in pt-BR, plain and short:
order received (with the Pix or boleto still pending), payment confirmed
(with the order link), order shipped (with tracking). An email failing must
never fail the order: log it and let the owner resend from the admin.

## Privacy

- Collect only what the order needs. No CPF, birth date or marketing opt-in
  unless the owner asks and says why.
- No analytics or tracking scripts unless the owner asks; if one is added it
  comes with a consent notice.
- The store has `/privacidade` and `/termos` pages. Their text comes from the
  owner. Never invent legal text: leave a clearly marked placeholder and tell
  the owner it's needed before launch.
- Customer data is never written to logs or error reports.

## Not in the baseline

Customer accounts, coupons, reviews, wishlists, multiple currencies or
languages, invoices (nota fiscal), subscriptions, marketplaces. Each is
built only on the owner's request and recorded here when it is.
