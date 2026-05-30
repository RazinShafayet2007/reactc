# reactc — Codebase Reference Guidelines

> Which codebases to study, what to take from each, and how to apply it
> when implementing reactc's compiler and runtime.

---

## The Three References

| Codebase | Role in reactc | What you steal |
|---|---|---|
| **React** | Source of truth for hook semantics and component model | Hook contracts, scheduling, reconciler concepts |
| **SolidJS** | Primary compiler and fine-grained reactivity model | Signal graph, JSX transform, no-VDOM output |
| **Svelte** | Robustness reference for the compiler itself | Analysis passes, IR design, code generation patterns |

Study them in this order. React first — you need to understand what you are preserving. Solid second — it is your target output model. Svelte third — only when a compiler pass feels underspecified.

---

## React — What to Study and Why

React is not your compilation target. It is your **contract**. Every hook reactc supports must behave identically to React from the developer's perspective.

### Where to look

```
facebook/react (github.com/facebook/react)

packages/react/src/ReactHooks.js        ← public hook signatures
packages/react-reconciler/src/          ← reconciler internals
packages/react-reconciler/src/ReactFiberHooks.js  ← hook implementation
packages/scheduler/src/                 ← scheduler and priority lanes
packages/react/src/jsx/                 ← JSX runtime
```

### What to extract per feature

**useState**
Read `mountState` and `updateState` in `ReactFiberHooks.js`. Understand the work-in-progress queue. reactc must produce output where `useState`'s update semantics — batching, bailout on same value, functional updater form `setState(prev => ...)` — are fully preserved on the VDOM path and correctly bridged on the signal path.

**useEffect**
Read `mountEffect` and `commitHookEffectListMount`. The key contract: effects fire after paint, cleanup runs before the next effect, and the dependency array comparison is `Object.is`. Your compiled effect output must match this timing exactly.

**useMemo / useCallback**
Read `mountMemo`. The contract is simple — referential stability when deps don't change. Your auto-memoizer's job is to prove stability at compile time so the runtime check becomes unnecessary.

**useRef**
It is a box. Read `mountRef`. No transformation needed — passthrough in reactc.

**Scheduler / priority lanes**
Read `packages/scheduler/src/forks/Scheduler.js` and `ReactFiberLane.js`. Your scheduler integration in `@reactc/runtime` must map signal flush priority to React's `InputContinuous`, `Normal`, and `Idle` lanes correctly.

**JSX runtime**
Read `packages/react/src/jsx/ReactJSXElement.js`. Your VDOM backend emits calls to `react/jsx-runtime`. Understand what `jsx()`, `jsxs()`, and `Fragment` expect so generated code is valid.

### What NOT to take from React

Do not follow React's runtime hook tracking model — the linked list of hooks on the fiber. That is exactly what the compiler replaces. Read it to understand the contract, not to copy the implementation.

---

## SolidJS — What to Study and Why

Solid is your **output model** for the signal path. When reactc promotes a component to fine-grained reactivity, the compiled output should look like code a Solid developer would write — signals, memos, effects, and direct DOM bindings.

### Where to look

```
solidjs/solid (github.com/solidjs/solid)

packages/solid/src/reactive/signal.ts     ← signal core primitives
packages/solid/src/reactive/signal.ts     ← createSignal, createMemo, createEffect, batch, untrack
packages/solid/web/src/                   ← DOM rendering layer
babel-plugin-solid (github.com/ryansolid/dom-expressions/tree/main/packages/babel-plugin-jsx-dom-expressions)
                                          ← JSX transform to DOM bindings
dom-expressions (github.com/ryansolid/dom-expressions)
                                          ← runtime that binds signals to DOM nodes
```

### What to extract per feature

**Signal primitives**
`signal.ts` is the most important file. Read `createSignal`, `createMemo`, `createEffect`, `batch`, and `untrack` in full. Your `@reactc/runtime` Signal Core is a direct lightweight adaptation of this. Key insight: Solid uses a global tracking stack (`Owner` + `Listener`) — understand this before writing your own.

**The JSX transform**
Study `babel-plugin-jsx-dom-expressions` closely. This is how Solid compiles JSX into DOM operations. When a JSX expression reads a signal, it is wrapped in an effect that re-runs only that expression — not the component. reactc's signal backend JSX transform follows this exact pattern.

**DOM bindings**
Read `dom-expressions`. Understand how `insert()`, `setAttribute()`, and text node helpers work. Your signal backend codegen produces calls to a similar thin DOM layer.

**Memos**
`createMemo` in Solid is synchronous and cached — it only recomputes when its tracked signals change. Your `MemoNode` in the ReactiveIR maps directly to this. Read how Solid handles diamond dependencies (a memo depending on two signals that both update in one batch).

**Suspense in Solid**
Read `SuspenseContext` in Solid's source. reactc excludes Suspense from MVP, but understanding how Solid handles it tells you what you will need to add in v2 and why it is complex.

### What NOT to take from Solid

Do not copy Solid's owner tree and disposal model wholesale. Solid's cleanup system is designed for a world without React's reconciler. On reactc's VDOM path, React handles cleanup. Only apply Solid's disposal pattern to the signal path's `createEffect` cleanup.

---

## Svelte — What to Study and Why

Svelte is your **compiler robustness reference**. When an analysis pass or IR design decision feels unclear, Svelte has already solved a version of that problem. It is the most mature component compiler and its architecture is the closest thing to a production blueprint for what reactc's compiler needs to be.

### Where to look

```
sveltejs/svelte (github.com/sveltejs/svelte)

packages/svelte/src/compiler/phases/1-parse/     ← parser and AST
packages/svelte/src/compiler/phases/2-analyze/   ← analysis passes
packages/svelte/src/compiler/phases/3-transform/ ← IR and code generation
packages/svelte/src/compiler/utils/              ← compiler utilities

Heinrich's compiler explainer (Svelte blog):
svelte.dev/blog/svelte-5-runes                   ← runes = Svelte's signal model
```

### What to extract per feature

**Analysis phase structure**
Read `phases/2-analyze/`. Svelte runs multiple sequential analysis passes over the AST, each adding annotations to nodes. Your `@reactc/analyzer` should follow this pattern — separate passes for component detection, binding analysis, and mutation tracking — rather than trying to do everything in one walk.

**IR design**
Read `phases/3-transform/`. Svelte maintains a typed IR between analysis and code generation. Study how IR nodes carry source spans, how nodes reference each other by ID, and how the IR is validated before codegen starts. Apply this discipline to your `@reactc/ir`.

**Mutation tracking**
Svelte tracks which variables are mutated and which are read-only. This is directly applicable to reactc's signal promoter — a variable that is never reassigned after initialization is a strong candidate for a `createMemo`, not a `createSignal`.

**Code generation patterns**
Read `phases/3-transform/`. Svelte uses a `CodeBuilder` abstraction for generating JavaScript — a structured way to emit code that avoids string concatenation bugs. Your codegen should follow the same approach (reactc uses `@babel/generator` for this purpose, but the structural discipline is the same).

**Runes (Svelte 5)**
Read the runes RFC and `svelte.dev/blog/svelte-5-runes`. Runes are Svelte's answer to the same problem reactc solves — making reactivity explicit at the language level so the compiler can be precise. The design decisions Svelte made (and the ones they rejected) are directly relevant to reactc's signal promotion heuristics.

**Error reporting**
Read how Svelte produces compiler diagnostics with source locations. reactc should match this quality — every error and warning pinpoints the exact source span and explains what to do.

### What NOT to take from Svelte

Do not follow Svelte's component scoping model. Svelte compiles each component to a self-contained class with an `$set` method. reactc operates inside React's component model — the output is always a function component, never a class with lifecycle methods.

---

## How to Use These References When Prompting the LLM

When working on a specific reactc feature, structure your LLM prompt like this:

```
I am implementing [feature] in reactc.

React's contract for this feature (from ReactFiberHooks.js):
[paste the relevant React source or describe the contract]

SolidJS's output model for this feature (from signal.ts or dom-expressions):
[paste the relevant Solid source or describe the pattern]

My reactc types so far:
[paste your current types]

Implement [specific function/class] following React's contract
on the VDOM path and Solid's output model on the signal path.
```

This three-part context (React contract + Solid model + your types) produces significantly more accurate output than describing the feature in prose.

---

## Quick Reference — Feature to Codebase Mapping

| reactc feature | Study in React | Study in Solid | Study in Svelte |
|---|---|---|---|
| Signal Core runtime | — | `signal.ts` fully | — |
| `useState` promotion | `mountState`, `updateState` | `createSignal` | mutation tracking |
| `useEffect` transform | `mountEffect`, `commitHookEffectListMount` | `createEffect` | — |
| `useMemo` elision | `mountMemo` | `createMemo` | — |
| JSX transform (signal path) | `jsx-runtime` contracts | `babel-plugin-jsx-dom-expressions` | `phases/3-transform` |
| JSX transform (VDOM path) | `jsx-runtime` fully | — | — |
| Hook dep inference | `ReactFiberHooks` dep comparison | — | `phases/2-analyze` |
| IR design | — | — | `phases/3-transform` IR |
| Analysis pass structure | — | — | `phases/2-analyze` |
| Scheduler integration | `ReactFiberLane`, `Scheduler.js` | `batch` | — |
| Error reporting / diagnostics | — | — | Svelte diagnostic system |
| React compat layer | All of `ReactFiberHooks` | `createSignal` bridge | — |

---

## Guiding Principle

> Read React to know what you must not break.
> Read Solid to know what you are building toward.
> Read Svelte when you do not know how to build it.
