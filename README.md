# Bookstore agent baseline

A set of rules for a coding agent that builds and maintains a small online
bookstore for a non-technical owner. The store sells one collection of books
as ebooks and as printed books shipped in Brazil.

There is no application code here yet. Copy these files into a new
repository, do the one-time setup, and then ask the agent for the store you
want: it reads `AGENTS.md` first and builds within these rules.

## What's inside

| File | For |
| --- | --- |
| [AGENTS.md](AGENTS.md) | The agent's entry point: how to work, and the rules that are never broken |
| [CLAUDE.md](CLAUDE.md) | Imports `AGENTS.md`, so Claude Code reads the same rules |
| [docs/stack.md](docs/stack.md) | The stack, the Cloudflare Workers limits, secrets |
| [docs/structure.md](docs/structure.md) | Folders, routes, modules, forms, React patterns |
| [docs/database.md](docs/database.md) | Supabase naming, migrations, access rules, storage |
| [docs/store.md](docs/store.md) | What the store does: catalog, checkout, orders, ebooks, shipping, admin |
| [docs/payments.md](docs/payments.md) | Mercado Pago hosted checkout and its webhook |
| [docs/testing.md](docs/testing.md) | Tests and the checks before a deploy |
| [docs/setup.md](docs/setup.md) | One-time account setup, deploys, costs |
| [docs/backlog.md](docs/backlog.md) | Parked work |

## The stack in one line

One TanStack Start app (React, TypeScript) on Cloudflare Workers, with
Supabase for the database, admin sign-in and files, Mercado Pago's hosted
checkout for payments and Resend for email. No separate backend.

## How to use it

1. Someone technical follows [docs/setup.md](docs/setup.md) once: accounts,
   the GitHub connection, secrets, the domain.
2. The owner opens the repository with a coding agent and asks, in their own
   words, for what they want ("quero uma página inicial com os livros da
   coleção").
3. The agent builds it, runs the checks and pushes. The site redeploys on
   every push to `main`.

`docs/store.md` describes a sensible default store. It's a starting point:
when the owner asks for something different, the agent builds that and
updates the doc.
