# Structure and code patterns

## Folders

```
src/
├── routes/                 File-based routes (pages and server routes)
│   ├── __root.tsx          Providers, document shell, not-found page
│   ├── _store/             Pathless layout: the public storefront
│   │   ├── route.tsx       Storefront layout with <Outlet />
│   │   ├── index.tsx       /            the collection
│   │   ├── livro.$slug.tsx /livro/:slug one book
│   │   ├── carrinho.tsx    /carrinho
│   │   ├── checkout.tsx    /checkout
│   │   └── pedido.$token.tsx /pedido/:token  order status and downloads
│   ├── admin/              Path layout: /admin/*, guarded by admin sign-in
│   │   ├── route.tsx       Guard in beforeLoad + admin layout
│   │   └── ...
│   └── api/                Server routes called by other services
│       └── mercado-pago.webhook.ts
├── modules/<name>/         One folder per feature (see Modules)
├── components/ui/          shadcn/ui primitives
├── components/layout/      PageHeader, StoreLayout, AdminLayout
├── lib/                    Small shared helpers (money, dates, errors, form)
└── server/                 Server-only plumbing: env.server.ts, supabase.server.ts
supabase/migrations/        SQL migrations
docs/                       These docs
```

Customer-facing URLs are in Portuguese; folder and file names that aren't
part of a URL are in English.

## Routes

Routing is file-based only.

| Pattern | Meaning |
| --- | --- |
| `__root.tsx` | Root route wrapping the whole app |
| `_folder/` | Pathless layout: wraps children, adds nothing to the URL |
| `folder/` | Path layout: adds its name to the URL |
| `route.tsx` | A folder's layout; renders `<Outlet />` |
| `index.tsx` | A folder's index page |
| `$param` | Dynamic segment |

- Layouts come from `route.tsx` with `<Outlet />`. Never wrap a page in a
  layout component by hand.
- Guards go in the layout's `beforeLoad` (the admin check), not in each page.
- Links use the real URL (`/carrinho`), never a pathless folder's name.
- The route tree file is generated. Never edit it; run the dev server or the
  build to regenerate it after adding a route.
- Route files stay declarative: `createFileRoute`, `validateSearch`, the
  loader, the page's layout. Logic lives in the module.
- Filters, tabs, pagination and anything a person might share or reload live
  in URL search params validated with Zod (`validateSearch`), not in
  `useState`. Update them with `navigate({ search: (prev) => ..., replace: true })`.

## Modules

Each feature is a folder under `src/modules/` (`catalog`, `cart`, `checkout`,
`order`, `shipping`, `admin`, ...), with the same files:

| File | Holds |
| --- | --- |
| `schemas.ts` | Zod schemas and the types inferred from them; mapping from database rows to what the UI gets. Safe to import anywhere. |
| `service.server.ts` | Business logic. Plain functions that receive a Supabase client and typed input. No request or cookie handling. |
| `functions.ts` | Server functions. Thin: validate input, check who's calling, call the service, return its result. |
| `queries.ts` | Query keys, `queryOptions` and mutation hooks wrapping the server functions. |
| `components/` | The feature's components. |
| `use<Feature>PageController.ts` | Page orchestration, when a page needs it (below). |

Rules:

- Components never call a server function or `fetch` directly. They use the
  hooks and `queryOptions` from `queries.ts`. Loaders prefetch with the same
  `queryOptions`.
- Query keys are built in one place per module, and every mutation
  invalidates or updates the queries it affects.
- Business logic belongs in `service.server.ts`, so it can be tested without
  the framework.
- A module imports another module only through its `schemas.ts`,
  `queries.ts` or service functions, never its components' internals.

### Adding a feature

1. Migration and types, if the data changes ([database.md](database.md)).
2. `schemas.ts`, then `service.server.ts` with tests.
3. `functions.ts`, then `queries.ts`.
4. Components and the route file.
5. Update [store.md](store.md) if what the store does changed.

### Page controllers

When a page accumulates query hooks, dialog state, search-param updates and
handlers, move that into `use<Feature>PageController(search)`. The route
passes `Route.useSearch()` in; the hook returns view-ready data, loading
flags and stable handlers. The route owns typing and search validation, the
hook owns behavior.

## Forms

- TanStack Form with Zod validators, through the app-wide helpers in
  `src/lib/form/` (`useAppForm`, bound field components, `withForm` for
  reusable sections). Don't hand-roll field state.
- A dialog that collects values and then submits is a form. A control that
  acts immediately (a quantity stepper, a filter) is not.
- The same Zod schema validates the form in the browser and the server
  function's input.
- Mutation failures are shown through `getErrorMessage`.

## UI

- Tailwind utilities first, shadcn/ui components second, custom CSS only when
  utilities can't express it. Add a missing shadcn component with
  `npx shadcn@latest add <component>` instead of building one.
- `PageHeader` gives every page its title and actions.
- Extract complex inline JSX (header actions, form sections, repeated rows)
  into named components.
- The storefront is used mostly on phones: build at 390 px wide first. Give
  flex and grid children `min-w-0`, let action rows wrap, keep tap targets
  reachable.
- Accessibility is part of done: real buttons and links, labels on every
  field, visible focus, alt text on covers.
- All copy is pt-BR and lives next to the component that shows it.

## React patterns

`npx react-doctor@latest . --verbose` must finish at 100/100 before a web
change is handed off. Treat a finding as a hypothesis: fix it, or when it's
proven wrong for this code, suppress that one line with a reason
(`// react-doctor-disable-next-line react-doctor/<rule> -- reason`). No broad
config disables.

- **Derived state:** don't copy props or query data into state with
  `useEffect`. Derive during render.
- **Effects that follow a handler:** if an effect only reacts to state a
  handler just set, do the work in the handler.
- **Stable shells:** keep dialogs, drawers and sheets mounted while their
  data loads, with a skeleton inside. Don't swap the whole shell or key it by
  `open`.
- **Heavy libraries:** load them with `React.lazy` and a dynamic `import()`,
  with a fixed-height skeleton fallback.
- **Refs:** never read or write `ref.current` during render.
