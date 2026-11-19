# Setup, access and deploys

You set the store's services up yourself, together with Product. They do
only what needs a person (create the account, confirm an email, enter a
card); you do everything else and end each step holding the access you need
to never send them back. [PERSONA.md](../PERSONA.md#when-something-outside-the-code-is-needed)
says how to guide them and how to handle what they give you.

Don't set everything up on day one. Do each service when the work first
needs it, in this order, and keep the status table current.

Prices and limits below were checked on 2026-11-19. Check the linked pages
again before quoting a price to Product.

## Status

Update this table as access is gained. Never write a secret here: only
whether it exists and where it's stored.

| Service | Account | Your access | Notes |
| --- | --- | --- | --- |
| GitHub | not yet | not yet | |
| Cloudflare | not yet | not yet | |
| Supabase | not yet | not yet | |
| Mercado Pago | not yet | not yet | |
| Resend | not yet | not yet | |
| Domain | not yet | not yet | |

## The services

For each: what Product does, the access you get, and what you then do alone.

### GitHub (first)

- **Product:** creates a GitHub account if they have none.
- **Access:** `gh auth login` (opens their browser).
- **You:** create the store's private repository, push, and manage it from
  then on.

### Supabase (when the first table is needed)

- **Product:** creates the account (signing in with GitHub is the shortest
  path).
- **Access:** `npx supabase login` (opens their browser).
- **You:** create the project in the São Paulo region with the CLI, link it,
  disable public sign-ups, read the project URL and keys, apply migrations,
  create storage buckets. Create Product's admin user and add it to
  `store_admin`; they set their own password through the reset email.

### Cloudflare (when there is something to show)

- **Product:** creates the account and confirms their email.
- **Access:** `npx wrangler login` (opens their browser).
- **You:** deploy, set every secret with `npx wrangler secret put`, add Cron
  Triggers. Connecting the repository so pushes to `main` deploy on their own
  (Workers Builds) needs a few clicks in the dashboard: guide Product through
  them once. Until then, deploy with `npm run deploy`.

### Mercado Pago (when checkout is built)

- **Product:** uses or creates their Mercado Pago account, since the money
  lands there. In Suas integrações they create an application for Checkout
  Pro; you give the exact clicks.
- **Access:** the application's test and production access tokens, and the
  webhook secret, pasted into the git-ignored file you prepared.
- **You:** configure the webhook URL if the API allows it (otherwise guide
  the clicks: Webhooks, event Pagamentos,
  `https://<domain>/api/mercado-pago/webhook`), store the secrets in
  Cloudflare, run the test purchases.

### Resend (when the first email is built)

- **Product:** creates the account.
- **Access:** an API key with full access, pasted into the git-ignored file.
- **You:** add and verify the sending domain through the API, add the DNS
  records it asks for, store the key in Cloudflare.

### Domain (before launch)

- **Product:** chooses the name and pays for it. This is a product and money
  decision: offer two or three available options with prices.
- **Access:** simplest is to register or transfer the domain in Cloudflare,
  where you already have access. A domain kept elsewhere needs Product to
  change its nameservers to Cloudflare once; give the exact steps.
- **You:** DNS records, the Worker's custom domain, email DNS for Resend.

If a CLI sign-in isn't possible on the machine you run on, ask for an API
token instead, with only the permissions the store needs, and say exactly
which boxes to tick.

## Creating the app

In an empty clone of the repository:

```sh
npm create cloudflare@latest -- . --framework=tanstack-start
```

That scaffolds TanStack Start with the Cloudflare Vite plugin, a
`wrangler.jsonc` (keep `nodejs_compat` in `compatibility_flags`) and the
`deploy` script. Then add Tailwind, shadcn/ui, Biome, Vitest, Playwright and
the Supabase CLI, following [stack.md](stack.md) and
[structure.md](structure.md). References:
[TanStack Start hosting guide](https://tanstack.com/start/latest/docs/framework/react/guide/hosting),
[Cloudflare's TanStack Start guide](https://developers.cloudflare.com/workers/framework-guides/web-apps/tanstack-start/).

## Deploys

- Code: push to `main` (or `npm run deploy` before Workers Builds is
  connected).
- Database: `npx supabase db push`, before the push that needs it
  ([database.md](database.md#migrations)).
- A bad deploy: roll back with `npx wrangler rollback`, tell Product in one
  sentence if customers were affected, then fix with a new commit. Never
  force-push `main`.

## Before launch

Yours to run, not Product's: a purchase of an ebook and of a printed book,
end to end, in production with a real low-value payment, then a refund
(Product approves the payment in their own Mercado Pago app). Product's
privacy and terms text is in place. A database export exists.

## What it costs, and the limits that matter

Costs are Product's decision; the technical judgment of when a paid plan is
needed is yours. Bring it to them as a price and a consequence.

| Service | Plan to start on | Watch for |
| --- | --- | --- |
| Cloudflare Workers | Free: 100,000 requests a day, 10 ms of CPU per request. Paid starts at US$ 5 a month. | Server rendering can exceed 10 ms of CPU. If the logs show CPU-limit errors, recommend Paid. Whether Cloudflare's terms allow a shop on the free plan was not confirmed when this was written: read the current terms before launch and recommend Paid if they don't. |
| Supabase | Free | A free project is **paused after about a week of low activity** and the store stops working until it's resumed. A few requests a day normally prevent that, with no guarantee. The free plan also has **no automatic backups**. Once the store takes real orders, recommend Pro (from US$ 25 a month). |
| Mercado Pago | No monthly fee | A fee per sale, by payment method. Product should know it before setting prices. |
| Resend | Free tier | Daily and monthly sending limits. |

Sources:
[Workers limits](https://developers.cloudflare.com/workers/platform/limits/),
[Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/),
[Supabase pricing](https://supabase.com/pricing),
[Supabase project pausing](https://supabase.com/docs/guides/platform/free-project-pausing).

While on Supabase Free, you keep the store awake with a daily Cron Trigger
that runs one cheap query, and you export the database on a schedule to
private storage. Both are stopgaps until Pro.
