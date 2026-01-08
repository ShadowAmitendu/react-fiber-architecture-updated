# React Fiber Architecture (Updated)

## Introduction

React Fiber is the core reconciliation architecture that underpins modern React. Originally introduced as a ground‑up rewrite of React’s reconciler (and shipped in React 16), Fiber enables incremental rendering, scheduling, and concurrency. Today, Fiber is not a future concept—it is the production engine behind features such as Concurrent Rendering, Suspense, streaming server rendering, Server Components, and the React 19 Actions model.

The fundamental goal of Fiber remains unchanged: **to make React better suited for complex, interactive user interfaces** by allowing rendering work to be split, prioritized, paused, resumed, or abandoned as conditions change.

---

## Goals of React Fiber

Fiber was designed to enable React to:

* Split rendering work into incremental units
* Assign priority to different updates
* Pause, resume, or abandon work
* Reuse previously completed work
* Support concurrency and interruption

While these were initially internal goals, they are now observable through public APIs such as `startTransition`, Suspense, streaming SSR, and Actions.

---

## About This Document

This document explains React Fiber conceptually rather than by walking through the source code. Fiber is an internal implementation detail, but understanding it provides clarity on why modern React behaves the way it does—especially under concurrent rendering.

This is not an official React document. The intent is to provide an accurate mental model aligned with the current React architecture (React 18 and React 19+).

---

## Prerequisites

Before reading further, familiarity with the following concepts is strongly recommended:

* React Components vs Elements vs Instances
* React Reconciliation
* React’s declarative rendering model
* React Design Principles (especially scheduling)

---

## Review of Core Concepts

### What Is Reconciliation?

**Reconciliation** is the algorithm React uses to compare one render tree with another and determine what changes are required.

An **update** is a change to the data used to render the application (commonly via `setState`, props, or context). React treats every update as if it could re-render the entire application, even though in practice it optimizes heavily.

Key reconciliation rules:

* Different component types produce different trees and are not diffed
* Lists are diffed using keys
* Keys must be stable, predictable, and unique

Fiber reimplements reconciliation but preserves these high‑level rules.

---

### Reconciliation vs Rendering

Reconciliation and rendering are separate phases:

* **Reconciliation** determines what changed
* **Rendering** applies those changes to a target environment

This separation allows React to support multiple renderers (DOM, Native, custom renderers) while sharing the same Fiber reconciler.

Fiber primarily replaces the reconciler. Renderers adapt to consume Fiber’s output.

---

## Scheduling

**Scheduling** is the process of deciding *when* work should be performed.

In user interfaces:

* Not all updates must be applied immediately
* Some updates are more important than others
* Blocking the main thread causes dropped frames and poor UX

React uses a **pull‑based model**, meaning it decides *when* to process updates rather than executing them immediately when data changes.

### Modern Scheduling (React 18+)

Fiber’s scheduler enables:

* Automatic batching across async boundaries
* Interruptible rendering
* Priority‑based updates
* Cooperative multitasking with the browser

Public APIs exposing this behavior include:

* `createRoot`
* `startTransition` / `useTransition`
* `useDeferredValue`

---

## What Is a Fiber?

A **fiber** is a JavaScript object that represents a unit of work.

Conceptually:

* A fiber is similar to a **stack frame**
* It represents the execution of a component
* It can be paused, resumed, cloned, or discarded

Traditional call stacks cannot be interrupted. Fiber replaces the call stack with a data structure React can control.

Each component instance is associated with at most two fibers:

* The **current** (committed) fiber
* The **work‑in‑progress** fiber

---

## Fiber as a Virtual Stack Frame

Rendering a React application is analogous to evaluating:

```
v = f(d)
```

Where rendering involves nested function calls. Fiber reifies each of these calls into an object stored in memory, allowing React to:

* Pause execution
* Resume later
* Switch priorities
* Reuse completed work

This capability is essential for concurrency and scheduling.

---

## Structure of a Fiber

A fiber contains metadata about a component, its inputs, and its outputs.

### Key Fields

#### type and key

* `type` identifies the component (function, class, or host string)
* `key` is used for reconciliation

#### child and sibling

* Fibers form a tree via `child` and `sibling`
* Children are stored as a singly linked list

#### return

* Points to the parent fiber
* Equivalent to a return address in a stack frame

#### pendingProps and memoizedProps

* `pendingProps`: incoming props for the next render
* `memoizedProps`: props from the last completed render
* Equality allows React to skip work

#### priority (lanes)

* Modern React uses **lanes** instead of a single priority number
* Lanes allow multiple priority levels to coexist

#### alternate

* Links the current fiber to its work‑in‑progress version
* Enables efficient cloning and reuse

#### output

* Represents the rendered result
* Created by host components and passed upward

---

## Lanes and Priority (Modern Model)

Earlier Fiber implementations used numeric priorities. Modern React uses **lanes**:

* A lane represents a category of work
* Multiple lanes can be pending simultaneously
* Enables fine‑grained prioritization and batching

Examples include:

* Sync updates
* Input‑continuous updates
* Transitions
* Idle work

The scheduler selects the highest‑priority lane available.

---

## Commit Phase

Once reconciliation completes:

1. React enters the **commit phase**
2. Changes are applied to the renderer (DOM, Native, etc.)
3. Effects and lifecycle hooks run

The commit phase is **not interruptible**. Fiber ensures only safe, minimal work reaches this stage.

---

## Fiber and Concurrency

Fiber enables concurrent rendering, meaning:

* Rendering work may be interrupted
* A render may never commit
* Multiple renders may be in flight

This is why render functions must be:

* Pure
* Free of side effects
* Idempotent

Side effects belong in effects or event handlers.

---

## Suspense and Streaming

Fiber allows React to:

* Suspend rendering when data is unavailable
* Resume rendering when promises resolve
* Stream HTML and component payloads incrementally

Suspense boundaries act as scheduling checkpoints within the fiber tree.

---

## Server Components

Server Components extend Fiber across the client–server boundary:

* Some fibers execute on the server
* Their output is serialized and streamed
* Client fibers hydrate and continue rendering

This model reduces client bundle size and leverages server resources, while still relying on Fiber’s reconciliation and scheduling.

---

## React 19 and Actions

React 19 introduces **Actions**, built on Fiber’s scheduling model:

* Declarative async mutations
* Automatic pending, error, and optimistic state
* Integrated with forms and transitions

Actions reduce boilerplate while remaining compatible with concurrent rendering.

---

## Error Boundaries

Because fibers are explicit execution units:

* Errors can be captured at specific boundaries
* Failed subtrees can be replaced without crashing the app

This capability is a direct consequence of Fiber’s architecture.

---

## Key Takeaways

* Fiber is React’s internal execution model
* It replaces the call stack with a controllable data structure
* It enables scheduling, concurrency, and interruption
* Modern React features are built on Fiber
* Understanding Fiber explains *why* React behaves the way it does

---

## Conclusion

React Fiber is no longer experimental—it is the foundation of React. From concurrent rendering to server components and Actions, Fiber enables React to scale across devices, network conditions, and interaction patterns. While application developers rarely interact with fibers directly, understanding the architecture provides deep insight into React’s design decisions and best practices.
