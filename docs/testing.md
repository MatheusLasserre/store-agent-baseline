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

## The money suite

Payments get more testing than everything else in the store put together.
`npm run test:money` runs it, and it runs in full for every change to
checkout, the webhook, order statuses, stock or downloads, plus
`npm run pay:sim -- all` ([payments.md](payments.md#dry-run)).

- **One test per invariant, at least.** Each numbered invariant in
  [payments.md](payments.md#what-must-never-happen) has tests named after
  it (`invariant 6: parallel duplicate notifications apply once`), so a
  missing one is visible.
- **Tests attack, not just confirm.** For each rule, the test tries to
  break it: pay with a forged signature, a smaller amount, another store's
  payment, a replayed notification, a test payment in production mode; set
  an order to paid from every code path that isn't the webhook; download
  after a refund.
- **Assert on the database, not on return values.** After each scenario,
  read back the order, its items, the stock, the payment record and the
  emails queued, and check all of them, including that nothing else
  changed.
- **Concurrency is tested for real:** fire the same notification many times
  in parallel against the local database and check there is one payment
  record, one stock change and one email.
- **Every transition is tested both ways:** each allowed status change
  works, and each change not in the table is rejected.
- **Failure in the middle:** make the database, Mercado Pago's API and the
  email service fail at each step of the webhook, and check the order ends
  either unchanged (and retried) or fully applied, never half.
- **A test must be able to fail.** When you add a money test, break the
  rule it guards once, on purpose, see the test go red, and restore it. A
  test that never failed proves nothing.
- **A bug becomes a scenario.** Any payment problem found later, in
  testing or in production, is first reproduced as a failing test and a
  `pay:sim` scenario, then fixed.

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
| Checkout, webhook, order status, stock or downloads | `npm run test:money` and `npm run pay:sim -- all`, in full, always |
| Any React code | `npx react-doctor@latest . --verbose` at 100/100 |

List each command you ran and its result in the commit message (Product
gets the plain-language summary, not the commands). If a test fails
only when the machine is busy, rerun that file alone before treating it as
real. Never skip, delete or weaken a failing test to get a push through:
fix the code, or fix the test and say why it was wrong.

After a deploy, open the live store and check the page you changed. A money
change also goes through the test-mode and, when it applies, the real
low-value purchase described in [payments.md](payments.md#dry-run) before
Product is told it's done.
