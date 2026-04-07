# Agent Prompt: Unified Draggable — Droppable-Owned Sorting

## Context & Repository

You are working in the fork **`spawnxe/dnd-kit-unified-draggable`** (forked from `clauderic/dnd-kit`). This is a TypeScript monorepo with the following relevant packages:

- `packages/abstract` — framework-agnostic core: `Entity`, `Draggable`, `Droppable`, `DragDropManager`, `DragDropRegistry`, plugins, sensors, modifiers
- `packages/dom` — DOM layer extending abstract: `Draggable`, `Droppable`, `Sortable` (which composes a `SortableDraggable` + `SortableDroppable`), `OptimisticSortingPlugin`, `SortableKeyboardPlugin`
- `packages/react` — React bindings: `useSortable`, `useDraggable`, `useDroppable`
- `packages/solid`, `packages/svelte`, `packages/vue` — other framework bindings

---

## The Problem with the Current Architecture

In the current system:

1. **`Sortable`** (`packages/dom/src/sortable/sortable.ts`) is a first-class composite entity that internally creates both a `SortableDraggable` and a `SortableDroppable`. It owns `index` and `group`.
2. **Sorting behaviour is item-owned**: each item that wants to be sortable must be declared as a `Sortable`, not a plain `Draggable`. The droppable container is passive.
3. **`OptimisticSortingPlugin`** (`packages/dom/src/sortable/plugins/OptimisticSortingPlugin.ts`) detects `SortableDroppable` instances in the registry by `instanceof` check to find sortable groups, re-orders DOM elements, and updates indices.
4. **`group`** is a property of each `Sortable` item — the container does not know whether its children are sortable.
5. This means: sorting behaviour, accepted types, and grouping are split across items and containers with no single source of truth.

---

## The Goal: Droppable-Owned Sorting

Redesign the system so that **the `Droppable` (container) is the authority** over whether its contents are sorted, what types it accepts, and how items are ordered. A draggable item dropped into a sortable droppable automatically participates in sorting — no `Sortable` composite entity is needed at the call site.

**Core principle:** `sortable` is a *behavioural mode* of a `Droppable`, not a separate entity type.

---

## Desired API (Target State)

### DOM / framework-agnostic layer

```typescript
// A container declares itself sortable and what it accepts
const list = new Droppable({
  id: 'list-1',
  sortable: true,               // NEW: opt-in to sorting behaviour
  accept: ['card'],             // only items of type 'card' are accepted &amp; sorted
  collisionDetector: closestCenter,
});

// Items are plain Draggables — no Sortable wrapper needed
const item = new Draggable({
  id: 'item-a',
  type: 'card',
});
```

### React layer

```tsx
// Container
function SortableList({ items }) {
  const { ref } = useDroppable({
    id: 'list-1',
    sortable: true,
    accept: ['card'],
  });
  return <ul>{items.map(...)}</ul>;
}

// Item — plain useDraggable, no useSortable needed
function Card({ id }) {
  const { ref, isDragging } = useDraggable({ id, type: 'card' });
  return <li>...</li>;
}
```

> **Backwards compatibility note:** `useSortable` / `Sortable` may remain as a convenience wrapper but must be re-implemented on top of the new primitives rather than being the canonical path.

---

## Implementation Plan

Work through the following steps **in order**, committing each step as an atomic commit on the `main` branch of `spawnxe/dnd-kit-unified-draggable`.

### Step 1 — Extend `Droppable` abstract interface (`packages/abstract`)

- Add `sortable?: boolean` to `Input<T>` in `packages/abstract/src/core/entities/droppable/droppable.ts`
- Add `@reactive public accessor sortable: boolean` to the `Droppable` class
- Add `sortingStrategy?: SortingStrategy` (type to be defined) to `Input<T>` — this allows per-droppable control of how items are re-ordered (e.g. vertical list, horizontal list, grid)
- Export `SortingStrategy` type from `packages/abstract/src/core/index.ts`

### Step 2 — Extend `Droppable` DOM layer (`packages/dom`)

- Extend `packages/dom/src/core/entities/droppable/droppable.ts` to forward `sortable` and `sortingStrategy` from input to super
- A `Droppable` with `sortable: true` should track the ordered list of `Draggable` children currently registered inside it. This ordered list is the source of truth for indices.
- Add a `children: Draggable[]` reactive property to DOM `Droppable`, populated by observing `manager.registry.draggables` and filtering by a spatial or explicit parent relationship.

### Step 3 — Introduce `DroppableSortingPlugin` (`packages/dom`)

- Create `packages/dom/src/core/plugins/DroppableSortingPlugin.ts`
- This plugin replaces `OptimisticSortingPlugin` as the canonical sorting engine
- On `dragover`: iterate `manager.registry.droppables`, find all with `sortable === true` that `accept` the current drag source's `type`. For each, compute the new order by running the droppable's `sortingStrategy` against current child positions.
- Reorder DOM children of the sortable droppable and update a stable index signal per draggable.
- On `dragend` (cancelled): restore original order.
- The plugin must **not** use `instanceof SortableDroppable` — it must use `droppable.sortable === true` duck-typing.

### Step 4 — Remove `Sortable` as a required primitive

- `packages/dom/src/sortable/sortable.ts` — `Sortable`, `SortableDraggable`, `SortableDroppable` should be **deprecated** (keep for backwards compatibility) but re-implemented as thin wrappers:
  - `Sortable` constructs a plain `Draggable` + sets `droppable.sortable = true` on the nearest registered parent `Droppable` (or registers a synthetic one)
  - `SortableDroppable` becomes an alias for `Droppable` with `sortable: true`
  - `SortableDraggable` becomes an alias for plain `Draggable`
- `OptimisticSortingPlugin` should delegate to `DroppableSortingPlugin` or be retired

### Step 5 — Update React bindings (`packages/react`)

- `useDroppable` — accept and forward `sortable` and `sortingStrategy`
- `useDraggable` — no change required; items gain sort participation automatically by being dropped into a sortable droppable
- `useSortable` — re-implement as sugar over `useDraggable` + a sortable `useDroppable`:

```typescript
export function useSortable(input) {
  const draggable = useDraggable({ ...input });
  const droppable = useDroppable({ ...input, sortable: true });
  return { ...draggable, ...droppable };
}
```

### Step 6 — Update Solid, Svelte, Vue bindings

- Mirror the React changes in `packages/solid`, `packages/svelte`, `packages/vue` — add `sortable` prop forwarding to their respective droppable primitives.

### Step 7 — Tests & Stories

- Add unit tests for `DroppableSortingPlugin` covering:
  - Single sortable list reorder
  - Cross-list transfer between two sortable droppables
  - Type filtering: non-accepted types must not trigger sorting
  - Cancelled drag restores original order
- Update or add Storybook stories demonstrating the new `useDroppable({ sortable: true })` pattern without `useSortable`

---

## Key Constraints

- **Do not break the existing public API.** All existing `useSortable` / `Sortable` usage must continue to work.
- **Type safety must be maintained throughout.** No `any` escapes in the new code paths.
- **Reactive state** must use the existing `@dnd-kit/state` primitives (`@reactive`, `@derived`, `signal`, `batch`). Do not introduce new state management.
- **No new dependencies.** Work only within the existing monorepo package graph.
- **The `accept` filter on the droppable is the single gate** for both drop acceptance and sort participation — if a type is not accepted, it is neither dropped nor sorted into the container.
- All new types and interfaces must be exported from the appropriate `index.ts` barrel files.
- Commit messages should follow the pattern: `feat(dom): ...`, `feat(abstract): ...`, `refactor(react): ...`

---

## Files to Focus On First

| File | Reason |
|---|---|
| `packages/abstract/src/core/entities/droppable/droppable.ts` | Add `sortable` + `sortingStrategy` to the base contract |
| `packages/dom/src/core/entities/droppable/droppable.ts` | DOM-layer droppable, add children tracking |
| `packages/dom/src/sortable/plugins/OptimisticSortingPlugin.ts` | Reference implementation to understand existing sort logic |
| `packages/dom/src/sortable/sortable.ts` | Understand the `SortableDraggable`/`SortableDroppable` composition to deprecate correctly |
| `packages/react/src/sortable/useSortable.ts` | React entrypoint to refactor |
| `packages/abstract/src/core/manager/registry.ts` | Understand how draggables/droppables are queried — needed for the plugin |

---

## Architecture Decision Summary

| Concern | Current Owner | New Owner |
|---|---|---|
| Is content sortable? | Each item (`Sortable`) | Container (`Droppable.sortable`) |
| What types are sorted? | Each item (`Sortable.type`) | Container (`Droppable.accept`) |
| What group does an item belong to? | Each item (`Sortable.group`) | Container identity (`Droppable.id`) |
| Sort order / index | Each item (`Sortable.index`) | Container (`Droppable.children[]`) |
| Sorting algorithm | Plugin via `instanceof` check | Plugin via `droppable.sortable` + `droppable.sortingStrategy` |
