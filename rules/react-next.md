# React & Next.js Rules

These rules govern all React and Next.js code generated in this project. They assume **React 19+** and **Next.js 14+ with the App Router**. Follow them precisely and without exception unless the user explicitly overrides a specific rule.

`rules/typescript.md` applies in full to every `.ts`/`.tsx` file. This document adds React- and Next.js-specific rules and, where noted, narrow exceptions to it.

---

## 1. Server Components Are the Default

- Every component in `app/` is a **Server Component** unless it declares `"use client"`. Keep it that way. Do not add `"use client"` to a file "just in case".
- Add `"use client"` only when the component needs one of: `useState`/`useReducer`, `useEffect`/lifecycle, event handlers, browser-only APIs (`window`, `localStorage`, `IntersectionObserver`), or a client-only third-party library.
- `"use client"` marks a **bundle boundary**: every module imported from that file, transitively, ships to the browser. Push the directive as far down the tree (as close to the leaves) as possible.
- Never place `"use client"` in a root `layout.tsx`, a `page.tsx` that only composes children, or a shared `utils` barrel file.

```tsx
// ❌ whole page becomes a Client Component to get one interactive button
"use client";
export default function ProductPage() { /* fetch + render + a button */ }

// ✅ server page, client leaf
// app/products/[id]/page.tsx  (Server Component)
export default async function ProductPage({ params }: ProductPageProps) {
    const { id } = await params;
    const product = await getProduct(id);
    
    return (
        <>
            <ProductDetails product={product} />
            <AddToCartButton productId={product.id} />
        </>
    );
}
```
- Server Components may render Client Components. Client Components may **not** import Server Components — pass server-rendered markup down as `children` or props instead.
- Props crossing the server → client boundary must be serialisable: no functions (other than Server Actions), no class instances, no `Date` methods relied upon after transfer, no `Symbol`.

---

## 2. File and Component Structure

- One component concept per file. Co-locate a component with its sub-components only when they are never used elsewhere.
- Name components in `PascalCase`; name files after their default export (`ProductCard.tsx`) or use the Next.js reserved name (`page.tsx`, `layout.tsx`).
- Next.js reserved files (`page`, `layout`, `template`, `loading`, `error`, `not-found`, `default`, `route`, `middleware`, `*.config.ts`) **must** use a default export. This is the only sanctioned exception to the named-export rule in `rules/typescript.md` §17. All other components use named exports.
- Declare components as function declarations, not arrow-function consts assigned to a variable, so stack traces and React DevTools names are meaningful.
- Use folder colocation: private implementation files live beside the route and are excluded from routing by prefixing the folder with `_` (`app/dashboard/_components/`).

---

## 3. Typing Components and Props

- Import types and hooks directly from `"react"` (e.g. `import type { ReactNode, ComponentPropsWithRef } from "react"`); do not use `React.<Something>` namespace access.
- Type props with a named `interface` (per `rules/typescript.md` §5). Do not use `FC` (or `React.FC`) — it adds nothing and historically mistyped `children`.
- Type `children` as `ReactNode`. Type render props and slots explicitly.
- Extend native element props when building a wrapper, so consumers keep every standard attribute.

```tsx
interface ButtonProps extends ComponentPropsWithRef<"button"> {
  readonly variant?: "primary" | "secondary";
}

export function Button({ variant = "primary", ...props }: ButtonProps) {
  return <button data-variant={variant} {...props} />;
}
```

- Prefer `ComponentPropsWithRef<"tag">` over the deprecated `ComponentPropsWithoutRef` + `forwardRef` combination (see §7).
- Props are **readonly**. Never mutate a prop or an object reached through a prop.
- Async Server Component route props are Promises in Next.js 15+. Type and `await` them:

```tsx
interface ProductPageProps {
readonly params: Promise<{ readonly id: string }>;
readonly searchParams: Promise<Record<string, string | string[] | undefined>>;
}
```

---

## 4. Rules of React (Non-Negotiable)

- **Components and hooks must be pure.** Given the same props/state/context they render the same output. No mutation of module-level variables, no `Math.random()` or `new Date()` during render for values that must be stable, no DOM reads/writes in the render body.
- **Never mutate values you did not create during this render.** Local objects created inside render may be mutated before they are returned; props, state, context, and hook results may not.
- **Call hooks only at the top level** of a component or another hook. Never inside conditions, loops, `try`/`catch`, or after an early `return`.
- **Call hooks only from React functions** — components or custom hooks whose names begin with `use`.
- Never call a component as a plain function (`Header()`); always render it as JSX (`<Header />`).
- Do not read or write refs during render — only in effects or event handlers.
- Treat React's rules as enforceable: `eslint-plugin-react-hooks` with `react-hooks/rules-of-hooks` and `react-hooks/exhaustive-deps` must be enabled and errors treated as blocking.

---

## 5. State

- Keep state **minimal**. Derive everything derivable during render instead of storing it in a second state variable or syncing it with an effect.

```tsx
// ❌ redundant state kept in sync
const [items, setItems] = useState<Item[]>([]);
const [count, setCount] = useState(0);
```
```tsx
// ✅ derived
const [items, setItems] = useState<Item[]>([]);
const count = items.length;
```

- Update state **immutably**. Use spread, `map`, `filter`, `toSorted`/`toSpliced`, or an immutability helper. Never `push`, `splice`, or assign into existing state.
- Use the **updater form** (`setCount(c => c + 1)`) whenever the next value depends on the previous one.
- Group state that always changes together into one object or `useReducer`. Reach for `useReducer` when there are more than ~3 interdependent transitions.
- Avoid contradictory state (`isLoading` + `isError` + `data` booleans). Model it as a discriminated union (`rules/typescript.md` §8).
- Lift state to the closest common ancestor — no higher. Do not hoist state into a global store to avoid passing one prop.
- Reset child state by changing its `key`, not by an effect that watches a prop.
- Do not duplicate props into state (`useState(props.value)`) unless you deliberately want an uncontrolled initial value; name it `initialValue` if so.

---

## 6. Effects

- An Effect is an escape hatch for **synchronising with an external system** (browser APIs, non-React widgets, subscriptions, analytics). If there is no external system, you probably do not need an Effect.
- Do **not** use an Effect to: transform data for rendering, reset state on prop change, handle user events, or fetch data in a Server-Component-capable tree.

```tsx
// ❌ event logic in an Effect
useEffect(() => {
  if (submitted) void postOrder(order);
}, [submitted, order]);

// ✅ event logic in the handler
function handleSubmit() {
  void postOrder(order);
}
```

- Every Effect that creates a subscription, timer, listener, or request must return a **cleanup** function; requests must be cancelled or ignored via `AbortController` or a stale flag.
- Dependency arrays are derived from the code, not tuned by hand. Never suppress `react-hooks/exhaustive-deps`; remove the dependency by moving the value inside the Effect, wrapping it, or extracting an Effect Event.
- Effects must be resilient to running twice (React Strict Mode double-invokes them in development). If double-invocation breaks the code, the Effect is wrong.
- For client-side data fetching in a Client Component, use a framework-aware library (React Query / SWR) or `use()` with a cached promise created outside render — not a bare `useEffect` + `fetch`.

---

## 7. Refs

- `ref` is an ordinary prop in React 19. **Do not use `forwardRef`** in new code; it is deprecated.

```tsx
// ✅ React 19
interface InputProps extends ComponentPropsWithRef<"input"> {}
export function Input({ ref, ...props }: InputProps) {
  return <input ref={ref} {...props} />;
}
```

- Use refs for values that do not drive rendering (timeout ids, DOM nodes, previous values). Never render a ref's value.
- Prefer declarative props over imperative DOM manipulation. Use `useImperativeHandle` only to expose a deliberately narrow imperative API.
- Ref callbacks may return a cleanup function — use it instead of pairing with an Effect for observers.

---

## 8. Lists, Keys, and Conditional Rendering

- Every list item needs a stable, content-derived `key`. Use the entity's id. **Never** use the array index for lists that can reorder, filter, or receive insertions. Never use `Math.random()` or `crypto.randomUUID()` as keys (or generate keys during render)
- Extract a branch into its own component when either arm exceeds a few lines. Do not nest ternaries in JSX.
- Do not build JSX inside a `for` loop with manual `push`; use `map`.

---

## 9. Composition over Configuration

- Prefer `children` and slot props to boolean flags that switch layout. A component with more than ~3 boolean props usually wants to be split.
- Pass server-rendered content into Client Components as `children` — this keeps the content on the server even though the wrapper is a client boundary.
- Avoid prop drilling deeper than two levels; use composition first, Context second.
- Use Context for genuinely ambient, rarely changing values (theme, locale, current user). Do not use Context as a substitute for a state manager on frequently changing values — every consumer re-renders.
- Context providers that hold state must be Client Components; keep the provider file small and place it as low in the tree as the data allows.

---

## 10. Performance

- **Do not micro-optimise by default.** Write clear code first. Fix the render tree (state placement, composition) before reaching for memoisation.
- If the React Compiler is enabled, do not hand-write `useMemo`, `useCallback`, or `memo` — the compiler handles it and manual memoisation adds noise.
- Without the compiler, use `useMemo` only for provably expensive computations or referential stability required by a dependency array; use `memo` only on components that re-render often with identical props.
- Never call a hook conditionally to "skip" work; memoise inside instead.
- Use `useTransition` for non-urgent state updates that would otherwise block typing or clicking, and `useDeferredValue` for expensive derived views of fast-changing input.
- Keep `key`s stable to avoid accidental remounts, which are far more expensive than re-renders.
- Use `next/image` for all raster images (automatic sizing, lazy loading, modern formats) and `next/font` for fonts (self-hosted, no layout shift). Never use a raw `<img>` for content images or a `<link>` to a font CDN.
- Use `next/dynamic` for heavy client-only widgets (editors, charts, maps) that are not needed on first paint.

---

## 11. Data Fetching (Next.js)

- Fetch data in Server Components, as close as possible to the component that needs it. Do not lift all fetching into the page and drill props down.
- Never create an internal Route Handler just so a Server Component can call it. Query the data source directly.
- Parallelise independent requests with `Promise.all`; only await sequentially when one request genuinely depends on another.

```tsx
const [artist, albums] = await Promise.all([getArtist(id), getAlbums(id)]);
```

- Deduplicate per-request work with React's `cache()` for non-`fetch` data sources (database/ORM calls).
- Wrap slow, independent sections in `<Suspense>` with a meaningful fallback so the shell streams immediately. Prefer granular `<Suspense>` boundaries over a single page-level `loading.tsx` for mixed-speed content.
- Make caching decisions **explicit**. State the intent for every request — cached, revalidated on an interval, or dynamic — rather than relying on defaults that vary by Next.js version.
- Prefer tag-based revalidation (`revalidateTag`) over `revalidatePath` so unrelated caches are not invalidated.
- `cookies()`, `headers()`, `params`, and `searchParams` are async and force dynamic rendering. Read them only in the component that needs them, never in a root layout.
- Never expose secrets to the client. Only `NEXT_PUBLIC_*` env vars reach the browser; read all other secrets in Server Components, Server Actions, or Route Handlers. Consider `server-only` on modules that must never be bundled for the client.

---

## 12. Mutations and Server Actions

- Use **Server Actions** for mutations instead of hand-rolled Route Handlers plus `fetch`. Mark them with `"use server"` and keep them in a dedicated `actions.ts` module.
- Every Server Action is a public HTTP endpoint. **Always** authenticate, authorise, and validate input inside the action — never trust the caller.

```ts
"use server";

export async function updateProfile(formData: FormData): Promise<ActionResult> {
  const session = await requireSession();          // authn + authz
  const input = profileSchema.parse(toObject(formData)); // validation
  await profiles.update(session.memberId, input);
  revalidateTag(`profile:${session.memberId}`);
  return { status: "ok" };
}
```

- Return a typed, serialisable result (a discriminated union) rather than throwing for expected failures. Reserve thrown errors for genuine faults.
- Wire actions to forms with the `action` prop so they work before JavaScript hydrates. Use `useActionState` for pending/error state and `useFormStatus` in nested submit buttons instead of prop-drilling a `pending` flag.
- Use `useOptimistic` for instant feedback on mutations; React reconciles or rolls back automatically. Do not hand-roll optimistic state with `useState` + `try/catch`.
- Revalidate (`revalidateTag`/`revalidatePath`) or `redirect` after a successful mutation. Never mutate and leave stale data on screen.
- Use Route Handlers (`app/api/*/route.ts`) only for webhooks, third-party callbacks, and public APIs consumed outside this app.

---

## 13. Forms and Validation

- Prefer uncontrolled forms with `FormData` and a Server Action. Reach for controlled inputs only when the value drives other UI during typing.
- Validate with a schema (Zod or equivalent) and derive the TypeScript type from the schema — never maintain both by hand.
- Validate on the server **always**; client-side validation is a UX affordance, not a guarantee.
- Return field-level errors from the action and render them next to their inputs, associated via `aria-describedby`.
- Disable the submit control while pending using `useFormStatus`, and make the disabled state visible.

---

## 14. Error Handling and Loading States

- Add `error.tsx` (Client Component) at each route segment that can fail meaningfully, and `global-error.tsx` at the root. Provide a `reset()` affordance.
- Use `not-found.tsx` with `notFound()` for missing resources; use `redirect()` for moved or gated routes. Do not call these inside `try`/`catch` blocks that swallow their control-flow errors.
- `loading.tsx` and `<Suspense>` fallbacks should be skeletons that match the final layout — never a bare centred spinner for full-page content.
- Never render a raw error message or stack trace to the user. Log the detail server-side and show an actionable message.
- Error boundaries do not catch errors in event handlers or async callbacks — handle those explicitly.

---

## 15. Routing and Navigation

- Use `<Link>` for internal navigation (prefetching, client-side transitions). Use `useRouter().push()` only for programmatic navigation after an event.
- Use route groups `(marketing)` for organisation without URL segments, and `_private` folders for non-routable colocation.
- Use `layout.tsx` for shared shell that must preserve state across navigation; use `template.tsx` only when you need a fresh instance per navigation.
- Read the current route in Client Components with `usePathname`, `useSearchParams`, and `useParams` — never with `headers()`.
- Components using `useSearchParams` must sit inside a `<Suspense>` boundary, or the whole route opts out of static rendering.
- Store shareable UI state (filters, tabs, pagination) in the URL rather than in component state.
- Type `generateMetadata` and `generateStaticParams` return values with Next.js's exported types (`Metadata`, `Route` helpers) rather than ad-hoc shapes.

---

## 16. Styling

- Pick one styling strategy per project and stay with it. Do not mix CSS Modules, Tailwind, and CSS-in-JS in the same codebase without an explicit decision.
- Prefer zero-runtime styling (CSS Modules, Tailwind) — runtime CSS-in-JS libraries generally require a client boundary and hurt Server Component rendering.
- Compose conditional class names with a small helper (`clsx`/`cn`); never build class strings with nested ternaries inline in JSX.
- Do not use inline `style` objects for anything static — they defeat caching and cannot express pseudo-states or media queries.

---

## 17. Accessibility

- Use semantic elements: `<button>` for actions, `<a>`/`<Link>` for navigation, `<form>` for submissions, real heading levels in order.
- Never attach `onClick` to a `<div>` or `<span>`. If a custom control is unavoidable, supply `role`, `tabIndex`, and keyboard handlers.
- Every input has an associated `<label>`; every non-decorative image has meaningful `alt`; decorative images get `alt=""`.
- Manage focus on route change, dialog open/close, and after destructive actions. Never remove focus outlines without an equivalent visible indicator.
- Announce async state changes via `aria-live` regions or `aria-busy`, not by visual spinner alone.

---

## 18. Testing

- Test behaviour through the DOM with React Testing Library. Query by role, label, and text — never by class name, test id (as a first choice), or internal state.
- Drive interactions with `@testing-library/user-event`, not synthetic `fireEvent` clicks, so keyboard and pointer semantics are exercised.
- Never assert on component internals, hook call counts, or implementation details. Never snapshot-test large trees.
- Test Server Components by asserting on their rendered output or by unit-testing the data functions they call; test Server Actions as plain async functions with mocked dependencies.
- Mock at the network boundary (MSW) rather than stubbing `fetch` per test.
- Follow `/tdd` for non-trivial component logic: failing test, minimal implementation, refactor.

---

## 19. Things to Never Do

- Never use `dangerouslySetInnerHTML` with unsanitised input.
- Never access `window`, `document`, or `localStorage` during render or in a Server Component — guard with an Effect or `typeof window !== "undefined"` inside a client-only path.
- Never mutate props, state, context values, or the DOM outside refs/effects.
- Never use array indices as keys for dynamic lists.
- Never use an Effect to synchronise state that can be derived.
- Never leak secrets through `NEXT_PUBLIC_*` or through props passed to Client Components.
- Never add `"use client"` to a shared utility, type, or barrel module.
- Never use `React.<Something>` namespace access; import types, hooks, and utilities directly from `"react"` (e.g. `import { useState, type ReactNode } from "react"`).
- Never suppress `react-hooks/exhaustive-deps` or `react-hooks/rules-of-hooks`.
