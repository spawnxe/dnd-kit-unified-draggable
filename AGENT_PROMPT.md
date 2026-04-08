# Agent Prompt: Unified Draggable — Droppable-Owned Sorting

## Context & Repository

You are working in the fork **`spawnxe/dnd-kit-unified-draggable`** (forked from `clauderic/dnd-kit`). This is a TypeScript monorepo with the following relevant packages:

- `packages/abstract` — framework-agnostic core: `Entity`, `Draggable`, `Droppable`, `DragDropManager`, `DragDropRegistry`, plugins, sensors, modifiers
- `packages/dom` — DOM layer extending abstract: `Draggable`, `Droppable`, `Sortable` (which composes a `SortableDraggable` + `SortableDroppable`), `OptimisticSortingPlugin`, `SortableKeyboardPlugin`
- `packages/react` — React bindings: `useSortable`, `useDraggable`, `useDroppable`
- `packages/solid`, `packages/svelte`, `packages/vue` — other framework bindings

---

## Understanding the Current Architecture

Read the following files to orient yourself before writing any code:

| File | What to understand |
|---|---|
| `packages/abstract/src/core/entities/droppable/droppable.ts` | Base `Droppable` class: `accept`, `accepts()`, `collisionDetector`, `collisionPriority`, `type`, `shape`, `isDropTarget` |
| `packages/abstract/src/core/entities/draggable/draggable.ts` | Base `Draggable` class: `type`, `sensors`, `modifiers`, `status`, reactive helpers |
| `packages/abstract/src/core/entities/entity/entity.ts` | `Entity` base: `id`, `data`, `disabled`, `effects()`, `register/unregister/destroy` |
| `packages/abstract/src/core/entities/entity/registry.ts` | `EntityRegistry<T>`: signal-backed `Map<id, entity>`; signals reactive iteration; use `.value` for reactive read |
| `packages/abstract/src/core/manager/registry.ts` | `DragDropRegistry`: holds `draggables`, `droppables`, `plugins`, `sensors`, `modifiers` |
| `packages/dom/src/core/entities/droppable/droppable.ts` | DOM-layer `Droppable`: adds `element`, `proxy`, `refreshShape()`; `collisionDetector` defaults to `defaultCollisionDetection` |
| `packages/dom/src/core/entities/draggable/draggable.ts` | DOM-layer `Draggable`: adds `element`, `handle`, sensor binding in effects |
| `packages/dom/src/sortable/sortable.ts` | `Sortable` composite: constructs `SortableDraggable` + `SortableDroppable`; owns `index`, `group`, `transition`, `animate()` |
| `packages/dom/src/sortable/plugins/OptimisticSortingPlugin.ts` | Current sorting engine: detects `SortableDroppable` via `instanceof`; on `dragover` calls `move()` from `@dnd-kit/helpers` and reorders DOM; on cancelled `dragend` restores original order |
| `packages/dom/src/sortable/plugins/SortableKeyboardPlugin.ts` | Keyboard sorting: gated on `isSortable(source)` instanceof check; navigates between sortable droppables |
| `packages/dom/src/sortable/utilities.ts` | `isSortable()`, `isSortableOperation()` — both rely on `instanceof SortableDroppable`/`SortableDraggable` |
| `packages/react/src/core/droppable/useDroppable.ts` | React `useDroppable`: creates `Droppable`, wires reactive updates |
| `packages/react/src/sortable/useSortable.ts` | React `useSortable`: creates `Sortable`, wires all reactive updates |

### Key observations about the current code

1. **`Sortable`** is a *composite*, not an entity — it owns `index` and `group`, and delegates to a `SortableDraggable` and a `SortableDroppable` it creates internally.
2. **`OptimisticSortingPlugin`** discovers sortable groups by iterating `manager.registry.droppables` and checking `droppable instanceof SortableDroppable`. Each `SortableDroppable` has a `.sortable` back-reference to the parent `Sortable`, from which the plugin reads `.group` and `.index`.
3. **`SortableKeyboardPlugin`** similarly uses `isSortable(source)` (an `instanceof` check) to decide whether to activate keyboard sorting, and reads `source.sortable.index` / `source.sortable.group` directly.
4. **`group`** is a free-form `UniqueIdentifier` on each `Sortable` item — the container `Droppable` has no direct concept of grouping or sortability.
5. **`move()`** from `@dnd-kit/helpers` is the canonical array-reorder utility used by `OptimisticSortingPlugin`. Its signature takes a state object keyed by group name and a drag event.
6. **Reactive primitives**: all mutable state uses `@reactive accessor` (which wraps a signal under the hood); computed derived values use `@derived get`; side-effectful observation uses `effects()` on the entity. The `batch()` utility from `@dnd-kit/state` coalesces multiple signal writes into a single render pass. Avoid introducing new reactive primitives.
7. **`collisionDetector`** is *required* on `abstract.Droppable.Input` but *optional* on `dom.Droppable.Input`, which provides `defaultCollisionDetection` as a fallback. Any new `Input` interface at the DOM layer should follow the same optional-with-default pattern.

---

## The Problem with the Current Architecture

1. **Sorting behaviour is item-owned**: each item that wants to be sortable must be declared as a `Sortable`, not a plain `Draggable`. Containers (`Droppable`) are passive.
2. **`group`** is a property of each item. Containers do not know whether their children are sorted or which children belong to them.
3. **`instanceof` coupling**: both `OptimisticSortingPlugin` and `SortableKeyboardPlugin` use `instanceof SortableDroppable` / `instanceof SortableDraggable` to gate their logic. This means non-`Sortable` entities can never participate in sorting, and the check cannot be duck-typed away without replacing the concrete classes.
4. **No nested-container support**: the current design gives no first-class way for a `Sortable` item to *also* be a droppable container for other items (e.g. a folder that is both draggable and can receive files dropped into it). `Sortable` already creates a `SortableDroppable` to receive *sibling* drops for reordering, but that droppable represents the item's own slot — not a child container.
5. **No layout modes**: there is no concept of *how* items are placed inside a droppable once dropped. "Sorted list" is the only model; free-floating canvas placement or snapping to a grid requires custom code with no framework support.

---

## The Goal: Droppable-Owned Sorting

Redesign the system so that **the `Droppable` (container) is the authority** over whether its contents are sorted, what types it accepts, and how items are ordered. A draggable item dropped into a sortable droppable automatically participates in sorting — no `Sortable` composite entity is needed at the call site.

**Core principle:** `sortable` is a *behavioural mode* of a `Droppable`, not a separate entity type.

---

## New Concepts to Introduce

### 1. `sortable` flag on `Droppable`

A boolean property on `Droppable` that opts the container into managed sorting. When `true`, the `DroppableSortingPlugin` (see Step 3) will manage the order of accepted `Draggable` items inside it.

### 2. `SortingStrategy`

A pure function that receives the **current state** of a sortable container and the active drag event and returns a new **index order** for the container's children. It replaces the hard-coded `move()` call in `OptimisticSortingPlugin`.

```typescript
/**
 * A function that computes the new sort order for items within a container.
 *
 * @param args.draggables     - Ordered list of Draggable children registered with this container
 * @param args.droppable      - The sortable Droppable container
 * @param args.dragOperation  - The current drag operation (source, target, position)
 * @returns                   - The reordered array of Draggables, or the original array if no change
 */
export type SortingStrategy = (args: {
  draggables: Draggable[];
  droppable: Droppable;
  dragOperation: DragOperation;
}) => Draggable[];
```

Built-in strategies to provide (each as a named export from `packages/dom/src/sortable/strategies/`):

- **`verticalListStrategy`** — reorder by vertical position (default for flow-based lists)
- **`horizontalListStrategy`** — reorder by horizontal position
- **`gridStrategy`** — reorder by row-then-column, useful for equal-size grid cells
- **`rectSortingStrategy`** — general 2-D reorder based on closest-center distances (the current behaviour of `move()` adapted as a strategy)

The default strategy when `sortingStrategy` is not provided should be `rectSortingStrategy` for maximum backward compatibility.

### 3. `PlacementMode`

An enum-like string union that describes how dropped items are laid out within a `Droppable`:

```typescript
export type PlacementMode =
  | 'flow'      // Items participate in normal document flow; position is determined by sort order (default)
  | 'freeform'  // Items are positioned absolutely at the drop coordinates within the container
  | 'grid';     // Items snap to a discrete grid defined by `gridSize` on the droppable
```

- When `placementMode === 'flow'` (the default), the `DroppableSortingPlugin` manages order as before.
- When `placementMode === 'freeform'`, the plugin records the **drop coordinates** relative to the container bounds and stores them on the draggable (e.g. as `draggable.data.position`). It does **not** reorder DOM siblings.
- When `placementMode === 'grid'`, drop coordinates are snapped to the nearest grid cell defined by `gridSize: { columns: number; rows: number }` (or pixel dimensions) on the `Droppable`.
- `sortable: true` implies `placementMode: 'flow'` if `placementMode` is not set explicitly. Setting `placementMode: 'freeform'` while `sortable: true` is a no-op for sorting (freeform placement overrides).

### 4. Explicit `parent` tracking on `Draggable`

To know which container owns a given `Draggable` child, the `Draggable` must carry an explicit **`parent`** identifier rather than relying on spatial containment. Spatial containment is unreliable during a drag because the dragged element has been lifted out of flow.

Add `parent?: UniqueIdentifier` to `Draggable.Input` and as a `@reactive accessor` on `Draggable`. This is set:
- Explicitly by the caller (recommended when the parent is known statically, e.g. in framework bindings like `useDraggable({ id, parent: listId })`).
- Automatically by the `DroppableSortingPlugin` when an item is dropped into a sortable container (the plugin sets `draggable.parent = droppable.id`).

The `children` list of a sortable `Droppable` is derived by filtering `manager.registry.draggables` where `draggable.parent === this.id`. This is a `@derived` getter on the DOM-layer `Droppable`.

### 5. Nested Containers (Draggable + Droppable)

A common pattern is an item that is **both draggable** (it can be moved) **and a droppable container** (it can receive children). Examples: a folder card in a Kanban board, a nested list, a tree node.

In the new design this is achieved by creating both a `Draggable` and a `Droppable` with **the same `id`** and, optionally, the same DOM `element`:

```typescript
// An item that is both draggable and a sortable container for nested items
const folder = new Draggable({ id: 'folder-1', type: 'folder', parent: 'root' });
const folderContainer = new Droppable({
  id: 'folder-1',          // same id as the draggable
  sortable: true,
  accept: ['card'],
  element: folderElement,
});
```

The framework bindings should expose a combined helper for this pattern. In React:

```tsx
function useNestedContainer(input: { id; parent; accept; type; sortable? }) {
  const draggable = useDraggable({ id: input.id, type: input.type, parent: input.parent });
  const droppable = useDroppable({ id: input.id, accept: input.accept, sortable: input.sortable ?? true });
  return { draggable, droppable, ref: draggable.ref }; // single ref attaches both
}
```

The `DroppableSortingPlugin` must handle the case where the drag source's `id` equals a registered `Droppable.id` (i.e. the dragged item is also a container). It must not treat the draggable's own container as a valid drop target for self-nesting.

---

## Desired API (Target State)

### DOM / framework-agnostic layer

```typescript
// ── Simple sorted list ─────────────────────────────────────────────────────
const list = new Droppable({
  id: 'list-1',
  sortable: true,                         // opt-in to sorting behaviour
  accept: ['card'],                       // gate: only 'card' items sort & drop here
  collisionDetector: closestCenter,       // optional; defaultCollisionDetection used if omitted
});

const item = new Draggable({
  id: 'item-a',
  type: 'card',
  parent: 'list-1',                       // explicit parent for children tracking
});

// ── Freeform canvas ────────────────────────────────────────────────────────
const canvas = new Droppable({
  id: 'canvas',
  placementMode: 'freeform',             // absolute positioning within the container
  accept: ['widget'],
});

// ── Grid canvas ────────────────────────────────────────────────────────────
const grid = new Droppable({
  id: 'grid',
  placementMode: 'grid',
  gridSize: { columnWidth: 120, rowHeight: 80 },
  accept: ['widget'],
});

// ── Nested container (folder that can be dragged and can receive cards) ────
const folder = new Draggable({ id: 'folder-1', type: 'folder', parent: 'board' });
const folderDroppable = new Droppable({
  id: 'folder-1',                        // same id ties them together
  sortable: true,
  accept: ['card'],
});
```

### React layer

```tsx
// ── Sorted list container ──────────────────────────────────────────────────
function SortableList({ id, items }) {
  const { ref } = useDroppable({ id, sortable: true, accept: ['card'] });
  return <ul ref={ref}>{items.map((item) => <Card key={item.id} {...item} parent={id} />)}</ul>;
}

// ── Plain draggable card — no useSortable needed ───────────────────────────
function Card({ id, parent }) {
  const { ref, isDragging } = useDraggable({ id, type: 'card', parent });
  return <li ref={ref} style={{ opacity: isDragging ? 0.5 : 1 }}>{id}</li>;
}

// ── Freeform canvas ────────────────────────────────────────────────────────
function Canvas({ id }) {
  const { ref } = useDroppable({ id, placementMode: 'freeform', accept: ['widget'] });
  return <div ref={ref} style={{ position: 'relative', width: 800, height: 600 }} />;
}

// ── Nested container (folder) ──────────────────────────────────────────────
function FolderCard({ id, parent }) {
  // This element can be dragged AND receives dropped cards
  const { ref: dragRef, isDragging } = useDraggable({ id, type: 'folder', parent });
  const { ref: dropRef, isDropTarget } = useDroppable({ id, sortable: true, accept: ['card'] });

  // Attach both to the same element via callback ref composition
  const ref = useComposedRef(dragRef, dropRef);
  return (
    <div ref={ref} style={{ opacity: isDragging ? 0.4 : 1, outline: isDropTarget ? '2px solid blue' : 'none' }}>
      Folder {id}
    </div>
  );
}
```

> **Backwards compatibility note:** `useSortable` / `Sortable` must remain and continue to work unchanged. They are re-implemented on top of the new primitives (see Step 4) — not removed.

---

## Implementation Plan

Work through the following steps **in order**, committing each step as an atomic commit on the `main` branch of `spawnxe/dnd-kit-unified-draggable`. Commit messages follow the pattern `feat(pkg): description`.

---

### Step 1 — Extend the abstract `Droppable` contract (`packages/abstract`)

**File:** `packages/abstract/src/core/entities/droppable/droppable.ts`

Add to `Input<T>`:

```typescript
sortable?: boolean;
sortingStrategy?: SortingStrategy;
placementMode?: PlacementMode;
gridSize?: GridSize;   // only meaningful when placementMode === 'grid'
```

Add to the `Droppable` class body:

```typescript
@reactive public accessor sortable: boolean = false;
@reactive public accessor sortingStrategy: SortingStrategy | undefined;
@reactive public accessor placementMode: PlacementMode = 'flow';
@reactive public accessor gridSize: GridSize | undefined;
```

**File:** `packages/abstract/src/core/entities/draggable/draggable.ts`

Add to `Input<T>`:

```typescript
parent?: UniqueIdentifier;
```

Add to `Draggable` class body:

```typescript
@reactive public accessor parent: UniqueIdentifier | undefined;
```

**File:** `packages/abstract/src/core/index.ts`

Export the new types:

```typescript
export type { SortingStrategy, PlacementMode, GridSize } from './entities/droppable/droppable.ts';
```

**Where to define the new types:**
Define `SortingStrategy`, `PlacementMode`, and `GridSize` in `packages/abstract/src/core/entities/droppable/droppable.ts` above the `Input` interface. Import `DragOperation` from `'../../manager/index.ts'` (already available in the file) and import `Draggable` (already imported).

```typescript
export type PlacementMode = 'flow' | 'freeform' | 'grid';

export interface GridSize {
  columnWidth: number;
  rowHeight: number;
}

export type SortingStrategy = (args: {
  draggables: Draggable[];
  droppable: Droppable;
  dragOperation: DragOperation;
}) => Draggable[];
```

---

### Step 2 — Extend the DOM `Droppable` and `Draggable` (`packages/dom`)

**File:** `packages/dom/src/core/entities/droppable/droppable.ts`

- Extend `Input<T>` to include `sortable`, `sortingStrategy`, `placementMode`, and `gridSize` (forwarded to `super`).
- Add a `@derived` getter `children` that returns the ordered list of `Draggable` children registered with this container:

```typescript
@derived
public get children(): Draggable[] {
  const {manager, id} = this;
  if (!manager) return [];

  // Reactively read the draggable registry signal (.value iterates reactively)
  const result: Draggable[] = [];
  for (const draggable of manager.registry.draggables.value) {
    if (draggable.parent === id) {
      result.push(draggable);
    }
  }
  return result;
}
```

> **Note:** Use `manager.registry.draggables.value` (not `.peek()`) so that the derived getter re-evaluates whenever the registry changes.

**File:** `packages/dom/src/core/entities/draggable/draggable.ts`

- Extend the DOM `Input<T>` interface to include `parent?: UniqueIdentifier`.
- Forward `parent` to the `super()` call so that the abstract `Draggable` stores it.

---

### Step 3 — Introduce `DroppableSortingPlugin` (`packages/dom`)

**File:** `packages/dom/src/core/plugins/DroppableSortingPlugin.ts`

This plugin is the canonical sorting engine for the new architecture. It **must not** use `instanceof SortableDroppable` or `instanceof SortableDraggable` anywhere — it gate-checks on `droppable.sortable === true` only.

#### Algorithm — `dragover` handler

```
1. Read dragOperation.source and dragOperation.target from the manager.
2. If source is null, return early.
3. Find the target droppable: manager.registry.droppables.get(target?.id).
4. If the target droppable does not have sortable === true, return early.
5. If the target droppable does not accept(source), return early.
6. Determine the effective sortingStrategy:
     strategy = targetDroppable.sortingStrategy ?? rectSortingStrategy
7. Compute newOrder = strategy({ draggables: targetDroppable.children, droppable: targetDroppable, dragOperation })
8. If newOrder === targetDroppable.children (reference equality, meaning no change), return early.
9. Queue a microtask (to align with OptimisticSortingPlugin's timing):
   a. If event.defaultPrevented, abort.
   b. Await manager.renderer.rendering (to read post-render positions).
   c. Re-compute newOrder again with the updated shapes (in case the renderer moved things).
   d. Reorder the DOM: for each item in newOrder, call targetDroppable.element.append(item.element)
      in the computed order (or use insertAdjacentElement for minimal DOM moves).
   e. batch(() => {
        for the source draggable: update draggable.parent = targetDroppable.id (cross-list transfer)
        update an internal Map<draggableId, number> within the plugin recording the current index
      })
   f. Disable collision observer → setDropTarget(source.id) → re-enable collision observer
      (same pattern as OptimisticSortingPlugin).
```

#### Algorithm — `dragend` (cancelled) handler

```
1. If !event.canceled, return early.
2. Restore each draggable's parent and DOM position to the snapshot taken at drag start.
3. batch(() => { restore parent and index snapshot for each affected draggable })
```

#### Snapshot at drag start

Subscribe to the `'dragstart'` event (or use a reactive effect on `dragOperation.status.dragging`) to record:

```typescript
interface DragSnapshot {
  parent: UniqueIdentifier | undefined;
  index: number;
}
// Map<draggableId, DragSnapshot>
```

This snapshot is cleared when a new drag starts (same pattern as `store.clear()` in `Sortable`).

#### Index accessor

The plugin exposes a helper method for reading the current computed index of a draggable within its parent container:

```typescript
public getIndex(draggableId: UniqueIdentifier): number {
  return this.#indexMap.get(draggableId) ?? -1;
}
```

This replaces `Sortable.index` as the source of truth for item order.

---

### Step 4 — Deprecate `Sortable` as a required primitive (`packages/dom`)

**File:** `packages/dom/src/sortable/sortable.ts`

Mark the class and its sub-classes with a JSDoc `@deprecated` tag. Re-implement them as thin wrappers so that all **existing** code continues to work without modification:

- `SortableDroppable` becomes an alias for `Droppable` with `sortable: true` in its constructor (it can simply call `super({ ...input, sortable: true }, manager)`).
- `SortableDraggable` becomes an alias for plain `Draggable` — no behaviour change needed since `parent` is now the mechanism for container membership, and `SortableDraggable` already delegates `index`/`group` to its `Sortable` parent.
- `Sortable` itself can remain functionally identical *as long as* the `DroppableSortingPlugin` is present. Consider making `Sortable` register `DroppableSortingPlugin` in its default `plugins` list (alongside or instead of `OptimisticSortingPlugin`).

**File:** `packages/dom/src/sortable/plugins/OptimisticSortingPlugin.ts`

- Add a JSDoc `@deprecated` comment.
- Internally, when the plugin detects a `SortableDroppable` target, it may delegate to `DroppableSortingPlugin` rather than replicating the logic:

```typescript
// Inside OptimisticSortingPlugin dragover handler:
const droppableSortingPlugin = manager.registry.plugins.get(DroppableSortingPlugin);
if (droppableSortingPlugin && target?.sortable) {
  // Let DroppableSortingPlugin handle it
  return;
}
// ...original instanceof-based path for pure backwards compat
```

**File:** `packages/dom/src/sortable/utilities.ts`

Add overloads that duck-type on the new `sortable` property in addition to the existing `instanceof` checks, so that new-style droppables are also recognised by legacy code that calls `isSortable()`:

```typescript
export function isSortable(element: Draggable | Droppable | null): boolean {
  if (!element) return false;
  // New duck-type path
  if ('sortable' in element && (element as Droppable).sortable === true) return true;
  // Legacy instanceof path
  return element instanceof SortableDroppable || element instanceof SortableDraggable;
}
```

---

### Step 5 — Update `SortableKeyboardPlugin` (`packages/dom`)

**File:** `packages/dom/src/sortable/plugins/SortableKeyboardPlugin.ts`

The plugin currently gates all keyboard sorting on `isSortable(dragOperation.source)` (an `instanceof` check). After updating `isSortable()` in Step 4 to duck-type on `droppable.sortable`, this plugin will automatically work for new-style containers **when the source is a `SortableDraggable`** (legacy path).

However, for plain `Draggable` sources dragged within a `sortable: true` container, the plugin must also activate. Update the guard:

```typescript
// Before:
if (!isSortable(dragOperation.source)) return;

// After:
const source = dragOperation.source;
const inSortableContainer =
  source?.parent != null &&
  (manager.registry.droppables.get(source.parent) as Droppable | undefined)?.sortable === true;

if (!isSortable(source) && !inSortableContainer) return;
```

Where the plugin previously read `source.sortable.index` / `source.sortable.group`, replace with:

```typescript
const plugin = manager.registry.plugins.get(DroppableSortingPlugin);
const index = plugin?.getIndex(source.id) ?? -1;
const containerDroppable = manager.registry.droppables.get(source.parent);
```

---

### Step 6 — Update React bindings (`packages/react`)

**File:** `packages/react/src/core/droppable/useDroppable.ts`

- Add `sortable`, `sortingStrategy`, `placementMode`, and `gridSize` to `UseDroppableInput<T>`.
- Forward them to the `Droppable` constructor and wire reactive updates with `useOnValueChange`.

**File:** `packages/react/src/core/draggable/useDraggable.ts`

- Add `parent?: UniqueIdentifier` to `UseDraggableInput<T>`.
- Forward it to the `Draggable` constructor and wire `useOnValueChange(parent, () => (draggable.parent = parent))`.

**File:** `packages/react/src/sortable/useSortable.ts`

`useSortable` must remain **API-identical** to the current version. Internally it can continue to use the `Sortable` composite (which has been re-implemented as a thin wrapper in Step 4). No consumer-visible changes. Add a JSDoc note pointing to the new `useDraggable` + `useDroppable` pattern as the preferred alternative.

---

### Step 7 — Update Solid, Svelte, Vue bindings

Mirror the React changes in each framework package:

- `packages/solid/src/core/droppable/useDroppable.ts` — add `sortable`, `sortingStrategy`, `placementMode`, `gridSize` to the input interface and the `createEffect` sync block.
- `packages/solid/src/core/draggable/useDraggable.ts` — add `parent` to the input interface and sync block.
- Apply the same pattern for `packages/svelte/` and `packages/vue/`.

Do **not** change the signatures of `useSortable` in any framework — it remains as-is.

---

### Step 8 — Introduce built-in sorting strategies (`packages/dom`)

**Directory:** `packages/dom/src/sortable/strategies/`

Create the following strategy files, each exporting a `SortingStrategy` function:

- `verticalListStrategy.ts` — sort by the vertical midpoint (`shape.center.y`) of each child
- `horizontalListStrategy.ts` — sort by the horizontal midpoint (`shape.center.x`)
- `gridStrategy.ts` — sort by row (floor(y / rowHeight)) then column (floor(x / columnWidth))
- `rectSortingStrategy.ts` — the existing behaviour from `move()` in `@dnd-kit/helpers`, adapted to the `SortingStrategy` signature

Export all four from `packages/dom/src/sortable/index.ts`.

---

### Step 9 — Tests & Stories

**Unit tests** for `DroppableSortingPlugin`:
- Single sortable list reorder (move item from index 2 to index 0)
- Cross-list transfer between two sortable droppables with different `accept` types
- `accept` filtering: items of non-accepted type must not trigger sorting
- Cancelled drag (`dragend` with `canceled: true`) restores original `parent` and DOM order
- Freeform placement: verify item position is stored on `draggable.data.position` rather than DOM being reordered
- Grid placement: verify drop coordinates are snapped to the nearest cell

**Storybook stories:**
- `SortableListNoPrimitive` — demonstrates `useDroppable({ sortable: true })` + `useDraggable({ parent })`, no `useSortable`
- `NestedContainers` — a folder-in-board pattern using `useDraggable` + `useDroppable` on the same element
- `FreeformCanvas` — `useDroppable({ placementMode: 'freeform' })` with absolutely-positioned widgets
- `GridCanvas` — `useDroppable({ placementMode: 'grid', gridSize: {...} })` with snap-to-grid

---

## Migration Guide (for existing consumers)

### Before (current `useSortable` pattern)

```tsx
function SortableItem({ id, index, group }) {
  const { ref, isDragging } = useSortable({ id, index, group });
  return <li ref={ref}>{id}</li>;
}
```

### After (new `useDraggable` + `useDroppable` pattern)

```tsx
function SortableItem({ id, parent }) {
  // No index or group prop required — the DroppableSortingPlugin manages order
  const { ref, isDragging } = useDraggable({ id, type: 'item', parent });
  return <li ref={ref}>{id}</li>;
}

function SortableList({ id }) {
  const { ref } = useDroppable({ id, sortable: true, accept: ['item'] });
  return <ul ref={ref}>{/* items */}</ul>;
}
```

**The `useSortable` / `Sortable` API is not removed.** Existing code continues to work without any changes. The new pattern is the preferred approach for new code only.

---

## Key Constraints

- **Do not break the existing public API.** All existing `useSortable` / `Sortable` usage must continue to work without modification.
- **Type safety must be maintained throughout.** No `any` escapes in new code paths.
- **Reactive state** must use the existing `@dnd-kit/state` primitives (`@reactive`, `@derived`, `signal`, `batch`, `effect`). Do not introduce new state management.
- **No new dependencies.** Work only within the existing monorepo package graph.
- **The `accept` filter on the droppable is the single gate** for both drop acceptance and sort participation — if a type is not accepted, it is neither dropped into nor sorted within the container.
- **`DroppableSortingPlugin` must not use `instanceof`** against `SortableDroppable` or `SortableDraggable`. It uses duck-typed `droppable.sortable === true` checks only.
- **Self-nesting prevention**: the `DroppableSortingPlugin` must never allow a draggable to be dropped into a droppable with the same `id` (i.e. a container cannot become a child of itself).
- All new types and interfaces must be exported from the appropriate `index.ts` barrel files.
- Commit messages should follow the pattern: `feat(dom): ...`, `feat(abstract): ...`, `refactor(react): ...`

---

## Files to Focus On First

| File | Reason |
|---|---|
| `packages/abstract/src/core/entities/droppable/droppable.ts` | Add `sortable`, `sortingStrategy`, `placementMode`, `gridSize` to the base contract |
| `packages/abstract/src/core/entities/draggable/draggable.ts` | Add `parent` to the base contract |
| `packages/dom/src/core/entities/droppable/droppable.ts` | Forward new inputs; add `children` derived getter |
| `packages/dom/src/core/entities/draggable/draggable.ts` | Forward `parent` input |
| `packages/dom/src/sortable/plugins/OptimisticSortingPlugin.ts` | Reference implementation for sort logic; deprecate and delegate |
| `packages/dom/src/sortable/sortable.ts` | Understand composite composition; add `@deprecated` JSDoc |
| `packages/dom/src/sortable/plugins/SortableKeyboardPlugin.ts` | Update `isSortable` guard; replace `source.sortable.index` reads |
| `packages/dom/src/sortable/utilities.ts` | Extend `isSortable` to duck-type on `droppable.sortable` |
| `packages/react/src/sortable/useSortable.ts` | Remain unchanged API; add deprecation note pointing to new pattern |
| `packages/react/src/core/droppable/useDroppable.ts` | Add `sortable`, `sortingStrategy`, `placementMode`, `gridSize` props |
| `packages/react/src/core/draggable/useDraggable.ts` | Add `parent` prop |
| `packages/abstract/src/core/manager/registry.ts` | Understand `EntityRegistry.value` (reactive iterator) vs `Symbol.iterator` (non-reactive) |

---

## Architecture Decision Summary

| Concern | Current Owner | New Owner |
|---|---|---|
| Is content sortable? | Each item (`Sortable`) | Container (`Droppable.sortable`) |
| What types are sorted/accepted? | Each item (`Sortable.type`) | Container (`Droppable.accept`) |
| What group does an item belong to? | Each item (`Sortable.group`) | Container identity (`Droppable.id`) |
| Sort order / index | Each item (`Sortable.index`) | Plugin (`DroppableSortingPlugin.getIndex(id)`) |
| Which container owns an item? | Implicit via `group` matching | Explicit `Draggable.parent` |
| Sorting algorithm | Hard-coded `move()` in plugin | Per-container `Droppable.sortingStrategy` |
| Item placement within container | Flow-only (sorted list) | `Droppable.placementMode`: `'flow'` \| `'freeform'` \| `'grid'` |
| Sorting detection | `instanceof SortableDroppable` | Duck-typed `droppable.sortable === true` |
| Nested containers | Not first-class; requires manual wiring | First-class: same `id` on `Draggable` + `Droppable` |
| Keyboard accessibility | Gated on `instanceof SortableDraggable` | Gated on `parent` → container `sortable === true` |
