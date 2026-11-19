# Testing and checks

Tests verify what a customer or the owner would see, not implementation
details. All new logic has tests.

## Tools

- **Vitest** for unit and integration tests.
- **React Testing Library** with **user-event** and **jest-dom** for
  components.
- **MSW** to stand in for Mercado Pago, Resend and ViaCEP. Unhandled requests
  fail the test, so an accidental real call is visible.
- **Playwright** for a few end-to-end flows in a real browser.
- The **local Supabase stack** (`npx supabase start`, needs Docker) for
  anything that touches the database. Don't mock the database, and don't
  skip these tests when Docker is missing: getting Docker installed is part
  of the preflight ([setup.md](setup.md#preflight)).

## What to test

| Kind | Covers |
| --- | --- |
| Unit | Pure logic: totals, the cents/reais conversion, shipping rate choice, the webhook signature check, status mapping |
| Service | `service.server.ts` functions against local Supabase: creating an order, marking it paid twice, stock going down once, an amount mismatch being flagged |
| Component / page | Real routes with a memory history, real query hooks, server functions replaced at the module boundary |
| End to end | Browse → cart → checkout up to the redirect; the order page for a paid ebook order; admin sign-in and marking an order shipped |

The money path always has tests: an order is never paid without a valid
signature, a matching amount and an approved payment, and a repeated
notification changes nothing.

## Style

- Test files sit next to the code: `*.test.ts` / `*.test.tsx`.
- Query by role and label (`getByRole`, `getByLabelText`); `getByText` when
  the text is the behavior. Avoid `container.querySelector`.
- `userEvent` for clicks and typing; `fireEvent` only for events a person
  can't perform.
- Async UI: `findBy...`, not sleeps.
- Build test data with small factory functions (`createBook()`,
  `createOrder()`) that take overrides, one per table.
- No snapshot tests of large trees, no tests of Tailwind classes, no tests
  of TanStack, Radix or shadcn internals.
- Test data is synthetic. Never copy real customer data into a test or
  fixture.

## Keeping tests honest

- Fake responses for outside services are typed against the same schemas
  the production code parses with (`satisfies`), so a fake can't drift from
  what the code expects.
- Tests that depend on time use fixed dates or an injected clock, never the
  real current date.
- Test the failure and the recovery, not only the happy path: the error
  shown, the retry that then works, the last data staying visible when a
  refresh fails.
- In local development and tests, outside services point at fakes or at an
  address that refuses connections, so nothing can reach a real payment or
  send a real email by accident. Local and test environments never use
  production keys, and production values are never copied into local files.

## Before a push to `main`

Run only what the change needs, narrowest first:

| Changed | Run |
| --- | --- |
| Anything | `npm run check` (Biome lint and format) and `npm run build` (includes the type check) |
| Logic or components | `npx vitest run <the affected files>` |
| Migrations, shared helpers, `src/server/` | the whole Vitest suite |
| A route or a flow covered end to end | `npm run test:e2e` |
| Any React code | `npx react-doctor@latest . --verbose` at 100/100 |

List each command you ran and its result in the commit message (Product
gets the plain-language summary, not the commands). If a test fails
only when the machine is busy, rerun that file alone before treating it as
real. Never skip, delete or weaken a failing test to get a push through:
fix the code, or fix the test and say why it was wrong.

After a deploy, open the live store and check the page you changed. A
checkout change is also checked with a Mercado Pago test payment before the
owner is told it's done.
