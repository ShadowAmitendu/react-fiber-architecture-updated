# React Fiber Architecture

## Overview

React Fiber is the core reconciliation architecture that powers modern React. Introduced as a complete rewrite of React’s original reconciler and shipped in React 16, Fiber fundamentally changed how React performs rendering work. Rather than rendering the entire component tree in a single, uninterrupted pass, Fiber enables incremental rendering, fine-grained scheduling, and controlled concurrency.

Today, Fiber is not an experimental concept. It is the production foundation behind key React features such as Concurrent Rendering, Suspense, streaming server-side rendering (SSR), Server Components, and the Actions model introduced in React 19.

At its core, Fiber exists to make React better suited for building complex, highly interactive user interfaces by allowing rendering work to be split, prioritized, paused, resumed, or discarded as application state and user interactions evolve.

---

## Design Goals of React Fiber

React Fiber was designed to address limitations in the original stack-based reconciler. Its primary goals include:

* Breaking rendering work into small, incremental units
* Assigning priority to different types of updates
* Pausing, resuming, or abandoning in-progress work
* Reusing previously completed work when possible
* Enabling safe concurrency and interruption

Although these goals were initially internal implementation concerns, they are now exposed through public APIs such as `startTransition`, Suspense, streaming SSR, and Actions.

---

## About This Document

This document presents a conceptual and architectural explanation of React Fiber rather than a walkthrough of the source code. Fiber is an internal abstraction, but understanding it provides important insight into why modern React behaves the way it does—particularly in concurrent rendering scenarios.

This is not an official React document. Its purpose is to present a clear and accurate mental model aligned with the current React architecture (React 18 and React 19+).

---

## Prerequisites

To fully benefit from this document, readers should be familiar with the following concepts:

* The distinction between React components, elements, and instances
* The basics of React reconciliation
* React’s declarative rendering model
* React design principles, especially scheduling

---

## Core Concepts Review

### Reconciliation

**Reconciliation** is the algorithm React uses to compare two render trees—typically the previous tree and a newly generated one—and determine the minimal set of changes required to update the UI.

An **update** is any change to the data used to render an application, commonly triggered by `setState`, new props, or context updates. Conceptually, React treats each update as if it could re-render the entire application, while internally applying optimizations to limit actual work.

Key reconciliation assumptions include:

* Different component types produce fundamentally different trees and are replaced rather than diffed
* Lists are reconciled using keys
* Keys must be stable, predictable, and unique

Fiber preserves these reconciliation rules while reimplementing the underlying algorithm.

---

### Reconciliation vs Rendering

Reconciliation and rendering are distinct phases in React’s architecture:

* **Reconciliation** computes what has changed
* **Rendering** applies those changes to a specific target environment

Because these phases are decoupled, React can support multiple renderers (such as React DOM and React Native) that share the same Fiber reconciler. Fiber replaces the reconciler, while renderers are adapted to consume Fiber’s output.

---

## Scheduling

**Scheduling** refers to deciding when rendering work should be performed.

In user interface applications:

* Not all updates need to be applied immediately
* Some updates are more important than others (for example, user input vs. background data loading)
* Blocking the main thread can lead to dropped frames and poor user experience

React adopts a **pull-based scheduling model**, meaning the framework determines when work should be executed rather than immediately reacting to every data change.

### Modern Scheduling in React 18+

Fiber’s scheduler enables several important capabilities:

* Automatic batching across synchronous and asynchronous boundaries
* Interruptible rendering
* Priority-based update handling
* Cooperative multitasking with the browser

These capabilities are exposed through APIs such as:

* `createRoot`
* `startTransition` and `useTransition`
* `useDeferredValue`

#### Example: Priority-based Updates with `startTransition`

```jsx
import { useState, startTransition } from "react";

function SearchExample() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  function handleChange(e) {
    const value = e.target.value;
    setQuery(value); // high-priority update

    startTransition(() => {
      setResults(expensiveSearch(value)); // low-priority update
    });
  }

  return <input value={query} onChange={handleChange} />;
}
```

This example shows how Fiber allows urgent updates (typing) to remain responsive while deferring expensive rendering work.

#### ## What Is a Fiber?

A **fiber** is a JavaScript object that represents a unit of work in React.

Conceptually, a fiber:

* Is analogous to a stack frame
* Represents the execution of a single component
* Can be paused, resumed, cloned, or discarded

Traditional JavaScript call stacks cannot be interrupted. Fiber replaces the call stack with a data structure that React can fully control, enabling advanced scheduling behavior.

Each component instance corresponds to at most two fibers at any given time:

* The **current fiber**, which represents the committed UI
* The **work-in-progress fiber**, which represents ongoing rendering work

#### Example: One Component, Multiple Fibers (Conceptual)

```jsx
function Counter({ value }) {
  return <h1>{value}</h1>;
}
```

When `value` changes, React creates a work-in-progress fiber for `Counter`. If rendering completes successfully, it replaces the current fiber during the commit phase.st two fibers at any given time:

* The **current fiber**, which represents the committed UI
* The **work-in-progress fiber**, which represents ongoing rendering work

---

## Fiber as a Virtual Stack Frame

Rendering a React application can be modeled as evaluating a function:

```
v = f(d)
```

Where rendering involves nested function calls. Fiber materializes each of these calls as an object stored in memory, allowing React to pause execution, resume it later, switch priorities, and reuse completed work. This abstraction is fundamental to concurrency and scheduling.

---

## Structure of a Fiber

A fiber stores metadata about a component, its inputs, and its outputs.

### Important Fields

#### type and key

* `type` identifies the component (function, class, or host element)
* `key` is used during reconciliation to preserve identity

#### child and sibling

* Fibers form a tree using `child` and `sibling` references
* Children are stored as a singly linked list

#### return

* Points to the parent fiber
* Conceptually equivalent to a return address in a call stack

#### pendingProps and memoizedProps

* `pendingProps` represent incoming props for the next render
* `memoizedProps` represent props from the last completed render
* Equality checks allow React to skip unnecessary work

#### priority and lanes

* Modern React uses **lanes** instead of a single priority value
* Lanes allow multiple priority levels to coexist

#### alternate

* Connects the current fiber with its work-in-progress cou## Lanes and Priority
  Earlier versions of Fiber used numeric priorities. Modern React uses **lanes**, which represent categories of work. Multiple lanes can be active simultaneously, allowing React to batch updates and select the highest-priority work at any given time.

Common lane categories include:

* Synchronous updates (user input)
* Input-continuous updates (typing, scrolling)
* Transitions (navigation, filtering)
* Idle work (background tasks)

#### Example: Transition Lane

```jsx
import { useState, useTransition } from "react";

function ListFilter({ items }) {
  const [filter, setFilter] = useState("");
  const [isPending, startTransition] = useTransition();

  function handleFilterChange(e) {
    startTransition(() => {
      setFilter(e.target.value);
    });
  }

  return (
    <>
      <input onChange={handleFilterChange} />
      {isPending && <p>Updating…</p>}
      <FilteredList items={items} filter={filter} />
    </>
  );
}
```

The filter update runs in a transition lane, allowing higher-priority updates to interrupt it if needed.s and select the highest-priority work at any given time.

Common lane categories include:

* Synchronous updates
* Input-continuous updates
* Transitions
* Idle work

---

## Commit Phase

After reconciliation completes, React enters the **commit phase**:

1. Changes are applied to the renderer (DOM, Native, etc.)
2. Effects and lifecycle hooks are executed

The commit phase is not interruptible. Fiber ensures that only minimal, safe work reaches this stage.

---

## Fiber and Concurrency

Fiber enables concurrent rendering, meaning:

* Rendering work may be interrupted
* A render may never commit
* Multiple renders may be ## Suspense and Streaming
  Fiber allows React to suspend rendering when data is unavailable, resume rendering when promises resolve, and stream HTML and component payloads incrementally. Suspense boundaries act as scheduling checkpoints within the fiber tree.

#### Example: Suspense with Data Fetching

```jsx
import { Suspense } from "react";

function App() {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <UserProfile />
    </Suspense>
  );
}
```

When `UserProfile` suspends, Fiber pauses rendering of that subtree and shows the fallback until the data is ready.resume rendering when promises resolve, and stream HTML and component payloads incrementally. Suspense boundaries act as scheduling checkpoints within the fiber tree.

---

## Server Components

Server Components extend Fiber across the client–server boundary:

* Some fibers execute on the server
* Their output is serialized and streamed to the client
* Client fibers hydrate and continue rendering

This approach reduces client bundle size while preserving React’s reconciliation and scheduling model.

---

## React 19 and Actions

React 19 introduces **Actions**, which are built on top of Fiber’s scheduling system. Actions provide:

* Declarative asynchronous mutations
* Automatic handling of p## Error Boundaries
  Because fibers represent explicit execution units, React can capture errors at specific boundaries. Failed subtrees can be replaced without crashing the entire application, enabling resilient error recovery.

#### Example: Error Boundary

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}
```

If a child component throws during rendering, only that subtree is replaced, not the entire app.on units, React can capture errors at specific boundaries. Failed subtrees can be replaced without crashing the entire application, enabling resilient error recovery.

---

## Key Takeaways

* Fiber is React’s internal execution and reconciliation model
* It replaces the traditional call stack with a controllable structure
* It enables scheduling, concurrency, and interruption
* Most modern React features are built on top of Fiber
* Understanding Fiber explains many modern React behaviors

---

## Conclusion

React Fiber is the architectural foundation of modern React. By enabling incremental rendering, priority-based scheduling, and concurrency, Fiber allows React to scale across devices, network conditions, and interaction patterns. While application developers rarely interact with fibers directly, understanding the architecture provides valuable insight into React’s design decisions and best practices.
