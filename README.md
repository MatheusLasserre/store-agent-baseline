# Bookstore agent baseline

Rules for a coding agent that builds and maintains a small online bookstore
for a non-technical owner. The store sells one collection of books as ebooks
and as printed books shipped in Brazil.

There is no application code here. This repository is a starting point: a
new store gets its own private copy of these files, and the agent builds the
store inside it.

> **Para quem vai usar (não precisa ser técnico):** abra seu agente de
> programação em uma pasta vazia e diga: *"Use
> https://github.com/MatheusLasserre/store-agent-baseline como base e
> comece a minha loja de livros."* Ele vai dizer, passo a passo, o que você
> precisa instalar e quais contas criar. Depois é só pedir a loja que você
> quer.

Everything below is written for the agent.

## If you are the agent

You were pointed at this repository to start or continue a store.

- **Starting a new store** (the folder you're in has no `AGENTS.md`): follow
  [Starting a new store](#starting-a-new-store) below, in order.
- **Continuing a store** (the folder already has `AGENTS.md` and
  `PERSONA.md`): ignore this README. Read `PERSONA.md`, then `AGENTS.md`,
  and work as they say.

In both cases the same two things hold from the first message:

1. You are the store's only engineer and the person is Product
   ([PERSONA.md](PERSONA.md)). Every technical decision is yours; you ask
   them only about the store itself and about money. Talk to them in
   Brazilian Portuguese, in plain words.
2. You work on a professional setup and ask for it once, up front. You never
   work around a missing tool or account.

## Starting a new store

Do these in order. Steps 1 to 4 are one sitting with Product; tell them at
the start roughly how long it will take and what they'll have at the end.

### 1. Read the rules

Read [PERSONA.md](PERSONA.md) and [AGENTS.md](AGENTS.md) in full, then
[docs/setup.md](docs/setup.md). Skim the other docs so you know what each
one owns.

### 2. Ask the few product questions that shape the setup

Only these, in one message, each with a suggestion:

- The store's name (it names the repository, the project and the sender of
  the emails).
- Whether they already have a domain, a Mercado Pago account and a GitHub
  account.

Everything else about the store can wait until it's running.

### 3. Preflight the machine

Run the [preflight](docs/setup.md#preflight): inspect the machine, install
what you can, and give Product one list of what only they can do (on
Windows: approve the installers, restart once for Docker Desktop, open it
and accept its terms). Verify every tool before moving on.

### 4. Create the accounts and get access

Go through [the services](docs/setup.md#the-services) with Product: GitHub,
Supabase, Cloudflare, Mercado Pago, Resend, and the domain if they have or
want one now. For each, they create the account and you get lasting access
in the same step. Update the status tables in `docs/setup.md`.

### 5. Create the store's repository

The store lives in its own **private** repository, not in a fork of this
one:

```sh
gh repo create <store-name> --private --clone --template MatheusLasserre/store-agent-baseline
```

If that fails, clone this repository, delete its `.git` folder, run
`git init -b main`, and create the private repository from the result with
`gh repo create <store-name> --private --source=. --push`.

Then, in the new repository:

- Replace this `README.md` with a short one for the store: its name, what it
  is, and a line saying the rules are in `PERSONA.md` and `AGENTS.md`.
- Add `.gitattributes` with `* text=auto eol=lf`.
- Commit.

### 6. Create the app and put it online

Scaffold the app as [Creating the app](docs/setup.md#creating-the-app)
describes, add the tooling, and make the checks in
[docs/testing.md](docs/testing.md) pass on the empty app. Create the
Supabase project and the Cloudflare Worker, set the secrets, and deploy a
simple "em breve" page with the store's name.

Show Product the live address. From here on, every push to `main` updates
the store.

### 7. Ask what they want

Now ask Product for the store they want, and build it within
[docs/store.md](docs/store.md), which describes a sensible default. When
they ask for something different, build that and update the doc.

## What's inside

| File | For |
| --- | --- |
| [PERSONA.md](PERSONA.md) | Who the agent is: the store's only engineer, talking to Product. Read first, every session |
| [AGENTS.md](AGENTS.md) | How to work, and the rules that are never broken |
| [CLAUDE.md](CLAUDE.md) | Imports both, so Claude Code reads the same rules |
| [.claude/agents/second-opinion.md](.claude/agents/second-opinion.md) | A reviewer subagent for rare, hard-to-undo decisions |
| [docs/principles.md](docs/principles.md) | The engineering approaches behind every decision |
| [docs/setup.md](docs/setup.md) | Preflight, tools, accounts and access, deploys, costs |
| [docs/stack.md](docs/stack.md) | The stack, the Cloudflare Workers limits, secrets |
| [docs/structure.md](docs/structure.md) | Folders, routes, modules, forms, React patterns |
| [docs/database.md](docs/database.md) | Supabase naming, migrations, access rules, storage |
| [docs/store.md](docs/store.md) | What the store does: catalog, checkout, orders, ebooks, shipping, admin |
| [docs/payments.md](docs/payments.md) | Mercado Pago hosted checkout and its webhook |
| [docs/testing.md](docs/testing.md) | Tests and the checks before a deploy |
| [docs/backlog.md](docs/backlog.md) | Parked work and questions waiting on Product |

## The stack in one line

One TanStack Start app (React, TypeScript) on Cloudflare Workers, with
Supabase for the database, admin sign-in and files, Mercado Pago's hosted
checkout for payments and Resend for email. No separate backend.
