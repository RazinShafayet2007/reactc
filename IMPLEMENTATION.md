# reactc — LLM-Assisted Implementation Guidelines

> A step-by-step workflow for building the React Compiler Architecture with an LLM.
> Each phase tells you what to build, how to prompt, what to verify, and when to commit.

---

## Core Principles

**One component per session.** Never ask the LLM to build two phases at once. The output gets shallow and hard to test.

**Tests before code.** For every component, ask for the test suite first. This forces the LLM to think about the interface before the implementation.

**Paste your actual code.** Always give the LLM the real files you have so far — not a description of them. Context precision drives output precision.

**Commit gates.** Never move to the next phase until the current one has passing tests and a clean commit. Each phase depends on the previous one being correct.

**Be adversarial.** After the LLM gives you an implementation, ask it to try to break it. "What input would make this fail?" is the most valuable follow-up you can ask.

---

## Phase 0 — Project Setup

**Goal:** Establish repo structure, tooling, and the test harness before writing a single compiler line.

### Prompt sequence

```
1. "Design a monorepo structure for a TypeScript compiler called reactc.
   It has these packages: @reactc/lexer, @reactc/parser, @reactc/analyzer,
   @reactc/ir, @reactc/optimizer, @reactc/codegen, @reactc/runtime,
   @reactc/cli. Use pnpm workspaces. Show me the full directory tree
   and every package.json."

2. "Write the root tsconfig.json and a shared tsconfig.base.json
   that all packages extend. Target ESNext, strict mode, composite builds."

3. "Set up Vitest as the test runner across all packages. Show me
   the root vitest.config.ts and an example test file structure
   for @reactc/lexer."

4. "Write a GitHub Actions CI workflow: install, typecheck,
   test, build — in that order. Fail fast on typecheck errors."
```

### Verify before moving on

- `pnpm install` succeeds with no errors
- `pnpm -r typecheck` passes on an empty codebase
- `pnpm -r test` runs and reports 0 tests (not a failure)
- CI workflow runs green on an empty push

### Commit message

```
chore: project scaffold — monorepo, tsconfig, vitest, CI
```

---

## Phase 1 — Lexer / Tokenizer (`@reactc/lexer`)

**Goal:** A function that takes a source string and returns a flat array of typed tokens with exact source positions.

### Prompt sequence

```
1. "Define the full TokenType enum for a JSX/TSX lexer. It must cover:
   keywords (import, export, return, const, let, function, etc.),
   JSX tokens (JSXOpen, JSXClose, JSXSelfClose, JSXText, JSXExprOpen),
   TypeScript tokens (generic angle brackets, type annotations, as, satisfies),
   operators, punctuation, literals, and identifiers.
   Output as a TypeScript const enum with a comment on each variant."

2. "Now write the Token interface: it holds TokenType, the raw text,
   and a SourceSpan { file, line, col, offset, length }.
   Then write the Lexer class with a single public method:
   tokenize(source: string, file: string): Token[]
   Show me the full implementation."

3. "Write the Vitest test suite for the Lexer. Include test cases for:
   - basic identifiers and keywords
   - JSX open/close/self-closing tags
   - TypeScript generic syntax like Foo<Bar, Baz>
   - template literals
   - nested JSX expressions {value}
   - edge case: empty input
   - edge case: unterminated string literal (should produce an Error token,
     not throw)"

4. [After tests pass] "What are the five most likely inputs that would
   cause this lexer to produce wrong tokens? Show me each one and
   how to fix the lexer to handle it."
```

### Key decisions to make explicit to the LLM

- JSX is context-sensitive (inside JSX, `<` is a tag, not less-than). Tell the LLM you want a mode stack — the lexer tracks whether it is in JSX mode or expression mode.
- TypeScript angle brackets (`<T>`) vs JSX tags must be disambiguated. Tell the LLM your disambiguation rule upfront (e.g. preceded by an identifier = generic, preceded by whitespace/operator = JSX).

### Verify before moving on

- All test cases pass
- `tokenize` never throws — bad syntax produces Error tokens with positions
- Source positions are exact (write a test that checks `token.span.offset` against `source.indexOf(token.text)`)

### Commit message

```
feat(lexer): tokenize JSX/TSX/TS source with source spans
```

---

## Phase 2 — AST Builder (`@reactc/parser`)

**Goal:** A function that takes a `Token[]` from the lexer and returns a typed AST. Wrap `@babel/parser` rather than writing a full parser from scratch.

### Prompt sequence

```
1. "I am wrapping @babel/parser to produce a Babel AST enriched with
   TypeScript type information. Write the parse() function signature:
     parse(source: string, file: string, tokens: Token[]): BabelFile
   Show how to configure @babel/parser plugins for JSX and TypeScript,
   and how to attach my Token[] to the AST nodes via a WeakMap so that
   downstream passes can look up source spans without re-scanning."

2. "Write the ASTNode union type — a discriminated union covering the
   node kinds I will use: ComponentDeclaration, HookCall, JSXElement,
   JSXAttribute, JSXExpression, ImportDeclaration, ExportDeclaration.
   Each node carries a SourceSpan and an optional typeAnnotation field."

3. "Write a NodeVisitor utility: a walk(node, visitors) function that
   does a depth-first traversal of the AST and calls the right visitor
   for each node type. Show a usage example that collects all HookCall
   nodes from a parsed file."

4. "Write Vitest tests for parse(). Test cases:
   - A simple functional component returning JSX
   - A component with useState and useEffect
   - A component using forwardRef
   - Invalid JSX (parser should return a ParseError, not throw)
   - Empty file"
```

### Verify before moving on

- The Token-to-ASTNode WeakMap is populated correctly (write a test)
- `walk()` visits every node — use a counter test
- ParseError is a value, not an exception

### Commit message

```
feat(parser): Babel-based AST builder with typed node visitors
```

---

## Phase 3 — Analysis (`@reactc/analyzer`)

This is the most important phase. Take it slowly — three separate sessions.

### 3a — Component Analyzer

```
1. "Given a BabelFile AST and the walk() utility, write a
   ComponentAnalyzer class with one method:
     analyze(ast: BabelFile): ComponentGraph
   ComponentGraph is a Map<string, ComponentNode> where ComponentNode
   holds the component's name, its source span, a list of child
   component names it renders, and whether it uses forwardRef or memo().
   Show the full implementation."

2. "Write tests for ComponentAnalyzer:
   - A file with two components where one renders the other
   - A forwardRef component
   - A memo() wrapped component
   - An anonymous default export component"
```

### 3b — Hook Graph Builder

```
1. "Write a HookGraphBuilder that takes a ComponentNode and its AST
   subtree and returns a HookGraph:
     - A list of HookCall nodes (name, arguments, return bindings)
     - A DependencyEdge[] list: each edge says 'this hook call reads
       this binding from this other hook call'
   Handle useState, useEffect, useMemo, useCallback, useRef,
   useContext, and custom hooks."

2. "Write tests for HookGraphBuilder:
   - useState value used in useEffect deps
   - useMemo depending on two useStates
   - Custom hook that calls useState internally
   - Missing dependency array in useEffect (should be flagged as a
     diagnostic, not an error)"

3. "What are the three hardest dependency inference cases for the
   Hook Graph Builder? Show me the AST shape for each and how
   to handle them."
```

### 3c — Type Resolver

```
1. "I want to use the TypeScript compiler API (ts.createProgram) to
   resolve types for hook return values and component props.
   Write a TypeResolver class:
     resolve(node: HookCall, program: ts.Program): ResolvedType
   ResolvedType is a discriminated union: Primitive | ObjectType |
   ArrayType | UnknownType. Show the implementation."

2. "Write a helper isPrimitiveHookState(hookCall, resolver): boolean
   that returns true when a useState's type argument is string, number,
   boolean, or a union of those. This is the primary signal-promotion
   heuristic."
```

### Verify before moving on

- `ComponentGraph` covers all component shapes you plan to support
- `HookGraph` correctly infers deps for at least 10 real-world hook patterns
- `isPrimitiveHookState` has 100% test coverage

### Commit message

```
feat(analyzer): component graph, hook graph, TypeScript type resolver
```

---

## Phase 4 — IR Generation (`@reactc/ir`)

**Goal:** Lower the analysis output into two typed graph IRs.

```
1. "Define the ReactiveIR type: a directed graph where nodes are one of:
   SignalNode (a mutable source), MemoNode (a derived computation),
   EffectNode (a side-effect with subscriptions), or DOMBindingNode
   (a JSX slot bound to a signal). Edges are SubscriptionEdge.
   Use a Map<NodeId, IRNode> plus an adjacency list. Show the full types."

2. "Define StaticIR: a tree of StaticNode, each holding a pre-computed
   JSX subtree that carries no reactive dependencies. Nodes reference
   their source HookCall or JSXElement for traceability."

3. "Write an IRBuilder class that takes a ComponentGraph + HookGraph
   and produces { reactive: ReactiveIR, static: StaticIR }.
   The rule: if a node's value is entirely determined by primitive
   signals, it goes into ReactiveIR. If it reads no signals at all,
   it goes into StaticIR. Ambiguous nodes go into ReactiveIR."

4. "Write tests for IRBuilder:
   - A counter component (useState<number>) → entire state in ReactiveIR
   - A static header component → entire JSX in StaticIR
   - A mixed component (reactive body, static footer)
   - A component with useMemo depending on a signal"
```

### Verify before moving on

- Every `HookCall` in the HookGraph maps to exactly one IR node (no orphans)
- A round-trip test: lower to IR, serialize to JSON, deserialize, compare node counts

### Commit message

```
feat(ir): ReactiveIR and StaticIR with IRBuilder lowering pass
```

---

## Phase 5 — Optimizer (`@reactc/optimizer`)

Three passes, three sessions.

### 5a — Auto-Memoizer

```
1. "Write an AutoMemoizer pass that takes a ReactiveIR and annotates
   MemoNodes with ShouldMemo | AlreadyStable | CanElide.
   ShouldMemo: insert React.memo or useMemo
   AlreadyStable: value never changes — move to StaticIR
   CanElide: useMemo wraps a primitive — remove the wrapper
   Show the full implementation."

2. "Write tests using fixture files: a component with an unnecessary
   useMemo wrapping a string literal should be CanElide. A component
   passing a new object literal as a prop on every render should
   be ShouldMemo."
```

### 5b — Signal Promoter

```
1. "Write a SignalPromoter pass. It takes a ReactiveIR and for each
   SignalNode calls isPrimitiveHookState from the TypeResolver.
   If true AND the signal has no Suspense-boundary ancestors AND
   no Context.Provider ancestors, it marks the node as Promoted.
   Return a PromotionReport: { promoted: NodeId[], kept: NodeId[],
   reason: Map<NodeId, string> }."

2. "Write tests:
   - useState<number> with no Suspense → Promoted
   - useState<string[]> → kept (non-primitive)
   - useState<number> inside a Suspense boundary → kept
   - Custom hook returning a number useState → Promoted"

3. "What would cause a false promotion (a signal that should stay
   as VDOM state but gets promoted)? Give me three real cases and
   add them as regression tests."
```

### 5c — Tree Shaker

```
1. "Write a TreeShaker pass that removes dead nodes from both IRs:
   - StaticIR nodes with no parent reference
   - ReactiveIR EffectNodes with no observable output
   - MemoNodes whose result is never read
   Return a ShakeReport listing removed NodeIds and why."

2. "Write tests using fixture components that have dead code:
   a useEffect that writes to a ref that's never read,
   a useMemo whose return value is discarded."
```

### Commit message

```
feat(optimizer): auto-memoizer, signal promoter, tree shaker passes
```

---

## Phase 6 — Code Generator (`@reactc/codegen`)

Two backends, two sessions.

### 6a — Signal Backend

```
1. "Write a SignalBackend that takes the optimized ReactiveIR and emits
   JavaScript as a string. Rules:
   - Promoted SignalNodes → createSignal() calls
   - MemoNodes → createMemo() calls
   - EffectNodes → createEffect() calls
   - DOMBindingNodes → () => signal() arrow inside a reactive JSX slot
   Use @babel/generator to produce clean output from a generated AST,
   not string concatenation."

2. "Write fixture tests: give the backend a ReactiveIR for a counter
   component and assert the output string contains createSignal,
   createEffect, and that the JSX slot reads () => count()."

3. "Show me how the signal backend handles a component with both
   promoted signals AND non-promoted VDOM state in the same render."
```

### 6b — VDOM Backend

```
1. "Write a VDOMBackend that takes the optimized ReactiveIR + StaticIR
   and emits React-compatible JSX. Rules:
   - ShouldMemo components are wrapped with React.memo()
   - Stable callback props are wrapped with useCallback()
   - Static subtrees are referenced by their hoisted module-scope variable
   Use @babel/generator. Output must be valid React JSX."

2. "Write fixture tests comparing input component source against expected
   compiled output. Include a component that gets a React.memo wrapper
   it didn't have in the source."
```

### Commit message

```
feat(codegen): signal backend and VDOM backend with Babel codegen
```

---

## Phase 7 — Runtime (`@reactc/runtime`)

### 7a — Signal Core

```
1. "Implement createSignal, createMemo, createEffect, batch, and untrack
   in plain TypeScript with no dependencies. Target size: under 2 KB
   minified. Use a simple push-based reactivity model with a global
   tracking stack. Show the full implementation with JSDoc."

2. "Write Vitest tests for every primitive. Include:
   - signal updates trigger effects
   - memo recomputes only when deps change
   - batch defers notifications until flush
   - untrack reads signal without subscribing
   - circular dependency detection throws"

3. "Benchmark: create 10,000 signals all updating at once. Measure
   time to settle. Show me the benchmark code."
```

### 7b — React Compat Layer

```
1. "Write useSignalState(initialValue): [() => T, (v: T) => void]
   that wraps createSignal with the useState interface. The getter is
   a function (not a value) so React's reconciler never sees it change.
   Show the implementation."

2. "Write useSignalEffect(fn, deps) that wraps createEffect and
   integrates with React's useEffect lifecycle (cleanup on unmount)."

3. "Write tests that mount a React component using useSignalState
   via @testing-library/react and assert that:
   - state updates cause DOM updates
   - React DevTools can see the component
   - unmounting cleans up the effect"
```

### 7c — Scheduler Integration

```
1. "Integrate the Signal Core's batch flush with React's scheduler
   package. Write scheduleSignalFlush() that calls scheduler.scheduleCallback
   at the right priority based on whether the update originates from
   a React synthetic event (InputContinuous) or from outside React
   (Normal)."

2. "Write a test that proves signal updates inside a React onClick
   handler are flushed at InputContinuous priority, not Normal."
```

### Commit message

```
feat(runtime): signal core, React compat hooks, scheduler integration
```

---

## Phase 8 — CLI (`@reactc/cli`)

```
1. "Write the reactc CLI using commander.js. Commands:
     reactc compile <file|glob>  — compile and emit to dist/
     reactc watch <file|glob>    — watch mode
     reactc check <file|glob>    — typecheck only, no emit
   Show the full implementation including config file loading
   (reactc.config.ts)."

2. "Write the reactc.config.ts schema using Zod:
   fields: entryPoints, outDir, signalPromotion (boolean),
   autoMemo (boolean), target ('signal' | 'vdom' | 'auto')."

3. "Write the compilation report emitter. After compiling, print
   a summary table: component name, strategy (signal/vdom/static),
   hooks memoized, estimated re-renders eliminated."
```

### Commit message

```
feat(cli): reactc compile/watch/check commands with config schema
```

---

## LLM Workflow Rules (Apply to Every Phase)

### Starting a new session

Always open with a context block:

```
I am building reactc, a React compiler with signal-based reactivity.
Here is the repo structure: [paste tree]
Here is what I have built so far: [paste relevant files]
I am now working on: [phase name]
The interfaces I need to satisfy: [paste types]
```

### Asking for implementation

- Ask for one function/class at a time, not a whole file
- Always specify: "Use TypeScript strict mode", "No any", "Export only what is needed"
- Always ask for JSDoc on every public method

### Asking for tests

- Specify the test runner: "Write Vitest tests using describe/it/expect"
- Ask for both happy path and edge cases in the same prompt
- Ask for fixture files separately from the test file

### Reviewing output

After every implementation, run this follow-up:

```
Review the code you just wrote for:
1. Any edge cases that would cause incorrect output (not a crash — wrong output)
2. Any TypeScript strict mode violations
3. Any missing null checks on values from WeakMap or Map lookups
4. Any place where an exception could escape that should be a typed error value
Fix all issues you find.
```

### When output is wrong

Do not ask the LLM to "fix" vague problems. Instead:

```
This test fails:
[paste the exact test]
The actual output is:
[paste the actual output]
The expected output is:
[paste what it should be]
Explain why the current code produces the wrong output, then fix it.
```

### Keeping sessions short

If a single LLM response is over ~200 lines of code, split the work. Long responses have higher error rates in compiler logic. Prefer multiple short, verified sessions over one long session.

---

## Testing Strategy Summary

| Phase | Test type | Fixture format |
|---|---|---|
| Lexer | Unit | Input string → expected Token[] |
| Parser | Unit | Input string → expected node count/kinds |
| Analyzer | Unit | Input string → expected graph structure |
| IR | Unit | Analyzer output → IR node count/types |
| Optimizer | Unit | IR in → annotated IR out, checked by node |
| Codegen | Snapshot | IR in → expected emitted JS string |
| Runtime | Unit + perf | Signal operations + timing |
| CLI | Integration | Input files → output files on disk |

Run snapshot tests with `vitest --update-snapshots` only on purpose — treat unexpected snapshot changes as failures, not updates.

---

## When to Stop Using the LLM

Switch to writing by hand when:

- The component is under 30 lines and you already know the pattern
- You are debugging a failing test (LLM debugging is slower than reading the stack trace yourself)
- You are writing types — type design benefits from your domain knowledge more than from LLM generation
- The LLM has given you two wrong answers in a row for the same problem (it is stuck; change your approach or the framing)
