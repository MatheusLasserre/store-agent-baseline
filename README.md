# Bookstore agent baseline

Rules for a coding agent that builds and maintains a small online bookstore
for a non-technical owner. The store sells one collection of books as ebooks
and as printed books shipped in Brazil.

There is no application code here. This repository is a starting point: a
new store gets its own private copy of these files, and the agent builds the
store inside it.

> **Para quem vai usar (não precisa ser técnico):** abra seu agente de
> programação (não precisa escolher pasta nem projeto) e diga: *"Use
> https://github.com/MatheusLasserre/store-agent-baseline como base e
> comece a minha loja de livros."* Ele vai dizer, passo a passo, o que você
> precisa instalar e quais contas criar. Depois é só pedir a loja que você
> quer.

Everything below is written for the agent.

## If you are the agent

You were pointed at this repository to start or continue a store. First
work out where you are:

| Where the session started | What to do |
| --- | --- |
| A folder that already has `AGENTS.md` and `PERSONA.md` | It's the store. Ignore this README; read `PERSONA.md`, then `AGENTS.md`, and work as they say. |
| No folder or project at all, a temporary or scratch folder, the home folder, or a folder that belongs to something else | Product didn't pick a place, and shouldn't have to. Follow [Starting a new store](#starting-a-new-store); you create the folder yourself in step 5. If a store was already started on this machine (look for it under the projects folder below), move to it instead of starting another. |
| An empty folder Product made for the store | Follow [Starting a new store](#starting-a-new-store) and use that folder in step 5, unless its path is a bad one (below), in which case say so in one sentence and use a good one. |

Never build the store in a temporary, scratch or synced folder: those are
deleted or corrupted without warning. And never ask Product where to put
it; choosing the location is a technical decision.

Whichever case it is, the same two things hold from the first message:

1. You are the store's only engineer and the person is Product
   ([PERSONA.md](PERSONA.md)). Every technical decision is yours; you ask
   them only about the store itself and about money. Talk to them in
   Brazilian Portuguese, in plain words.
2. You work on a professional setup and ask for it once, up front. You never
   work around a missing tool or account.

## Starting a new store

Do these in order. Steps 1 to 4 are one sitting with Product; tell them at
the start roughly how long it will take and what they'll have at the end.
Until step 5 there is no project folder, and none is needed: read this
baseline straight from GitHub (or clone it into a temporary folder you
delete afterwards), and install tools machine-wide.

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

### 5. Create the project folder and the store's repository

**Pick the place.** One projects folder for everything, with a short path,
no spaces or accents, and outside any synced folder (OneDrive, Dropbox,
iCloud, Google Drive) and outside Desktop, Documents and Downloads:

| System | Projects folder |
| --- | --- |
| Windows | `C:\dev` |
| macOS, Linux | `~/dev` |

Create it if it doesn't exist. The store goes in a folder inside it, named
after the store in lower-case kebab-case (`C:\dev\livraria-da-ana`).

**Create the repository there.** The store lives in its own **private**
repository on Product's GitHub account, not in a fork of this one. From the
projects folder:

```sh
gh repo create <store-name> --private --clone --template MatheusLasserre/store-agent-baseline
```

That creates the repository and the `<store-name>` folder with these files
in it. If it fails, clone this repository into `<store-name>`, delete its
`.git` folder, run `git init -b main`, and create the private repository
from the result with `gh repo create <store-name> --private --source=. --push`.

**Move the session into it.** From now on all work happens in that folder.
If your tool can switch the session's working folder, do it. If it can't,
work there with absolute paths for the rest of this session.

**Make the next session start in the right place.** Sessions opened
elsewhere won't load `PERSONA.md` and `AGENTS.md` on their own, so this
matters. Give Product the exact clicks, for the tool they're using, to open
the store's folder as their project from now on (and pin or favorite it if
the tool allows), and have them do it once while you watch for it to work.
Write the folder's full path in `docs/setup.md` under Status.

Then, in the new repository:

- Replace this `README.md` with a short one for the store: its name, what it
  is, where the folder lives, and a line saying the rules are in
  `PERSONA.md` and `AGENTS.md`.
- Add `.gitattributes` with `* text=auto eol=lf`.
- Commit and push.

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
