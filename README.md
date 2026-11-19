# Bookstore agent baseline

A set of rules for a coding agent that builds and maintains a small online
bookstore for a non-technical owner. The store sells one collection of books
as ebooks and as printed books shipped in Brazil.

There is no application code here yet. Copy these files into a new
repository and ask the agent for the store you want: it reads `PERSONA.md`
and `AGENTS.md` first and builds within these rules.

## What's inside

| File | For |
| --- | --- |
| [PERSONA.md](PERSONA.md) | Who the agent is: the store's only engineer, talking to Product. It makes every technical decision and asks only product questions |
| [AGENTS.md](AGENTS.md) | How to work, and the rules that are never broken |
| [CLAUDE.md](CLAUDE.md) | Imports both, so Claude Code reads the same rules |
| [.claude/agents/second-opinion.md](.claude/agents/second-opinion.md) | A reviewer subagent the engineer may consult on rare, hard-to-undo decisions |
| [docs/stack.md](docs/stack.md) | The stack, the Cloudflare Workers limits, secrets |
| [docs/structure.md](docs/structure.md) | Folders, routes, modules, forms, React patterns |
| [docs/database.md](docs/database.md) | Supabase naming, migrations, access rules, storage |
| [docs/store.md](docs/store.md) | What the store does: catalog, checkout, orders, ebooks, shipping, admin |
| [docs/payments.md](docs/payments.md) | Mercado Pago hosted checkout and its webhook |
| [docs/testing.md](docs/testing.md) | Tests and the checks before a deploy |
| [docs/setup.md](docs/setup.md) | How the agent sets up accounts and access with the owner, deploys, costs |
| [docs/backlog.md](docs/backlog.md) | Parked work and questions waiting on the owner |

## The stack in one line

One TanStack Start app (React, TypeScript) on Cloudflare Workers, with
Supabase for the database, admin sign-in and files, Mercado Pago's hosted
checkout for payments and Resend for email. No separate backend.

## How to use it

No technical person is needed. The agent acts as the engineer and treats the
person as Product:

1. Open the repository with a coding agent and ask, in your own words, for
   what you want ("quero uma página inicial com os livros da coleção").
2. The agent decides everything technical on its own. It asks only about the
   store itself: what it should do, what it should say, and anything that
   costs money.
3. When an account is needed (hosting, database, payments, email), the agent
   walks you through creating it step by step and asks for access, so it can
   handle that service from then on.
4. The agent builds, checks and publishes. The site updates on every push to
   `main`.

`docs/store.md` describes a sensible default store. It's a starting point:
when the owner asks for something different, the agent builds that and
updates the doc.
