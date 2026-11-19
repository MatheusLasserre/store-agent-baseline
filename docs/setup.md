# Setup and deploys

This file is for the technical helper who sets the accounts up once. The
agent reads it to know what exists, and never does these steps itself: they
create accounts, cost money or handle production secrets.

Prices and limits below were checked on 2026-11-19. Check the linked pages
again before relying on them.

## One-time checklist

1. **GitHub.** Create the store's repository from this baseline. The owner
   needs access; the coding agent works in a clone of it.
2. **Supabase.** Create a project in the São Paulo region. Disable public
   sign-ups (Authentication settings). Copy the project URL, the publishable
   key and the secret key. On the machine the agent runs on, run
   `npx supabase login` and `npx supabase link` once.
3. **Cloudflare.** Create an account and connect the GitHub repository with
   Workers Builds, production branch `main`. Each push to `main` then builds
   and deploys.
4. **Mercado Pago.** The owner's account, with an application for Checkout
   Pro. Copy the test and production access tokens. Configure the webhook
   (Suas integrações → Webhooks, event Pagamentos) to
   `https://<domain>/api/mercado-pago/webhook` and copy its secret.
5. **Resend.** Create an account, verify the store's domain, create an API
   key.
6. **Domain.** Point it at the Worker (Cloudflare custom domain).
7. **Secrets.** Set every variable from [stack.md](stack.md#configuration-and-secrets)
   in Cloudflare (Worker settings → Variables and Secrets, or
   `npx wrangler secret put <NAME>`). For local work, copy
   `.dev.vars.example` to `.dev.vars` with the test values.
8. **First admin.** Create the owner's user in Supabase Auth and insert its
   id into `store_admin`.
9. **Before launch.** A test purchase of an ebook and of a printed book, end
   to end, in production with a real low-value payment, then a refund. The
   owner's privacy and terms text is in place.

## Creating the app

The first task for the agent, in an empty clone of the repository:

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

- Code: push to `main`. Cloudflare builds and deploys it.
- Database: `npx supabase db push`, before the push that needs it
  ([database.md](database.md#migrations)).
- A bad deploy is rolled back from the Cloudflare dashboard (Deployments →
  roll back), then fixed with a new commit. Never force-push `main`.

## What it costs, and the limits that matter

| Service | Plan to start on | Watch for |
| --- | --- | --- |
| Cloudflare Workers | Free: 100,000 requests a day, 10 ms of CPU per request. Paid starts at US$ 5 a month. | Server rendering can exceed 10 ms of CPU. If the logs show CPU-limit errors, move to Paid. Whether Cloudflare's terms allow a shop on the free plan was not confirmed: read the terms, or start on Paid. |
| Supabase | Free | A free project is **paused after about a week of low activity** and its site stops working until someone resumes it in the dashboard. A few requests a day normally prevent that, with no guarantee. The free plan also has **no automatic backups**. Once the store takes real orders, Pro (from US$ 25 a month) is the safe choice. |
| Mercado Pago | No monthly fee | A fee per sale, by payment method. |
| Resend | Free tier | Daily and monthly sending limits. |

Sources:
[Workers limits](https://developers.cloudflare.com/workers/platform/limits/),
[Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/),
[Supabase pricing](https://supabase.com/pricing),
[Supabase project pausing](https://supabase.com/docs/guides/platform/free-project-pausing).

On Supabase Free, keep the store awake with a daily Cron Trigger that runs
one cheap query, and export the database on a schedule the helper sets up.
Both are stopgaps until Pro.
