# Stack

One TypeScript app, one repository, one deploy. There is no separate backend
service and there never will be one at this size.

| Layer | Choice |
| --- | --- |
| App | TanStack Start (React 19, TanStack Router, server functions), Vite |
| Data fetching and forms | TanStack Query, TanStack Form, Zod |
| UI | Tailwind CSS v4, shadcn/ui |
| Database, admin sign-in, file storage | Supabase (Postgres, Auth, Storage) |
| Payments | Mercado Pago Checkout Pro (hosted) |
| Email | Resend, over its HTTP API |
| Hosting | Cloudflare Workers, deployed on every push to `main` |
| Tooling | npm (one root lockfile), Biome (lint and format), Vitest, Playwright |

Pin exact versions in `package.json`. TanStack Start's API has changed between
releases: before using a Start or Router API, check the docs for the version
installed here rather than relying on memory. The same goes for Supabase and
Mercado Pago.

## Server functions hold everything private

Server functions (`createServerFn`) and server routes exist only in TanStack
Start; Router and Query alone run in the browser. The split is strict:

- **Server only:** every database query, every Mercado Pago or Resend call,
  every read of a secret, every price or stock decision.
- **Browser:** rendering, forms, navigation, calling server functions through
  the query hooks ([structure.md](structure.md#modules)).

Server-only code lives in files ending in `.server.ts` or under `src/server/`,
and is imported only from server function handlers and server routes. If a
secret or the Supabase secret key could end up in a browser bundle, the
change is wrong.

Every server function validates its input with a Zod schema before doing
anything else. Admin functions check the signed-in admin first
([database.md](database.md#who-can-do-what)).

## Cloudflare Workers is not Node

The app runs in the Workers runtime with the `nodejs_compat` flag. That
covers many Node built-ins but not all of Node:

- No filesystem. Files go to Supabase Storage.
- No long-running process, no in-memory state between requests, no
  `setInterval` jobs. Scheduled work uses a Cron Trigger; work after a
  response uses `waitUntil`.
- CPU time per request is short (10 ms on the free plan). Keep server work
  small: no image processing, no PDF generation, no heavy loops in a request.
  Resize images before upload, in the admin's browser.
- Outbound calls use `fetch`. Don't add an SDK when one or two `fetch` calls
  do the job: call Mercado Pago and Resend over HTTP. Never add Nodemailer,
  `pg`, Prisma, Express, `sharp` or anything else that needs TCP sockets, a
  native module or the filesystem.
- Use Web Crypto (`crypto.subtle`) for hashes and HMAC.
- Before adding any package, check that it supports Workers.

Run and test against the Workers runtime locally (`npm run dev` uses the
Cloudflare Vite plugin), so a Node-only import fails on your machine and not
in production.

## Configuration and secrets

- Every environment variable is declared once, in `src/server/env.server.ts`,
  in a Zod schema. That module reads the Workers environment
  (`import { env } from "cloudflare:workers"`), validates it and exports a
  typed object. No other file reads environment variables.
- A missing or malformed variable is an error that names the variable and
  never prints its value. Missing configuration never switches the app into a
  mock or demo mode.
- Values the browser needs (the site URL, the Supabase URL and publishable
  key) are the only ones exposed with the `VITE_` prefix. Everything else is
  server-only.
- `.env.example` and `.dev.vars.example` list every variable with a fake
  value and one line saying what it's for. Keep them current. `.env`,
  `.dev.vars` and anything else holding real values stay git-ignored.

Expected variables (extend the list here when one is added):

| Variable | Where | Purpose |
| --- | --- | --- |
| `VITE_SITE_URL` | public | The store's public address |
| `VITE_SUPABASE_URL` | public | Supabase project URL |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | public | Supabase publishable key |
| `SUPABASE_SECRET_KEY` | secret | Server-side database access |
| `MERCADO_PAGO_ACCESS_TOKEN` | secret | Creating checkouts, reading payments |
| `MERCADO_PAGO_WEBHOOK_SECRET` | secret | Checking webhook signatures |
| `RESEND_API_KEY` | secret | Sending email |
| `EMAIL_FROM` | secret | The sender address |

## Conventions that apply everywhere

- **Money is integer cents** (`price_cents: 4990` is R$ 49,90), in the
  database, in code and across the wire. Format only at display, with
  `Intl.NumberFormat("pt-BR", { style: "currency", currency: "BRL" })` through
  one helper in `src/lib/money.ts`. Never use floats for money.
- **Dates:** store timestamps as `timestamptz` in UTC. A calendar date shown
  to a person is computed in `America/Sao_Paulo`, through one helper.
- **Errors:** server functions throw a typed `AppError` (a code plus a pt-BR
  message safe to show). The UI shows messages through one
  `getErrorMessage` helper. Unexpected errors are logged on the server
  without personal data and shown as a generic pt-BR message.
- **No nested `if`/`else` chains to pick a value or component by type.** Use
  a mapping object, a `Map` or a small function.
- **Imports** use the `@/` alias for anything under `src`. Helpers used by
  one feature stay in that feature's folder.
