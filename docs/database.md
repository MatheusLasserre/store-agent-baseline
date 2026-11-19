# Database, storage and access

Supabase provides Postgres, admin sign-in and file storage. All access goes
through server functions and server routes.

## Naming

- **Tables:** singular, snake_case (`book`, `order_item`). Quote reserved
  words (`"order"`).
- **Primary keys:** `{table}_id` (`book_id`, `order_id`).
- **Foreign keys:** the same name as the key they reference
  (`book_id uuid not null references book(book_id)`). Index every foreign key.
- **Lookup tables:** `lkp_` prefix (`lkp_order_status`), with a text code as
  the key. Mirror each one as a Zod enum in the module's `schemas.ts`.
- **Junction tables:** `xref_` prefix.
- **Money:** `bigint` columns ending in `_cents`.
- **Timestamps:** `timestamptz`, named `created_at`, `updated_at`, `paid_at`.

## Primary keys are UUID v7

Generate ids in the application with `v7()` from the `uuid` package, through
one helper (`src/lib/id.ts`). UUID v7 is time-ordered and indexes well. Don't
use `default gen_random_uuid()` on primary keys. Ids that come from another
system (a Supabase Auth user, a Mercado Pago payment) keep the value that
system gave them.

## Migrations

- Schema changes are SQL files in `supabase/migrations`, created with
  `npx supabase migration new <name>`. Never change the schema from the
  Supabase dashboard, and never edit a migration that has already been
  applied to production: write a new one.
- Migrations only add: new tables, new nullable columns, new indexes. The
  deployed app must keep working between the migration and the next deploy.
  Renaming or removing something that holds data follows rule 6 in
  [AGENTS.md](../AGENTS.md).
- After a migration, regenerate the database types
  (`npx supabase gen types typescript --local > src/server/database.types.ts`)
  and commit them. Never edit the generated file.
- Develop against the local Supabase stack (`npx supabase start`). Apply to
  production with `npx supabase db push`, before pushing the code that needs
  it, and only when the checks pass.

## Who can do what

There are two kinds of people: customers, who buy without an account, and
admins (the owner), who sign in with Supabase Auth.

- **Row level security is enabled on every table**, in the migration that
  creates it. Add no policies for the public: with RLS on and no policy, the
  publishable key can read and write nothing.
- **The browser never queries Supabase tables.** It uses Supabase only to
  sign an admin in and out.
- **Server code uses the secret key** (`SUPABASE_SECRET_KEY`) through the one
  client factory in `src/server/supabase.server.ts`. That key bypasses RLS,
  so every server function is responsible for its own check:
  - Public functions return only published books and only public columns.
  - Order functions are reached with the order's unguessable access token
    ([store.md](store.md#orders)), never a bare order id.
  - Admin functions first call one `requireAdmin()` helper. It reads the
    session from the request's cookies with `@supabase/ssr`, verifies it with
    `getClaims()` (never trust `getSession()` on the server), and confirms the
    user is in the `store_admin` table.
- You add an admin by creating the Supabase Auth user and inserting its id
  into `store_admin`, when Product asks. There is no public sign-up; keep
  sign-ups disabled in the Supabase project.

## Queries

- Select the columns you need, not `*`, in anything a customer triggers.
- No query inside a loop: fetch related rows in one query (an `in (...)`
  filter or an embedded select).
- Paginate every list that can grow (orders, books in the admin). Responses
  have the shape `{ data: [...], pagination: { page, page_size, total_count, total_pages } }`.
- Anything that must be all-or-nothing (creating an order with its items,
  marking an order paid and taking stock) is one Postgres function called
  with `rpc`, so it runs in one transaction. Postgres functions get RLS-safe
  settings: `security invoker`, an explicit `search_path`.

## Storage

| Bucket | Access | Holds |
| --- | --- | --- |
| `book-cover` | public | Cover images, resized before upload |
| `ebook-file` | private | The ebook files being sold |

Ebook files are never public and their paths never reach the browser. A
download is a short-lived signed URL created by the server after it has
checked that the order is paid ([store.md](store.md#ebooks)). Uploads happen
only from admin functions.

## Starting schema

A starting point, to refine as the store takes shape. Keep this section in
step with the migrations.

| Table | Purpose |
| --- | --- |
| `book` | One title in the collection: slug, title, author, description, cover path, published flag, display order |
| `lkp_book_format` | `ebook`, `print` |
| `book_format` | A book in one format: price in cents; for print, stock quantity and weight in grams; for ebook, the file path |
| `lkp_order_status` | `pending`, `paid`, `shipped`, `completed`, `cancelled`, `refunded` |
| `"order"` | Customer name and email, access token, status, totals in cents, shipping address and cost when it has print items, Mercado Pago payment id, timestamps |
| `order_item` | Order, book format, quantity, and the title and unit price copied at purchase time |
| `shipping_rate` | What delivery costs, by region ([store.md](store.md#shipping)) |
| `store_admin` | The Supabase Auth users allowed into `/admin` |
| `webhook_event` | Mercado Pago notifications already handled, for idempotency |

An order item copies the title and price when the order is created: changing
a book later never changes a past order.
