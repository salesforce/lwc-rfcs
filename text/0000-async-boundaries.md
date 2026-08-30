---
title: Async Boundaries for Lightning Web Components
status: DRAFTED
created_at: 2025-01-12
updated_at: 2025-01-12
pr: https://github.com/salesforce/lwc-rfcs/pull/94
---

# Async Boundaries for Lightning Web Components

## Summary

This RFC proposes the introduction of declarative async boundaries to Lightning Web Components, enabling parent components to coordinate loading and error states across multiple async child components. This feature, inspired by Vue 3's `<Suspense>` and React's `<Suspense>`, would provide a standardized, framework-native solution for managing asynchronous UI rendering.

## Basic example

```html
<!-- dashboard.html -->
<template>
  <lwc:suspense>
    <c-dashboard-skeleton slot="fallback"></c-dashboard-skeleton>
    <c-error-panel slot="error"></c-error-panel>

    <!-- These components have async dependencies (@wire, imperative Apex, etc.) -->
    <div class="dashboard-grid">
      <c-account-list></c-account-list>
      <c-opportunity-chart></c-opportunity-chart>
      <c-recent-activity></c-recent-activity>
    </div>
  </lwc:suspense>
</template>
```

The `<lwc:suspense>` boundary automatically:

1. Detects when child components are waiting on async data (via `@wire` or other mechanisms)
2. Displays the `fallback` slot content until all children are ready
3. Displays the `error` slot content if any child encounters an error
4. Renders the default content once all async dependencies resolve

## Motivation

### The Problem

LWC applications frequently deal with asynchronous data fetching. Currently, every component that fetches data must independently manage its own loading, error, and success states. This leads to several problems:

#### 1. Boilerplate Explosion

Every async component requires the same pattern:

```javascript
// accountList.js - This pattern is repeated in EVERY async component
import { LightningElement, wire } from "lwc";
import getAccounts from "@salesforce/apex/AccountController.getAccounts";

export default class AccountList extends LightningElement {
  accounts;
  error;
  isLoading = true;

  @wire(getAccounts)
  wiredAccounts({ error, data }) {
    this.isLoading = false;
    if (data) {
      this.accounts = data;
      this.error = undefined;
    } else if (error) {
      this.error = error;
      this.accounts = undefined;
    }
  }

  get hasAccounts() {
    return this.accounts && this.accounts.length > 0;
  }
}
```

```html
<!-- accountList.html - Same conditional structure in EVERY template -->
<template>
  <template lwc:if="{isLoading}">
    <lightning-spinner alternative-text="Loading..."></lightning-spinner>
  </template>
  <template lwc:elseif="{error}">
    <c-error-panel errors="{error}"></c-error-panel>
  </template>
  <template lwc:elseif="{hasAccounts}">
    <!-- Actual content -->
  </template>
  <template lwc:else>
    <p>No accounts found.</p>
  </template>
</template>
```

This pattern is duplicated across potentially hundreds of components in a large application.

#### 2. No Coordination Between Sibling Components

When a page contains multiple async components, each manages its own loading state independently:

```
Time 0ms:    [Spinner1] [Spinner2] [Spinner3] [Spinner4] [Spinner5]
Time 150ms:  [Accounts] [Spinner2] [Spinner3] [Spinner4] [Spinner5]
Time 300ms:  [Accounts] [Contacts] [Spinner3] [Spinner4] [Spinner5]
Time 400ms:  [Accounts] [Contacts] [  Chart ] [Spinner4] [Spinner5]
Time 500ms:  [Accounts] [Contacts] [  Chart ] [ Tasks  ] [Spinner5]
Time 600ms:  [Accounts] [Contacts] [  Chart ] [ Tasks  ] [Activity]
```

This creates:

- **Layout shift**: Components pop in at different times, causing the page to jump
- **Visual noise**: Multiple spinners competing for attention
- **Poor perceived performance**: Users see a janky, incomplete UI for extended periods

#### 3. Difficulty Implementing Skeleton Loading Patterns

Modern UX best practices favor skeleton screens over spinners. However, implementing coordinated skeleton loading across multiple components requires significant custom infrastructure:

```html
<!-- Current approach: Complex parent orchestration -->
<template>
  <template lwc:if="{isAnyChildLoading}">
    <c-page-skeleton></c-page-skeleton>
  </template>
  <template lwc:else>
    <c-account-list onloadingchange="{handleLoadingChange}"></c-account-list>
    <c-contact-panel onloadingchange="{handleLoadingChange}"></c-contact-panel>
    <c-opportunity-chart
      onloadingchange="{handleLoadingChange}"
    ></c-opportunity-chart>
  </template>
</template>
```

```javascript
// Parent must manually track all children's loading states
loadingStates = new Map();

handleLoadingChange(event) {
    this.loadingStates.set(event.target, event.detail.isLoading);
    this.isAnyChildLoading = [...this.loadingStates.values()].some(v => v);
}
```

This approach is fragile, verbose, and doesn't scale.

### Current Workarounds and Their Limitations

Developers have created several workarounds, each with significant drawbacks:

#### Workaround 1: Manual State Tracking

Every component tracks its own `isLoading`, `error`, and `data` states.

| Pros                 | Cons                             |
| -------------------- | -------------------------------- |
| Simple to understand | Massive boilerplate duplication  |
| No dependencies      | No coordination between siblings |
| Full control         | Easy to forget edge cases        |

#### Workaround 2: Base Class / Mixin Pattern

```javascript
// asyncAwareMixin.js
export const AsyncAwareMixin = (Base) =>
  class extends Base {
    _loadingCount = 0;

    get isLoading() {
      return this._loadingCount > 0;
    }

    startLoading() {
      this._loadingCount++;
    }
    finishLoading() {
      this._loadingCount = Math.max(0, this._loadingCount - 1);
    }
  };
```

| Pros                         | Cons                                                           |
| ---------------------------- | -------------------------------------------------------------- |
| Reduces some boilerplate     | Still requires manual `startLoading()`/`finishLoading()` calls |
| Consistent state management  | Template boilerplate remains                                   |
| Tracks concurrent operations | No parent-level coordination                                   |

#### Workaround 3: Wrapper Component Pattern

```html
<!-- asyncContainer.html -->
<template>
  <template lwc:if="{isLoading}">
    <lightning-spinner></lightning-spinner>
  </template>
  <template lwc:elseif="{hasError}">
    <c-error-panel errors="{error}"></c-error-panel>
  </template>
  <template lwc:else>
    <slot></slot>
  </template>
</template>
```

| Pros                         | Cons                                         |
| ---------------------------- | -------------------------------------------- |
| Centralizes loading/error UI | Parent still tracks state manually           |
| Consistent look and feel     | Just moves boilerplate, doesn't eliminate it |
| Reusable                     | No automatic detection of async state        |

#### Workaround 4: Lightning Message Service (LMS) Coordination

```javascript
// Child publishes its loading state
publish(this.messageContext, LOADING_CHANNEL, {
    componentId: this.componentId,
    isLoading: true
});

// Parent subscribes and aggregates
handleLoadingMessage({ componentId, isLoading }) {
    if (isLoading) {
        this.loadingComponents.add(componentId);
    } else {
        this.loadingComponents.delete(componentId);
    }
}
```

| Pros                              | Cons                              |
| --------------------------------- | --------------------------------- |
| Enables parent-level coordination | Significant complexity            |
| Works across unrelated components | Every child must manually publish |
| Decoupled architecture            | Race conditions possible          |

#### Workaround 5: State Management Libraries (Redux, Signals)

```javascript
import { ReduxElement } from "c/lwcRedux";

export default class AccountList extends ReduxElement {
  mapStateToProps(state) {
    return {
      accounts: state.accounts.data,
      isLoading: state.accounts.isLoading,
      error: state.accounts.error,
    };
  }
}
```

| Pros                               | Cons                      |
| ---------------------------------- | ------------------------- |
| Global state accessible everywhere | External dependency       |
| Single source of truth             | Learning curve            |
| Good for complex apps              | Overkill for simple cases |

### Comparison Summary

| Approach                        | Boilerplate | Coordination | Complexity | Automatic |
| ------------------------------- | ----------- | ------------ | ---------- | --------- |
| Manual state tracking           | High        | None         | Low        | No        |
| Mixin/Base class                | Medium      | None         | Medium     | No        |
| Wrapper component               | Medium      | None         | Low        | No        |
| LMS pub/sub                     | High        | Yes          | High       | No        |
| Redux/Signals                   | Medium      | Yes          | High       | No        |
| **Async Boundaries (proposed)** | **Low**     | **Yes**      | **Low**    | **Yes**   |

## Detailed design

### Core API

#### 1. The `<lwc:suspense>` Boundary

```html
<lwc:suspense timeout="5000">
  <slot name="fallback">
    <!-- Shown while children are loading -->
  </slot>
  <slot name="error">
    <!-- Shown if any child errors -->
  </slot>
  <slot name="timeout">
    <!-- Shown if timeout is exceeded -->
  </slot>
  <!-- Default slot: shown when all children are ready -->
</lwc:suspense>
```

**Attributes:**

- `timeout` (optional): Milliseconds before showing timeout slot (default: none)

**Slots:**

- `fallback`: UI shown while any child is suspended
- `error`: UI shown if any child encounters an error
- `timeout`: UI shown if timeout is exceeded
- default: UI shown when all children are resolved

#### 2. Automatic `@wire` Detection

The framework automatically detects suspended components using `@wire`:

```javascript
import { LightningElement, wire } from "lwc";
import getAccounts from "@salesforce/apex/AccountController.getAccounts";

export default class AccountList extends LightningElement {
  @wire(getAccounts)
  accounts; // Framework automatically tracks pending state
}
```

When `accounts` is pending:

- Component is marked as "suspended"
- Nearest `<lwc:suspense>` ancestor shows fallback
- When `accounts` resolves, component renders
- On error, nearest `<lwc:suspense>` shows error slot

#### 3. Explicit `suspend()` API

For imperative async operations:

```javascript
import { LightningElement } from "lwc";
import { suspend } from "lwc";

export default class DataFetcher extends LightningElement {
  data;

  async connectedCallback() {
    await suspend(this, async () => {
      const response = await fetch("/api/data");
      this.data = await response.json();
    });
  }
}
```

**Behavior:**

- Component is suspended until the promise resolves
- Errors are caught and propagated to nearest suspense boundary
- Multiple concurrent `suspend()` calls are tracked independently

#### 4. Component Suspense Context

Every LWC component automatically has access to its nearest suspense boundary through the `this.suspense` property. This property is `undefined` if the component is not wrapped in a `<lwc:suspense>` boundary.

```javascript
import { LightningElement } from "lwc";

export default class MyComponent extends LightningElement {
  connectedCallback() {
    if (this.suspense) {
      // Component is inside a suspense boundary
      console.log("Is suspended:", this.suspense.isSuspended);
      console.log("Has error:", this.suspense.hasError);
      console.log("Pending count:", this.suspense.pendingCount);
    } else {
      // Component is not wrapped in suspense
      console.log("No suspense boundary found");
    }
  }
}
```

**Available Properties:**

| Property       | Type      | Description                                                  |
| -------------- | --------- | ------------------------------------------------------------ |
| `isSuspended`  | `boolean` | `true` if the boundary is currently showing fallback content |
| `hasError`     | `boolean` | `true` if any child has errored                              |
| `pendingCount` | `number`  | Number of children currently suspended                       |
| `hasTimedOut`  | `boolean` | `true` if the timeout has been exceeded                      |

**Use Cases:**

1. **Conditional Behavior**: Components can adapt their behavior based on whether they're in a suspense boundary
2. **Debugging**: Log suspense state during development
3. **Analytics**: Track loading performance at the component level
4. **Progressive Enhancement**: Components can opt into simpler rendering when suspense is handling loading states

```javascript
// Example: Skip local loading state if suspense is handling it
export default class SmartComponent extends LightningElement {
  showLocalSpinner = false;

  @wire(getData)
  wiredData({ data, error }) {
    // Only show local spinner if not in suspense
    this.showLocalSpinner = !this.suspense && !data && !error;
  }
}
```

This property is read-only and automatically maintained by the framework. Components cannot modify the suspense boundary state through this property.

#### 5. Error Handling

Errors in suspended components are caught by the nearest `<lwc:suspense>` boundary:

```html
<template>
  <lwc:suspense>
    <c-spinner slot="fallback"></c-spinner>
    <c-error-display slot="error"></c-error-display>
    <c-data-component></c-data-component>
  </lwc:suspense>
</template>
```

```javascript
// errorDisplay.js
export default class ErrorDisplay extends LightningElement {
  @api error; // Automatically populated by framework

  handleRetry() {
    this.dispatchEvent(new CustomEvent("retry"));
  }
}
```

#### 6. Events

Suspense boundaries dispatch events at key lifecycle points:

```javascript
// parent.js
handleSuspenseChange(event) {
    console.log('Suspended:', event.detail.suspended);
    console.log('Pending count:', event.detail.pendingCount);
}

handleSuspenseResolved(event) {
    console.log('All children loaded!');
}

handleSuspenseError(event) {
    console.log('Error:', event.detail.error);
    console.log('Failed component:', event.detail.componentName);
}

handleSuspenseTimeout(event) {
    console.log('Timeout after', event.detail.timeout, 'ms');
}
```

```html
<!-- parent.html -->
<template>
  <lwc:suspense
    onsuspensechange="{handleSuspenseChange}"
    onsuspenseresolved="{handleSuspenseResolved}"
    onsuspenseerror="{handleSuspenseError}"
    onsusensetimeout="{handleSuspenseTimeout}"
  >
    <!-- children -->
  </lwc:suspense>
</template>
```

**Event Details:**

| Event              | Detail                                         | Description                          |
| ------------------ | ---------------------------------------------- | ------------------------------------ |
| `suspensechange`   | `{ suspended: boolean, pendingCount: number }` | Fired when suspension state changes  |
| `suspenseresolved` | `{}`                                           | Fired when all children are resolved |
| `suspenseerror`    | `{ error: Error, componentName: string }`      | Fired when a child errors            |
| `suspensetimeout`  | `{ timeout: number }`                          | Fired when timeout is exceeded       |

### Implementation Details

#### Suspension Detection

The framework maintains a suspension context for each suspense boundary:

```typescript
interface SuspensionContext {
  boundary: SuspenseBoundary;
  pendingChildren: Set<LightningElement>;
  errors: Map<LightningElement, Error>;
  resolved: boolean;
  timedOut: boolean;
}
```

When a component uses `@wire` or `suspend()`:

1. Component registers itself with nearest suspense ancestor
2. Suspense context adds component to `pendingChildren`
3. Boundary shows fallback slot
4. When component resolves, it's removed from `pendingChildren`
5. When `pendingChildren.size === 0`, boundary shows default slot

#### Nested Boundaries

Suspense boundaries can be nested for granular control:

```html
<template>
  <!-- Outer boundary: Skeleton for entire page -->
  <lwc:suspense>
    <c-page-skeleton slot="fallback"></c-page-skeleton>

    <div class="page">
      <!-- Header loads independently -->
      <lwc:suspense>
        <c-header-skeleton slot="fallback"></c-header-skeleton>
        <c-page-header></c-page-header>
      </lwc:suspense>

      <!-- Main content waits for all sections -->
      <lwc:suspense>
        <c-content-skeleton slot="fallback"></c-content-skeleton>
        <div class="content">
          <c-section-one></c-section-one>
          <c-section-two></c-section-two>
        </div>
      </lwc:suspense>
    </div>
  </lwc:suspense>
</template>
```

**Resolution order:**

1. Header resolves first → Header skeleton disappears, header appears
2. Sections resolve → Content skeleton disappears, sections appear
3. Outer boundary resolves last (after all children)

#### Interaction with Lightning Data Service (LDS)

LDS-backed components automatically participate in suspense:

```javascript
import { LightningElement, wire } from "lwc";
import { getRecord } from "lightning/uiRecordApi";

export default class AccountDetail extends LightningElement {
  @wire(getRecord, { recordId: "$recordId", fields: FIELDS })
  account; // Automatically suspends until LDS cache resolves
}
```

**Caching behavior:**

- If LDS has cached data → Component resolves immediately
- If LDS must fetch → Component suspends until fetch completes
- If LDS invalidates → Component can optionally re-suspend

#### Conditional Suspense

Suspense boundaries can be conditionally enabled:

```html
<template>
  <lwc:suspense lwc:if="{enableSuspense}">
    <c-spinner slot="fallback"></c-spinner>
    <c-content></c-content>
  </lwc:suspense>

  <c-content lwc:else></c-content>
</template>
```

This allows progressive migration or A/B testing.

### Advanced Use Cases

#### Progressive Enhancement

```html
<template>
  <!-- Fast: Show header immediately -->
  <c-header></c-header>

  <!-- Medium: Show sidebar after quick data fetch -->
  <lwc:suspense>
    <c-sidebar-skeleton slot="fallback"></c-sidebar-skeleton>
    <c-sidebar></c-sidebar>
  </lwc:suspense>

  <!-- Slow: Wait for heavy dashboard data -->
  <lwc:suspense timeout="3000">
    <c-dashboard-skeleton slot="fallback"></c-dashboard-skeleton>
    <c-timeout-message slot="timeout"></c-timeout-message>
    <c-dashboard></c-dashboard>
  </lwc:suspense>
</template>
```

#### Retry Pattern

```html
<!-- parent.html -->
<template>
  <lwc:suspense key="{suspenseKey}">
    <c-spinner slot="fallback"></c-spinner>
    <c-error-panel slot="error" onretry="{handleRetry}"> </c-error-panel>
    <c-data-component></c-data-component>
  </lwc:suspense>
</template>
```

```javascript
// parent.js
export default class Parent extends LightningElement {
  suspenseKey = 0;

  handleRetry() {
    // Changing key forces suspense boundary to reset
    this.suspenseKey++;
  }
}
```

#### Loading State Composition

```html
<template>
  <lwc:suspense>
    <!-- Fallback slot can itself contain suspense boundaries -->
    <div slot="fallback" class="skeleton-layout">
      <c-header-skeleton></c-header-skeleton>
      <lwc:suspense>
        <c-nav-skeleton slot="fallback"></c-nav-skeleton>
        <c-navigation></c-navigation>
      </lwc:suspense>
      <c-content-skeleton></c-content-skeleton>
    </div>

    <!-- Actual content -->
    <c-page-content></c-page-content>
  </lwc:suspense>
</template>
```

### Framework Integration

#### Wire Service Integration

The `@wire` decorator is enhanced to participate in suspense:

```javascript
// Before: Manual state tracking
@wire(getAccounts)
wiredAccounts({ error, data }) {
    this.isLoading = false;
    if (data) this.accounts = data;
    else if (error) this.error = error;
}

// After: Automatic suspension
@wire(getAccounts)
accounts;  // Framework handles loading/error/data states
```

#### Composition with Other LWC Features

**With `lwc:if`:**

```html
<lwc:suspense lwc:if="{showContent}">
  <c-spinner slot="fallback"></c-spinner>
  <c-content></c-content>
</lwc:suspense>
```

**With `iterator`:**

```html
<lwc:suspense>
  <c-list-skeleton slot="fallback"></c-list-skeleton>
  <template for:each="{items}" for:item="item">
    <c-item-card key="{item.id}" item="{item}"></c-item-card>
  </template>
</lwc:suspense>
```

**With `lwc:slot`:**

```html
<!-- parent.html -->
<c-async-wrapper>
  <c-custom-content></c-custom-content>
</c-async-wrapper>

<!-- asyncWrapper.html -->
<template>
  <lwc:suspense>
    <c-spinner slot="fallback"></c-spinner>
    <slot></slot>
    <!-- Slotted content participates in suspense -->
  </lwc:suspense>
</template>
```

### Testing Support

#### Jest Utilities

```javascript
import { createElement } from "lwc";
import { flushSuspense } from "@lwc/test-utils";
import MyComponent from "c/myComponent";

it("renders after suspension", async () => {
  const element = createElement("c-my-component", { is: MyComponent });
  document.body.appendChild(element);

  // Wait for suspense to resolve
  await flushSuspense(element);

  expect(element.shadowRoot.querySelector(".content")).not.toBeNull();
});

it("handles suspense errors", async () => {
  const element = createElement("c-my-component", { is: MyComponent });
  document.body.appendChild(element);

  // Force suspense to reject
  await expect(flushSuspense(element)).rejects.toThrow();

  expect(element.shadowRoot.querySelector(".error")).not.toBeNull();
});
```

### Backward Compatibility

This feature is **fully backward compatible**:

1. **Existing `@wire` code continues to work** unchanged
2. **Async `connectedCallback()` remains functional** (doesn't auto-suspend)
3. **Manual loading state management still works** within or outside suspense
4. **Opt-in only**: Components must be wrapped in `<lwc:suspense>` to participate

Migration can happen incrementally, component by component.

### Comparison with React and Vue

#### React Suspense

```jsx
// React
<Suspense fallback={<Spinner />}>
  <AsyncComponent />
</Suspense>;

// Component throws promise to suspend
function AsyncComponent() {
  const data = use(dataPromise); // Suspends until promise resolves
  return <div>{data}</div>;
}
```

#### Vue Suspense

```vue
<!-- Vue -->
<Suspense>
    <template #default>
        <AsyncComponent />
    </template>
    <template #fallback>
        <Spinner />
    </template>
</Suspense>

<script setup>
// Component suspends via async setup()
const data = await fetchData();
</script>
```

#### LWC Suspense (Proposed)

```html
<!-- LWC -->
<template>
  <lwc:suspense>
    <c-spinner slot="fallback"></c-spinner>
    <c-async-component></c-async-component>
  </lwc:suspense>
</template>
```

```javascript
// asyncComponent.js
import { LightningElement } from "lwc";
import { suspend } from "lwc";

export default class AsyncComponent extends LightningElement {
  data;

  async connectedCallback() {
    // Explicit suspend() triggers Suspense
    await suspend(this, async () => {
      this.data = await fetchData();
    });
  }
}
```

### Key Differences

1. **Detection Mechanism**: Vue uses implicit `async setup()`, LWC uses explicit `suspend()` or automatic `@wire` detection
2. **Explicitness**: Vue's suspension is implicit (any await in setup); LWC requires explicit opt-in for imperative async
3. **Error Handling**: Vue requires `onErrorCaptured` in parent; LWC provides dedicated `error` slot
4. **Timeout**: LWC adds built-in timeout support (Vue requires custom implementation)
5. **Platform Integration**: LWC version integrates with Salesforce-specific patterns like `@wire` and Lightning Data Service
6. **Backwards Compatibility**: LWC's explicit approach ensures existing `async connectedCallback()` code continues to work unchanged

## Drawbacks

### 1. Learning Curve

Developers must learn a new mental model for async handling. However, this is offset by the simplicity of the resulting code.

### 2. Migration Effort

Existing components won't automatically benefit. Developers must:

- Add `<lwc:suspense>` boundaries to parent components
- Potentially refactor components to remove manual loading state management

### 3. Debugging Complexity

When a suspense boundary isn't resolving, debugging which child is pending could be challenging. Mitigation:

- DevTools integration showing pending children
- Console warnings for long-pending states

### 4. Performance Considerations

The framework must track pending states across component boundaries. This adds overhead, though likely minimal compared to the network requests being awaited.

### 5. Not a Complete Solution

Suspense handles **initial** loading well but doesn't address:

- Refetch/refresh patterns
- Optimistic updates
- Stale-while-revalidate patterns

These would require additional features (potentially a separate RFC for reactive data fetching primitives).

## Alternatives

### Alternative 1: Do Nothing

Continue with current patterns. Developers use manual state tracking or community libraries.

**Impact**: Continued boilerplate, inconsistent implementations, poor coordination.

### Alternative 2: Enhanced Wire Decorator

Add loading/error states directly to `@wire` return values:

```javascript
@wire(getAccounts)
accounts;  // { data, error, pending }
```

**Limitation**: Still requires manual template conditionals; no parent coordination.

### Alternative 3: Higher-Order Component Pattern

Provide a wrapper component that handles async for a single child:

```html
<c-async-loader>
  <c-my-component slot="content"></c-my-component>
  <c-spinner slot="loading"></c-spinner>
</c-async-loader>
```

**Limitation**: Doesn't coordinate across siblings; adds nesting depth.

### Alternative 4: Directive-Based Approach

```html
<div lwc:suspense="loading">
  <c-spinner lwc:suspense-fallback></c-spinner>
  <c-content lwc:suspense-content></c-content>
</div>
```

**Limitation**: More verbose; less clear semantic meaning.

## Adoption strategy

### Phase 1: Opt-in Feature

Release behind a feature flag. Early adopters can experiment and provide feedback.

### Phase 2: Documentation and Patterns

Publish best practices for:

- When to use suspense boundaries
- Granularity of boundaries
- Error handling patterns
- Testing suspended components

### Phase 3: Tooling Support

- VS Code extension shows suspense boundaries in component tree
- Chrome DevTools extension shows pending/resolved state
- Jest utilities for testing suspended components

### Phase 4: General Availability

Remove feature flag. Consider deprecation warnings for common anti-patterns (e.g., manual `isLoading` tracking inside suspense boundaries).

## How we teach this

### Terminology

- **Async Boundary**: A `<lwc:suspense>` element that coordinates loading states
- **Suspended Component**: A component with pending async dependencies
- **Fallback Content**: UI shown while children are suspended
- **Resolved**: State when all async dependencies have completed

### Example Curriculum

1. **Basic Usage**: Single component with fallback
2. **Multiple Children**: Coordinating several async components
3. **Nested Boundaries**: Granular loading control
4. **Error Handling**: Graceful degradation
5. **Advanced Patterns**: Timeout, events, conditional boundaries

### Migration Guide

Before:

```javascript
// accountList.js
export default class AccountList extends LightningElement {
  accounts;
  error;
  isLoading = true;

  @wire(getAccounts)
  wiredAccounts({ error, data }) {
    this.isLoading = false;
    if (data) this.accounts = data;
    else if (error) this.error = error;
  }
}
```

```html
<!-- accountList.html -->
<template>
  <template lwc:if="{isLoading}">
    <lightning-spinner></lightning-spinner>
  </template>
  <template lwc:elseif="{error}">
    <c-error-panel></c-error-panel>
  </template>
  <template lwc:else>
    <!-- content -->
  </template>
</template>
```

After:

```javascript
// accountList.js
export default class AccountList extends LightningElement {
  @wire(getAccounts)
  accounts;
}
```

```html
<!-- accountList.html -->
<template>
  <template lwc:if="{accounts.data}">
    <!-- content -->
  </template>
  <template lwc:else>
    <p>No accounts found.</p>
  </template>
</template>
```

```html
<!-- parent.html -->
<template>
  <lwc:suspense>
    <lightning-spinner slot="fallback"></lightning-spinner>
    <c-error-panel slot="error"></c-error-panel>
    <c-account-list></c-account-list>
  </lwc:suspense>
</template>
```

## Unresolved questions

1. **Server-Side Rendering**: How should suspense interact with LWC SSR? Should the server await all suspended components before sending HTML?

2. **Streaming**: Could suspense enable streaming HTML responses, sending fallback content immediately and replacing with resolved content?

3. **Retry Mechanism**: Should suspense boundaries support automatic retry on error?

4. **Transition Support**: Should we integrate with a future animation/transition system?

5. **DevTools**: What debugging information should be exposed in browser DevTools?

6. **Imperative API**: Beyond `suspend()`, are there other imperative needs (e.g., manually triggering re-suspension)?

7. **Lightning Locker/LWS**: Are there security considerations for suspense boundaries crossing namespace boundaries?

## Future Possibilities

1. **Suspense + Transitions**: Animate between fallback and resolved states
2. **Suspense + Streaming SSR**: Progressive HTML delivery
3. **Data Fetching Primitives**: `useFetch`, `useAsyncData` composables (similar to Nuxt)
4. **Stale-While-Revalidate**: Show cached data while revalidating in background
5. **Prefetching**: Trigger data fetching before navigation completes

---

## References

- [Vue 3 Suspense Documentation](https://vuejs.org/guide/built-ins/suspense.html)
- [React Suspense Documentation](https://react.dev/reference/react/Suspense)
- [Nuxt Data Fetching](https://nuxt.com/docs/getting-started/data-fetching)
- [LWC Wire Service](https://developer.salesforce.com/docs/platform/lwc/guide/data-wire-service-about.html)
