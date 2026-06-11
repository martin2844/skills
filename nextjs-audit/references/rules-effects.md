# Rules — useEffect Anti-Patterns

From the react.dev "You Might Not Need an Effect" catalog. Effects
are an escape hatch for synchronizing with external systems; they
are *not* the way to derive state, fetch data, transform props, or
sequence work. Each anti-pattern below has a non-effect solution.

Legit effects (subscriptions to non-React widgets, focus management,
analytics-on-mount, listening to `window`/`document` events) pass
silently. The catalog only flags effects that have a simpler home.

---

## E1. Derived state in an effect

**Question:** Does this effect compute a value from props/state and
write it to a separate state variable?

**Apply to:**
```tsx
const [filtered, setFiltered] = useState([]);
useEffect(() => { setFiltered(items.filter(p)) }, [items]);
```

**Prove it by:** read the effect. Its body is a synchronous
calculation. The state it sets is fully determined by the
dependencies. Triggers extra renders (one for the input change,
one for the effect-driven state set).

**Good finding:** "`components/Catalog.tsx:18` derives `filtered`
from `items` via `useEffect`. Move it to a plain const:
`const filtered = items.filter(p)`. If `p` is expensive, wrap with
`useMemo(() => items.filter(p), [items])`. The effect causes an
extra render and lets `items` and `filtered` briefly disagree."

**Don't report:** state writes from effects whose source is
external (a fetch, a subscription).

---

## E2. State sync on prop change (resetting state when props change)

**Question:** Does this effect reset internal state whenever a
prop changes?

**Apply to:**
```tsx
useEffect(() => { setSelected(null) }, [userId]);
```

**Prove it by:** the effect is reacting to a prop change to reset
identity-tied state. React provides two cleaner mechanisms:
**a `key` prop on the parent** (full remount, all state resets), or
**reading the prop in render and comparing** (rare, advanced —
see react.dev).

**Good finding:** "`components/Profile.tsx:24` does
`useEffect(() => { setSelected(null) }, [userId])`. Pass `key={userId}`
to the wrapping component so React resets state on identity change
— removes the effect and the brief inconsistent render."

**Don't report:** effects that sync to an *external* system on
prop change (e.g. opening a new WebSocket per `roomId`) — that's
legitimate.

---

## E3. Event-handler logic inside an effect

**Question:** Does this effect react to an event (click, submit)
rather than a state change?

**Apply to:** effects that look like "when X changes, send a
network request" where X is set in response to a user action.

**Prove it by:** find the setter for the trigger state. Is it
called from an event handler? If yes, put the logic in the handler
directly. Effects fire on every render where deps changed —
including unrelated changes that happen to land alongside.

**Good finding:** "`components/Checkout.tsx:42` sets
`setSubmitted(true)` in the click handler, and a separate
`useEffect(() => { if (submitted) sendPayment() }, [submitted])`
fires the request. Move the `sendPayment()` call into the click
handler; drop the effect and the `submitted` state."

**Don't report:** effects that *also* run on mount or external
state changes — only the click-driven variant is the anti-pattern.

---

## E4. Fetching data in an effect that should be a server fetch

**Question:** Does this `"use client"` component fetch data in
`useEffect` that the server could have fetched and passed as a
prop?

**Apply to:** `useEffect(() => fetch('/api/...'))` in client
components where the data is needed for initial render.

**Prove it by:** check where the fetch URL points. An internal
route handler the server component could call directly? An
internal data source? Then the fetch should be in the parent
server component (or via a server action called on mount via
`Suspense + use`). Effect-fetching causes a render flash (empty →
loading → data) and ships an extra round trip.

**Good finding:** "`components/Dashboard.tsx:18` does
`useEffect(() => { fetch('/api/stats').then(setStats) }, [])`.
`/api/stats` reads `db('stats').first()` — there's no reason this
isn't fetched in the parent server component and passed as a prop.
Move the data fetch up; the dashboard renders with data on first
paint."

**Don't report:** effects fetching client-only data (browser-side
geolocation, user input echo, WebSocket-driven streams), or data
that legitimately depends on client state.

---

## E5. Chained effects (effect-driven state-machine)

**Question:** Are there multiple effects in this component where
one effect's output triggers the next?

**Apply to:** components with 3+ effects whose deps form a chain
(`a → effect → b → effect → c`).

**Prove it by:** map the dep arrows. If effect 2's deps include
state set by effect 1, you have a chain. Each link is a render +
commit cycle, and the chain is fragile (any intermediate render
that happens to set state restarts the chain partway).

**Good finding:** "`hooks/usePlateLookup.ts` has five effects
forming `plate → effect → fastResult → effect → liveResult →
effect → polledResult → effect → uiState`. Each step is a render
boundary. Collapse into a `useReducer` with one effect per external
system (the fetch, the WebSocket), or into a state-machine library."

**Don't report:** independent effects with overlapping deps that
aren't actually chained (read the deps carefully).

---

## E6. Initializing state from props in an effect

**Question:** Does the effect `setState(props.x)` on mount?

**Apply to:**
```tsx
useEffect(() => { setX(props.x) }, []);
// or
useEffect(() => { setX(props.x) }, [props.x]);
```

**Prove it by:** the first form is `useState(props.x)` written as
an effect, and ignores future prop changes. The second form is
"sync state with prop" — covered by E2. The right answer is almost
always `useState(props.x)` (initialize once) or derive from props
in render.

**Good finding:** "`components/Editor.tsx:22` does
`const [value, setValue] = useState(''); useEffect(() => setValue(initial), [])`.
Replace with `useState(initial)` — drops the effect and the
empty-string flash on first render."

**Don't report:** state initialization from async sources (legit).

---

## E7. Subscribing to an external store in an effect (use a hook)

**Question:** Does the effect manually subscribe to a store with
`store.subscribe(cb)`?

**Apply to:** Zustand / Redux / custom store subscriptions in
effects.

**Prove it by:** look for `useSyncExternalStore`. React's official
mechanism for subscribing to external stores is
`useSyncExternalStore` (or the store library's `use*` hook, which
is built on it). Hand-rolled subscriptions in effects miss the
tear-free guarantees and the SSR snapshot.

**Good finding:** "`components/Banner.tsx:14` does
`useEffect(() => store.subscribe(() => setState(store.get())), [])`.
Use `useSyncExternalStore(store.subscribe, store.get)` — fixes the
SSR mismatch and the tear during concurrent renders."

**Don't report:** legit external-event subscriptions
(`window.addEventListener`, `MutationObserver`) — those are
correctly placed in effects.

---

## E8. `useEffect` for one-time setup on mount (run during render
helpers exist)

**Question:** Is the effect setting up something synchronous and
non-DOM (logger init, analytics identify, store population)
purely on mount?

**Apply to:** `useEffect(() => { doOnce() }, [])` patterns.

**Prove it by:** check what `doOnce` does. If it's a synchronous,
idempotent setup of a non-React-tree resource, a top-level
`if (!initialized) initialize(); initialized = true;` next to the
module — or in `instrumentation.ts` for server-side — runs once
per process and avoids the post-paint delay.

**Good finding:** "`app/providers.tsx:18` does
`useEffect(() => { analytics.init() }, [])`. The init now runs
*after* first paint, missing the page-view event for the initial
render. Either initialize at the module top level (analytics
libraries handle this themselves), or in `instrumentation.ts`
client init."

**Don't report:** mount effects that *must* run client-side after
hydration (DOM measurement, browser-only APIs) — those are
correct.

---

## E9. `useEffect` to call a parent setter

**Question:** Does this effect call a parent's setter (`onChange`,
`onValueChange`) to "report back" derived state?

**Apply to:**
```tsx
useEffect(() => { onChange(value) }, [value]);
```

**Prove it by:** find where `value` is set. If it's a controlled
input, the parent already owns it. If `value` is computed from
props, lift the computation to the parent and skip the round trip.

**Good finding:** "`components/Form.tsx:24` has
`useEffect(() => onChange(combined), [combined])` where `combined`
is derived from `a` and `b` props. Move the derivation to the
parent — `combined` should be computed where `a` and `b` live, not
echoed back from a child."

**Don't report:** legit `onChange` calls inside event handlers
(those aren't effects).

---

## Cross-cutting

- useEffect findings are typically Should-fix or Nit; only flag as
  Blocker if the anti-pattern causes a visible bug (wrong data,
  flashing UI, broken hydration).
- Don't pad. A codebase using `useEffect` correctly for
  subscriptions and DOM measurement should pass this category
  clean.
- When flagging, link to the relevant react.dev section
  ("react.dev: You Might Not Need an Effect — `<section>`") so the
  author can read the canonical explanation.
