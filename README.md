# reactc

A compiler for React that eliminates runtime overhead through static analysis and fine-grained signal-based reactivity. Write standard React — get faster output.

```bash
npx reactc compile src/
```

---

## How it works

React figures out what changed at runtime. reactc figures it out at build time.

The compiler analyzes your components, infers which state is fine-grained enough to bypass the VDOM entirely, inserts memoization where it is provably needed, and emits two kinds of output depending on what it finds:

- **Signal path** — state is compiled to signals that update individual DOM nodes directly. No reconciler. No re-render.
- **VDOM path** — components stay in React's rendering model but with memoization pre-inserted. `React.memo`, `useMemo`, and `useCallback` are added automatically where stable.

You do not choose which path a component takes. The compiler decides based on static analysis.

---

## Status

> Early development. Not ready for production use.

| Phase | Status |
|---|---|
| Lexer / tokenizer | 🔲 Planned |
| AST builder | 🔲 Planned |
| Component analyzer | 🔲 Planned |
| Hook graph builder | 🔲 Planned |
| Type resolver | 🔲 Planned |
| IR generation | 🔲 Planned |
| Auto-memoizer | 🔲 Planned |
| Signal promoter | 🔲 Planned |
| Tree shaker | 🔲 Planned |
| Signal backend | 🔲 Planned |
| VDOM backend | 🔲 Planned |
| Signal Core runtime | 🔲 Planned |
| React compat layer | 🔲 Planned |
| Scheduler integration | 🔲 Planned |
| CLI | 🔲 Planned |

---

## MVP Features

### Supported hooks

| Hook | Treatment |
|---|---|
| `useState` | Promoted to a signal when state is a primitive type and conditions allow. Otherwise kept as VDOM state with auto-memoization. |
| `useEffect` | Compiled with static dependency tracking. Dependencies are inferred from the AST — stale dep arrays are corrected at compile time. |
| `useMemo` | Auto-elided when the compiler proves the value is already stable. Kept and wrapped with `React.memo` when necessary. |
| `useCallback` | Same treatment as `useMemo`. |
| `useRef` | Passed through unchanged. No transformation. |
| `useContext` | Supported. Not signal-promoted in MVP — context updates still trigger VDOM reconciliation. |

### Supported JSX features

- Functional components — arrow functions and named declarations
- `React.memo` — auto-inserted by the optimizer, you never write it manually
- `forwardRef` — detected and preserved as-is
- Conditional rendering — `&&` and ternary expressions
- List rendering — `array.map()` returning JSX

### Explicitly out of MVP scope

- `useReducer`
- `useTransition` / `useDeferredValue`
- `Suspense` / `React.lazy`
- Class components
- Server components / RSC
- Custom hook analysis (custom hooks work but the compiler does not look inside them)

---

## Installation

> Not yet published. Clone and build locally.

```bash
git clone https://github.com/your-username/reactc.git
cd reactc
pnpm install
pnpm build
```

---

## Usage

### Compile

```bash
reactc compile src/
```

Compiles all `.jsx` and `.tsx` files in `src/` and emits to `dist/`.

### Watch

```bash
reactc watch src/
```

### Typecheck only

```bash
reactc check src/
```

### Config

Create a `reactc.config.ts` at the project root:

```ts
import { defineConfig } from 'reactc'

export default defineConfig({
  entryPoints: ['src/index.tsx'],
  outDir: 'dist',
  signalPromotion: true,
  autoMemo: true,
  target: 'auto', // 'signal' | 'vdom' | 'auto'
})
```

| Option | Default | Description |
|---|---|---|
| `entryPoints` | — | Files to compile |
| `outDir` | `dist` | Output directory |
| `signalPromotion` | `true` | Promote eligible `useState` calls to signals |
| `autoMemo` | `true` | Auto-insert `React.memo` / `useMemo` / `useCallback` |
| `target` | `'auto'` | Force all components to one backend, or let the compiler decide |

---

## Compilation report

After each compile, reactc prints a report showing what the compiler did per component:

```
reactc — compilation report

Component          Strategy   Signals   Hooks memoized   Re-renders eliminated
─────────────────────────────────────────────────────────────────────────────
Counter            signal     1         0                all
UserProfile        vdom       0         3                partial
Header             static     0         0                all
ProductList        vdom       0         5                partial
```

---

## Packages

reactc is a monorepo. Each package is independently versioned.

| Package | Description |
|---|---|
| `@reactc/lexer` | Tokenizer — produces typed tokens with source spans |
| `@reactc/parser` | AST builder — Babel-compatible AST with TypeScript types |
| `@reactc/analyzer` | Component graph, hook graph, and type resolution |
| `@reactc/ir` | Reactive IR and Static IR definitions and builder |
| `@reactc/optimizer` | Auto-memoizer, signal promoter, and tree shaker passes |
| `@reactc/codegen` | Signal backend and VDOM backend code generators |
| `@reactc/runtime` | Signal Core, React compat hooks, scheduler integration |
| `@reactc/cli` | CLI — compile, watch, check commands |

---

## Architecture

```
Source (JSX / TSX / TS)
        │
        ▼
  Lexer / tokenizer
  AST builder
        │
        ▼
  Component analyzer
  Hook graph builder
  Type resolver
        │
        ▼
  Reactive IR ◄──── IR builder ────► Static IR
        │
        ▼
  Auto-memoizer
  Signal promoter
  Tree shaker
        │
      ┌─┴─────────────┐
      ▼               ▼
 Signal backend   VDOM backend
      │               │
      └──────┬─────────┘
             ▼
  Signal Core runtime
  React compat layer
  Scheduler
             │
             ▼
  Optimized bundle (JS + sourcemaps + .d.ts)
```

---

## Design references

reactc's design draws from three codebases:

- **React** (`facebook/react`) — source of truth for hook semantics, scheduling, and the JSX runtime contract. Everything reactc outputs must behave identically to React from the developer's perspective.
- **SolidJS** (`solidjs/solid`) — model for the signal runtime and the JSX-to-DOM-binding transform used by the signal backend.
- **Svelte** (`sveltejs/svelte`) — reference for compiler architecture: analysis pass structure, IR design, and diagnostic reporting.

---

## Contributing

The project is in early architecture phase. Issues and design discussions are welcome. Code contributions will be opened once the core compiler pipeline is working end to end.

---

## License

MIT
