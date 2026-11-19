# Agent guide

You are building and maintaining a small online bookstore for a non-technical
owner. It sells one collection of books, each as an ebook, a printed book
shipped in Brazil, or both. Read this file fully before every task, then the
docs it points to for the area you touch.

`CLAUDE.md` only imports this file (`@AGENTS.md`), so every agent reads the
same rules: edit `AGENTS.md`, never `CLAUDE.md`.

## Where the rules live

| Area | Read |
| --- | --- |
| Stack, runtime limits, secrets | [docs/stack.md](docs/stack.md) |
| Folders, routes, modules, forms, React patterns | [docs/structure.md](docs/structure.md) |
| Database, migrations, storage, access rules | [docs/database.md](docs/database.md) |
| What the store does: catalog, orders, shipping, ebooks, admin | [docs/store.md](docs/store.md) |
| Mercado Pago checkout and webhook | [docs/payments.md](docs/payments.md) |
| Tests and the checks before a push | [docs/testing.md](docs/testing.md) |
| One-time account setup and deploys (for the technical helper) | [docs/setup.md](docs/setup.md) |
| Work the owner asked for but that isn't built yet | [docs/backlog.md](docs/backlog.md) |

When these docs don't decide something, choose the simplest option that fits
the stack, say which assumption you made, and write the decision into the
relevant doc in the same change.

## The owner

The owner doesn't read code and can't debug. That changes how you work:

- Talk to the owner in Brazilian Portuguese, in plain words, with no jargon.
  Say what changed in the store as a customer or the owner would see it.
- Code, identifiers, comments, commit messages and these docs are in English.
  Everything a customer or the owner reads on screen is in pt-BR.
- Ask before coding when a request can be read two ways that would look
  different to a customer. Otherwise decide, and state the assumption.
- Never ask the owner to edit code, run a migration by hand or read a stack
  trace. If something needs a technical person (a new account, a secret, a
  domain, a paid plan), say so plainly and stop that part of the work: it goes
  to the technical helper named in [docs/setup.md](docs/setup.md).
- The store is small and stays small. Don't add a separate backend, queues,
  caches, microservices or scaling work. Don't replace anything in the stack.

## Rules that are never broken

1. **Card data never touches this app.** Payment happens on Mercado Pago's
   hosted checkout. No card form, no card fields, no card numbers in logs.
2. **Secrets live only in environment variables** (Cloudflare secrets in
   production, `.dev.vars` locally, which is git-ignored). Never commit a
   secret, print one, or put one in code that reaches the browser.
3. **The browser is never trusted.** Prices, totals, shipping cost, stock and
   "is this order paid" are computed or checked on the server from the
   database. An order becomes paid only from the verified webhook
   ([docs/payments.md](docs/payments.md)).
4. **This runs on Cloudflare Workers, not Node.** No filesystem, no
   long-running process, no Node-only libraries
   ([docs/stack.md](docs/stack.md)).
5. **Every table has row level security on**, and the browser never queries
   the database directly ([docs/database.md](docs/database.md)).
6. **Never destroy data.** Migrations only add. Dropping or rewriting a table
   or column that holds data, deleting orders, or emptying a storage bucket
   needs the owner's explicit yes after you explained what is lost, plus a
   backup first.
7. **`main` is production.** A push to `main` deploys the store. Push only
   after the checks in [docs/testing.md](docs/testing.md) pass.
8. **Personal data is kept to the minimum.** Collect only what an order needs
   (name, email, and for printed books a delivery address and phone). Never
   log it, never send it to a service these docs don't name.

## How to work on a request

1. **State intent.** Open with one sentence saying what you'll do next.
2. **Plan.** Keep a short task list with concrete steps and update it as you go.
3. **Gather context.** Read the docs for the area, then the code. Prefer
   reading a whole file over many small reads. Reuse existing helpers and
   components instead of writing a second version.
4. **Implement.** Keep the change scoped to the request. No drive-by
   reformatting or refactors.
5. **Test.** New logic gets tests. Run the narrowest check first, then the
   rest of what [docs/testing.md](docs/testing.md) asks for.
6. **Document.** When behavior or a convention changes, update the doc that
   owns it in the same change. A doc describes the store as it is now.
7. **Summarize.** Tell the owner what changed, in pt-BR. Separately list what
   you ran and its result, and anything left undone.

Report honestly: if a check fails or a step was skipped, say so. Never claim
something works that you didn't run.

## Commits

- Small commits, one purpose each, imperative subject in English
  (`Add shipping rate table`).
- A commit that changes the database carries its migration.
- Work that takes more than one sitting goes on a branch and merges to `main`
  only when it's complete and checked. Half a feature never reaches `main`.

## Dependencies

Add a dependency only when the stack in [docs/stack.md](docs/stack.md) can't
do the job reasonably. Before adding one, confirm it runs on Cloudflare
Workers. Note the addition and the reason in the commit message and in
`docs/stack.md`. One lockfile, at the repository root, always committed.

## Things that need the technical helper

Stop and say so instead of working around any of these: creating or changing
accounts (Cloudflare, Supabase, Mercado Pago, Resend, the domain), anything
that costs money, rotating or adding a production secret, restoring a backup,
and anything touching invoices (nota fiscal) or legal text. Add the item to
[docs/backlog.md](docs/backlog.md) under "Needs the technical helper".
