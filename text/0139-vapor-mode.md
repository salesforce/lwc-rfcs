---
title: Vapor Mode (VDOM-less rendering)
status: DRAFTED
created_at: 2026-09-10
updated_at: 2026-09-10
champion: John Hefferman (jhefferman)
pr: (leave this empty until the PR is created)
---

# Vapor Mode

## Summary

Vapor Mode is an opt-in, alternative compilation and rendering strategy for
Lightning Web Components that eliminates the virtual DOM. Instead of compiling a
template to a function that produces a virtual node (VNode) tree — which the
engine then diffs and patches — the Vapor compiler emits code that creates the
real DOM once and wraps each dynamic binding in a fine-grained reactive effect
that writes directly to the DOM. Update cost then scales with the size of the
*change*, not the size of the *template*.

The approach is modeled on Vue Vapor (itself inspired by Solid.js) and adapted
to LWC's component model, template syntax, and reactivity. It is delivered as
two additive packages — `@lwc/template-compiler-vapor` and `@lwc/engine-vapor` —
that sit alongside the existing compiler and engine and change no existing
behavior. This RFC documents the design as realized in the MVP (see
*Implementation status*).

## Basic example

Given this template:

```html
<template>
    <button class={btnClass} onclick={handleClick}>{label}</button>
</template>
```

The **standard** LWC compiler produces a template function that returns a VNode
tree on every render, which the engine diffs against the previous tree:

```js
function tmpl($api, $cmp, $slotset, $ctx) {
    const { h: api_element, t: api_text, b: api_bind } = $api;
    return [
        api_element('button', {
            className: $cmp.btnClass,
            on: { click: api_bind($cmp.handleClick) },
            key: 0,
        }, [api_text($cmp.label)]),
    ];
}
```

The **Vapor** compiler produces direct DOM operations with no VNodes:

```js
import { child, delegate, delegateEvents, renderEffect, setClass, setText, template }
    from '@lwc/engine-vapor';

const t0 = template("<button> </button>");
delegateEvents("click");

export default function render($cmp, $slotset) {
    const n0 = t0();                                  // real <button>, cloned from a cached template
    const n1 = child(n0);                             // the text node
    delegate(n0, "click", e => $cmp.handleClick(e));  // one delegated listener
    renderEffect(() => setClass(n0, $cmp.btnClass));  // re-runs only when btnClass changes
    renderEffect(() => setText(n1, $cmp.label));      // re-runs only when label changes
    return n0;
}
```

When `this.label` changes, only the second effect re-runs and writes
`n1.nodeValue` directly. No tree is rebuilt and no diff is performed.

The authored `.html` template and `.js` class are **identical** in both modes.
Vapor is a compilation target, not a new authoring model.

## Motivation

The virtual DOM is the dominant cost center in LWC's runtime update path. On
every change to reactive state, the engine:

1. Re-invokes the template function, allocating a fresh VNode tree.
2. Walks the new tree and diffs it against the previous tree.
3. Patches the real DOM where differences are found.

Steps 1 and 2 cost CPU and memory proportional to the **size of the template**,
not the **size of the change**. A component with 200 nodes that changes one text
value still allocates ~200 VNodes and diffs all of them. This is wasteful in the
common case where a small amount of state drives a small, localized DOM change.

Constraints we are trying to satisfy:

- **Update cost proportional to what changed**, not to template size.
- **Lower memory churn** — avoid per-render allocation of a VNode tree.
- **Smaller runtime** when an application uses only Vapor components — the entire
  diffing engine can be tree-shaken away.
- **No change to the authoring experience** — developers write the same `.html`
  templates and the same `LightningElement` JavaScript classes.
- **Coexistence** — Vapor and standard components must interoperate in the same
  app during a long migration period.

Benchmarks of the MVP on the js-framework-benchmark operation set (see *Detailed
design → Performance*) show Vapor **decisively faster on updates** — the case
that dominates real interactive applications — at the cost of being modestly
slower on initial mount and full teardown. This is the same trade-off profile as
Vue Vapor and Solid.js, and it is the honest, expected shape for VDOM-less
rendering.

## Detailed design

Vapor Mode is delivered as two new packages. **No existing package's behavior
changes.**

- `@lwc/template-compiler-vapor` — compiles a `.html` template to a render
  module of direct DOM operations.
- `@lwc/engine-vapor` — the VDOM-less runtime providing `template()`,
  `renderEffect()`, `setText()`, `createIf()`, `createFor()`, slots, event
  delegation, the reactive membrane, and an `lwc`-compatible facade
  (`LightningElement`, `createElement`, decorators, …).

### Terminology

- **Block** — a unit of rendered output. Concretely, the runtime type is
  `Block = Node | VaporFragment | DynamicFragment | Block[]`: a single DOM
  `Node`, a multi-root `VaporFragment` (a group of nodes bracketed by marker
  comments), a `DynamicFragment` (an anchored, reactively-swapped placeholder),
  or an array of these. Blocks are the Vapor analog of a VNode subtree, but they
  *are* the real nodes.
- **Render effect** — a reactive effect (`renderEffect(fn)`) that tracks the
  reactive values read during `fn` and re-runs `fn` (writing to the DOM) when any
  of them change.
- **Dynamic fragment** — a `Block` placeholder anchored by a comment node whose
  contents are swapped reactively. It is the shared primitive behind `lwc:if`,
  `for:each`, and `<slot>`.

### Compilation pipeline

The Vapor compiler is a three-stage pipeline that mirrors the standard
compiler's structure (parse → transform → generate). The entry point is
`compileVapor(source, options)`, which returns `{ code, warnings }`:

1. **Parse.** The template HTML is parsed into a node tree by a **focused parser
   internal to `@lwc/template-compiler-vapor`** (`src/parser.ts`). It recognizes
   the subset of LWC template syntax relevant to Vapor: elements, interpolation
   (`{expr}`), bound attributes (`attr={expr}`), event handlers
   (`onclick={fn}`), and the structural directives `if:true` / `if:false` /
   `lwc:if` / `lwc:elseif` / `lwc:else`, `for:each` / `for:item` / `for:index`,
   `iterator:*`, `key`, `lwc:ref`, `lwc:spread`, `lwc:on`, `lwc:dom="manual"`,
   `lwc:inner-html`, `lwc:is`, and scoped-slot bindings (`lwc:slot-data` /
   `lwc:slot-bind`). The package does still depend on `@lwc/template-compiler`,
   but today only for `generateScopeTokens`, so that scoped-style and
   synthetic-shadow scope tokens are computed identically to the standard
   compiler. (Fully sharing the standard parser/AST is called out under
   *Unresolved questions*.)

2. **Transform (AST → IR).** The tree is lowered to an intermediate
   representation (`src/ir.ts`, `src/transform.ts`) that explicitly separates:
   - **static structure** — accumulated into an HTML string per block,
     instantiated once at runtime via `template()` and cloned per use;
   - **node references** — traversal paths used to reach the dynamic nodes inside
     a cloned template. A `NodeRef` is a `varName` plus an ordered list of
     `Accessor`s, each `{ kind: 'child' }`, `{ kind: 'next' }`, or
     `{ kind: 'nthChild', index }`;
   - **operations** — one-time side effects (event listeners, refs, control-flow
     construction);
   - **effects** — reactive bindings, each destined to be wrapped in its own
     `renderEffect`.

   The IR node types (`IRNodeTypes`) are: `ROOT`, `BLOCK`, `SET_PROP`,
   `SET_ATTR`, `SET_CLASS`, `SET_STYLE`, `SET_TEXT`, `SET_EVENT`,
   `SET_DYNAMIC_PROPS`, `SET_HTML`, `LWC_ON`, `IF`, `FOR`, `SLOT`, `COMPONENT`,
   and `REF`.

3. **Generate (IR → JS).** The IR is emitted (`src/codegen/generate.ts`) as an
   ES module that imports helpers from `@lwc/engine-vapor` and default-exports a
   `render($cmp, $slotset)` function. A dead-code pass prunes node-reference
   traversals that no operation or effect actually consumes.

### Static hoisting and node references

Static subtrees become a single template string, parsed into DOM once and cloned
on each instantiation. Dynamic descendants are reached by cheap traversal rather
than by `querySelector`:

```js
const t0 = template("<div class=\"card\"><h2> </h2><p> </p></div>");

export default function render($cmp) {
    const n0 = t0();            // clone the static card
    const n1 = child(n0);       // <h2>
    const n2 = child(n1);       // <h2>'s text node
    const n3 = nthChild(n0, 1); // <p>
    const n4 = child(n3);       // <p>'s text node
    renderEffect(() => setText(n2, $cmp.title));
    renderEffect(() => setText(n4, $cmp.body));
    return n0;
}
```

When a block's static template has more than one top-level node, the template
factory returns a cloned `DocumentFragment`, the root variable is that fragment,
and top-level nodes are reached via `nthChild`.

### Reactivity

Vapor ships a fine-grained reactive core (`src/reactivity.ts`,
`src/renderEffect.ts`):

- Reactive component fields are read through a `reactive()` proxy that calls
  `track(dep)` during effect execution, recording a dependency between the
  currently-running effect and that field.
- Writing a field calls `trigger(dep)`, which notifies (and re-runs) exactly the
  effects that read it.
- The dependency graph is an **intrusive doubly-linked list** ported from Vue
  Vapor / alien-signals — *not* a mark-and-sweep `Set`-based tracker. A `Link` is
  a single dependency edge that lives simultaneously in two lists: the dep's
  subscriber list and the effect's dependency list. This buys two properties that
  the benchmarks depend on:
  - **Teardown is O(1) pointer splices per edge** (`unlink`) rather than O(deps)
    hash-set deletes — the source of the large clear/teardown wins.
  - **Steady-state re-runs reuse the existing `Link` nodes positionally**, so an
    update in the hot path allocates nothing.

  LWC has no computed layer, so a subscriber is always a plain effect and a dep is
  always plain reactive data; Vue's cascading-computed cleanup and dirty-flag
  bookkeeping are intentionally dropped.
- Arrays are instrumented (`push`, `pop`, `shift`, `unshift`, `splice`, `sort`,
  `reverse`) and an `ITERATE_KEY` supports length/iteration tracking; an
  `EPOCH_KEY` supports component-level "something changed" tracking.
- Writes can be **batched** so multiple field updates in one turn flush their
  effects once.

An **async render model with Vue parity** is implemented (`ASYNC_RENDER_PARITY.md`):
effects can defer their post-initial-run re-notify to a shared microtask-batched
queue (Vue Vapor's `notify → queueJob`), with the initial mount kept synchronous.
It is **off by default** in the MVP behind flags (`ENABLE_ASYNC_RERENDER`,
`VUE_PARITY_ASYNC`); see *Implementation status*.

### Control flow

**Conditionals** (`lwc:if` / `lwc:elseif` / `lwc:else`, and legacy
`if:true` / `if:false`) compile to `createIf`, which returns a dynamic fragment
whose contents are swapped when the condition changes. `lwc:if` brackets its
content with a `<!---->` comment pair; the legacy `if:true` / `if:false` form
renders no bookends, matching engine-core semantics:

```js
const d0 = createIf(
    () => $cmp.visible,
    () => { const n0 = t0(); return n0; }   // positive branch
);
```

**Loops** (`for:each` and `iterator:*`) compile to `createFor` / `createIterator`,
which maintain keyed or unkeyed blocks and reconcile them against the source. The
keyed path uses a longest-increasing-subsequence move strategy (as in Vue /
Inferno) so reorders move the minimum number of DOM nodes. When a loop appears
inside a static parent, the compiler emits a comment anchor inside the parent's
static template and inserts the loop's blocks at that anchor; an **anchorless**
mode (`ANCHORLESS_ITERATOR_DESIGN.md`) avoids the anchor comment when the loop's
position can be derived from its siblings:

```js
const t1 = template("<ul><!--lwc-vapor--></ul>");
const t0 = template("<li> </li>");

export default function render($cmp) {
    const n0 = t1();
    const n1 = nthChild(n0, 0);                  // the anchor comment
    const d0 = createFor(
        () => $cmp.items,
        (item) => {
            const n2 = t0();
            const n3 = child(n2);
            renderEffect(() => setText(n3, item.name));
            return n2;
        },
        (item) => item.id                        // key function
    );
    insert(d0, n0, n1);                          // mount the list at the anchor
    return n0;
}
```

### Events

Events that bubble and are safe to delegate (click, input, change, keydown, …)
are attached once at the document level via `delegateEvents()`, with per-element
handlers dispatched by walking the composed path (respecting shadow boundaries).
Non-delegatable events use a direct listener (`on`) with automatic cleanup tied
to the owning scope. `lwc:on={obj}` binds each property of an object as a
listener, and event objects can be spread with `spreadEvents`.

### Components, slots, and refs

- **Child components** are created through the facade's `createElement` /
  `createChildComponent`, with props passed as live getters so the child's
  bindings stay reactive. `lwc:is` compiles through `createDynamicComponent`.
- **Slots** compile to `createSlot(name, $slotset, fallback?, …)`, a dynamic
  fragment that renders passed-in slot content or fallback. Light-DOM and
  shadow-DOM distribution, **multi-level slot forwarding** (slot → slot chains),
  and **scoped slots** (`scopedSlot` / `scopedSlotFragment`, driven by
  `lwc:slot-data` / `lwc:slot-bind`) are implemented; see
  `SLOT_DISTRIBUTION_PARITY.md`.
- **`lwc:ref`** compiles to `applyRefs(node, name)`, populating `this.refs`.

### Component lifecycle and the `lwc` facade

`@lwc/engine-vapor` defines a component instance (`VaporComponentInstance`) that
owns its root block, its shadow root (native open/closed or light DOM, matching
`renderMode` / `shadowMode`), its reactive props, and its effect scope.
`createVaporComponent` / `mountComponent` / `unmountComponent` drive it, and
lifecycle hooks (`connectedCallback`, `renderedCallback`,
`disconnectedCallback`, `errorCallback`, …) map onto the standard custom-element
lifecycle.

Crucially, the runtime ships a **compat facade** (`src/compat/`) that re-exports
the public `lwc` API surface — `LightningElement`, `createElement`,
`registerComponent`, `registerTemplate`, the `@api` / `@track` / `@wire`
decorators, `createContextProvider`, `readonly` / `unwrap`, the hot-swap APIs,
and more — from a single built bundle. This is what lets a Vapor build serve both
the `@lwc/engine-vapor` runtime imports and the `lwc` module alias, so the
authored component class needs no changes.

### Opt-in mechanism

Vapor compilation is gated by a single build-time flag,
`enableVaporCompilation`, plumbed through both `@lwc/compiler`
(`DEFAULT_OPTIONS.enableVaporCompilation = false`) and `@lwc/rollup-plugin`
(`RollupLwcOptions.enableVaporCompilation`). When the flag is set, the template
transformer routes to `compileVapor()` and returns the direct-DOM render module;
when it is unset, compilation is byte-for-byte unchanged. There is no
per-component or API-version opt-in yet (see *Adoption strategy* and *Unresolved
questions*).

### Performance

The MVP is benchmarked with the **"krausest" js-framework-benchmark operation
set ported directly from Vue Vapor's own benchmark** (create 1k/10k rows, update
every 10th row, swap, remove, clear), using Vue's exact adjective/colour/noun
data generator. A comparison driver (`scripts/compare-vapor.mjs`) runs the *same*
components against both engines by rebuilding `perf-benchmarks-components` with
`PERF_ENGINES=dom,vapor` and rewriting the benchmark imports at bundle time, so
the two runs are isolated and directly comparable.

The measured profile (Vapor vs. VDOM, same component, in-process):

| Operation | Verdict |
| --- | --- |
| **Swap two rows in a 1,000-row list** | **Vapor ~10.1× faster** |
| **Remove rows one-by-one** | **Vapor ~6.3× faster** |
| **Update a single bound value** | **Vapor ~1.4× faster** |
| Create / clear in bulk, and update-everything | VDOM ~1.2–2.65× faster |

This is the **same performance profile as Vue Vapor and Solid.js**:

- **Surgical updates win decisively.** Fine-grained reactivity touches only the
  DOM nodes whose data changed; there is no tree rebuild or diff. This dominates
  real interactive applications.
- **Bulk create / full teardown / update-everything are modestly slower.** Each
  row sets up its own effect scope and reactive refs — real per-row cost the VDOM
  baseline does not pay, and a full-list rebuild is exactly the case a diff is
  already good at.

The mount/teardown/bulk tax is the accepted trade-off for eliminating the virtual
DOM; it is bounded (a small constant factor) and is outweighed by the update
gains over a component's lifetime. The intrusive-linked-list teardown (O(1)
splices) already reclaims much of the teardown cost; further reduction (pooling
effect scopes, batching scope teardown) is future work.

### Implementation status

This RFC documents an MVP checkpoint, not a finished feature. What is **built and
tested** today:

- Both packages exist and compile: a full parse → IR → codegen pipeline and the
  fine-grained runtime, including `template`/`child`/`nthChild`/`next`, the
  `set*` binding helpers, `renderEffect`, `createIf`, `createFor` /
  `createIterator` (keyed LIS reconciliation), `createSlot` with scoped slots and
  multi-level forwarding, event delegation, `applyRefs`, the reactive membrane,
  and the `lwc` compat facade.
- Directive coverage includes `lwc:if` / `lwc:elseif` / `lwc:else`, legacy
  `if:true` / `if:false`, `for:each`, `iterator:*`, `lwc:ref`, `lwc:spread`,
  `lwc:on`, `lwc:inner-html`, `lwc:dom="manual"`, and `lwc:is`.
- **87 unit tests** pass across the two packages.
- The full Web-Test-Runner integration suite runs under the Vapor runtime at
  **3,248 passed / 50 failed / 1,169 skipped** at the synchronous baseline
  (3,246 / 52 with the Vue-parity async model enabled). All remaining failures are
  triaged into ten remediation groups (A–J) in
  `packages/@lwc/engine-vapor/VAPOR_REMAINING_FAILURES.md`.

The **most significant known gap** is *whole-template re-render*. Vapor's
fine-grained effects re-run only the bindings whose exact dependencies changed;
several LWC semantics (observed fields, external side-effects, some
memoization/`@wire` de-dup, scoped-slot rehydration counts) expect that *any*
tracked change re-invokes `render()` and re-reads *all* bindings. Enabling that
globally would forfeit the fine-grained wins, so it must be an opt-in, per-component
lever — the largest remaining piece of runtime work and the head of remediation
Group A. Other open areas are SSR/hydration (the MVP is client-only), a callable
`LightningElement` base for Locker, and full synthetic-shadow integration.

## Drawbacks

- **Implementation and maintenance cost.** Two substantial new packages (a
  compiler and a runtime) must be built and kept in sync with template-syntax and
  reactivity changes. This is the largest cost.
- **Two rendering models.** Until/unless Vapor fully replaces the VDOM engine,
  LWC maintains two renderers. Every feature must be considered in both.
- **A second reactivity system (for now).** The MVP ships its own `reactive()`
  membrane rather than reusing engine-core's `@track` / `@api` / signals
  machinery. Maintaining two reactive cores is a real cost and a correctness risk
  until they are unified.
- **Interop complexity.** A standard parent rendering a Vapor child (and vice
  versa) requires a documented boundary protocol. This is among the riskiest
  areas.
- **Whole-template semantics.** Some existing behaviors assume a full re-render on
  any change; matching them without giving back the fine-grained wins is
  non-trivial (see *Implementation status*).
- **SSR/hydration.** Server rendering and hydration must be designed for the
  Vapor output format; the MVP is client-only.
- **Debugging shift.** Stack traces and devtools reflect effects and direct DOM
  ops rather than a VNode tree; tooling and docs must adapt.
- **Cannot be implemented in user space.** It requires compiler and engine
  changes.

## Alternatives

- **Do nothing / keep optimizing the VDOM.** Continued incremental wins (static
  hoisting, `parseFragment`, compiler-informed patch flags) are possible, but they
  cannot make update cost proportional to change size; the diff is fundamental to
  the model.
- **Compile to imperative patches but keep a VNode tree** (the "block tree"
  approach). Less invasive, but retains per-render allocation and a diff step, and
  has a smaller ceiling on gains.
- **Adopt Vue Vapor's output verbatim.** Rejected: LWC's template syntax
  (`for:each`, `lwc:if`, `lwc:ref`, `lwc:on`, …), its shadow/light DOM model,
  scoped styles, and reactivity differ enough that a direct port would not fit.
  The MVP instead borrows Vapor's *reactivity data structures* (the intrusive
  linked-list graph, LIS reconciliation) while lowering LWC's own directive
  semantics.
- **Reuse the standard template compiler's parser/AST from day one.** Deferred in
  the MVP in favor of a focused parser, to iterate quickly on the IR and codegen;
  reusing the standard parser is the intended production direction (see
  *Unresolved questions*) and is why the package already depends on
  `@lwc/template-compiler` for scope-token computation.

Impact of not doing this: LWC's update performance remains bounded by VDOM
overhead, and bundle size cannot shed the diffing engine for Vapor-only apps.

## Adoption strategy

This is **opt-in and non-breaking**. Proposed phasing:

1. **Experimental (today).** Ship both packages behind the compiler flag
   `enableVaporCompilation`. Templates compile to Vapor only when opted in. No
   existing component is affected.
2. **Per-component opt-in.** Add a per-component way to declare Vapor rendering
   (e.g. a static class field or template marker), enabling gradual migration.
   Not built yet.
3. **Interop guarantees.** Standard ↔ Vapor parent/child rendering is supported
   throughout, so teams migrate leaf components first and work upward.

Because the authored `.html` and `.js` are unchanged, **no codemod is required**
for the common case. A lint/validation pass should flag any template construct
not yet supported by the Vapor compiler so authors know before opting in, and the
supported-feature matrix (including the whole-template-re-render caveat) must be
published alongside.

# How we teach this

Vapor Mode should be taught as **an implementation detail, not a new programming
model**: "the same component, compiled to skip the virtual DOM." The headline for
developers is "faster updates and smaller bundles, same code."

Documentation additions:

- A conceptual page contrasting VDOM diffing with fine-grained reactive updates,
  using the basic example above.
- A short "compiled output" explainer for advanced users debugging Vapor
  components (what `renderEffect`, `template`, dynamic fragments, and delegated
  events are).
- Migration guidance: how to opt in, how to read the supported-feature matrix,
  the whole-template-re-render caveat, and how interop works.

Existing patterns (`@api`, `@track`, `lwc:if`, `for:each`, `lwc:ref`, `lwc:on`,
slots, shadow/light DOM) carry over unchanged conceptually, which keeps the
teaching burden low.

# Unresolved questions

- **Reactivity unification.** The MVP ships its own fine-grained reactive core.
  Production must bridge to LWC's existing reactivity membrane, `@track` / `@api`,
  and signals so there is one source of truth. What is the exact bridge, and what
  are its performance implications?
- **Whole-template re-render.** What is the precise opt-in signal (compiler-emitted
  vs. runtime flag), and how is it scoped per component so the fine-grained wins
  are preserved for everything else?
- **Parser/AST unification.** When and how does the Vapor compiler move from its
  focused parser to fully reusing `@lwc/template-compiler`'s parser and AST, so
  directive semantics, validation, and error messages are guaranteed identical?
- **SSR and hydration.** What is the Vapor output's serialized form, and how do
  effects "adopt" server-rendered DOM on the client?
- **Synthetic shadow.** How does Vapor interact with synthetic-shadow styling and
  scoping, and with scoped-CSS tokens beyond the shared `generateScopeTokens`?
- **Locker.** What is the callable-`LightningElement`-base shape Locker requires
  (remediation Group I)?
- **Opt-in surface.** Compiler flag vs. per-component declaration vs.
  API-version — which (or which combination) is the right developer-facing knob?
