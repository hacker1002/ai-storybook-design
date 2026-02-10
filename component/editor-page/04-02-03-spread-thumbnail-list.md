# SpreadThumbnailList: Component Design

> **Note:** Component này hỗ trợ 2 layout modes: `horizontal` (filmstrip) và `grid` (vertical scroll). Cùng actions, chỉ khác scroll direction.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
Layout: Horizontal (filmstrip with native scroll-x)
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList                                │
│  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐     │
│  │Thumbnail│  ║Thumbnail║  │Thumbnail│  │Thumbnail│  │Thumbnail│  │ + NEW │     │
│  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  │       │     │
│  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘  └───────┘     │
│       ↑           ↑                                                  ↑          │
│  SpreadThumbnail  selected                                    NewSpreadBtn      │
│                                 ←────── scroll-x ──────→                        │
└─────────────────────────────────────────────────────────────────────────────────┘

Layout: Grid (vertical scroll)
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Thumbnail  │  │  Thumbnail  │  │  Thumbnail  │  │  Thumbnail  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘             │
│  ┌─────────────┐  ┌───────────────────┐                                         │
│  │  Thumbnail  │  │   NewSpreadBtn    │  ← vertical scroll, CSS grid            │
│  └─────────────┘  └───────────────────┘                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                    ┌─────────────────────┐
                    │    SnapshotStore    │
                    │   (Zustand global)  │
                    └──────────┬──────────┘
                               │
                     ┌─────────┼─────────┐
                     │         │         │
                     ▼         ▼         ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: mode, dummyType?, selectedId, displayField, ...                     │  │
│  │  Local State: draggedId, dropTargetId                                       │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                        │                │
│         ▼                                                        ▼                │
│  ┌─────────────────────┐                                 ┌─────────────────┐      │
│  │   SpreadThumbnail   │                                 │ NewSpreadButton │      │
│  │   (for each id)     │                                 │                 │      │
│  │                     │                                 │ Props:          │      │
│  │ Props:              │                                 │ • size          │      │
│  │ • spreadId          │                                 │                 │      │
│  │ • mode, dummyType   │                                 │ Callbacks:      │      │
│  │ • isSelected        │                                 │ • onClick       │      │
│  │ • displayField      │                                 └─────────────────┘      │
│  │ • isDragging, ...   │                                                          │
│  │                     │                                                          │
│  │ (uses stores)       │                                                          │
│  │                     │                                                          │
│  │ Callbacks:          │                                                          │
│  │ • onClick           │                                                          │
│  │ • onDragStart/Over  │                                                          │
│  └─────────────────────┘                                                          │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Thumbnails container (horizontal filmstrip hoặc vertical grid) cho spread navigation và reorder bằng drag-drop.

**Shared Types:**

```typescript
type ThumbnailListLayout = 'horizontal' | 'grid';
type SpreadViewMode = 'dummy' | 'finalize';
type DummyType = 'prose' | 'poetry';
type DisplayField = 'art_note' | 'visual_description';
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SpreadThumbnailListProps {
  mode: SpreadViewMode;
  dummyId?: string;                      // Required for dummy mode (UUID)
  selectedId: string | null;             // ID-based selection
  displayField: DisplayField;
  isDragEnabled: boolean;
  canAdd: boolean;

  // Layout configuration
  layout: ThumbnailListLayout;           // 'horizontal' = filmstrip, 'grid' = vertical
  columnsPerRow?: number;                // Only for grid layout, default 4, range 2-6

  // Selection callback (parent manages selection state)
  onSpreadClick: (id: string) => void;
  onColumnsChange?: (columns: number) => void;  // Only for grid layout
}

interface SpreadThumbnailListState {
  // Drag-drop only (scroll handled by CSS)
  draggedId: string | null;
  dropTargetId: string | null;
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors (mode-conditional, shallow compare for minimal re-renders)
spreadIds = mode === 'dummy'
  ? useDummySpreadIds(dummyId!)      // string[]
  : useSpreadIds();                   // string[]

// SnapshotStore Actions (no re-render)
const { addDummySpread, reorderDummySpreads, addSpread, reorderSpreads } = useSnapshotActions();

// Handlers
handleAddSpread(): void {
  const newSpread = createEmptySpread();
  mode === 'dummy'
    ? addDummySpread(dummyId!, newSpread)
    : addSpread(newSpread);
}

handleDragEnd(activeId: string, overId: string): void {
  const oldIndex = spreadIds.indexOf(activeId);
  const newIndex = spreadIds.indexOf(overId);
  if (oldIndex !== newIndex) {
    mode === 'dummy'
      ? reorderDummySpreads(dummyId!, oldIndex, newIndex)
      : reorderSpreads(oldIndex, newIndex);
  }
}
```

### 2.3 Render Logic (pseudo)

```
SpreadThumbnailList:
  spreadIds = mode === 'dummy' ? useDummySpreadIds(dummyId) : useSpreadIds()
  actions = useSnapshotActions()

  thumbnailSize = layout === 'horizontal' ? 'small' : 'medium'
  gridColumns = columnsPerRow ?? 4

  RENDER ScrollContainer (native scroll, scroll-snap):
    FOR EACH id IN spreadIds:
      RENDER SpreadThumbnail với:
        - spreadId, mode, dummyId, isSelected: id === selectedId
        - displayField, size: thumbnailSize
        - isDragging: id === draggedId, isDropTarget: id === dropTargetId
        - isDragEnabled, onClick, onDragStart, onDragOver, onDragEnd

    IF canAdd:
      RENDER NewSpreadButton onClick → addSpread/addDummySpread

  ON selectedId change → scrollThumbnailIntoView(selectedId)

  ON dragEnd(activeId, overId) → reorderSpreads/reorderDummySpreads
```

### 2.4 Visual

**Normal State (horizontal):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐     │
│  │  ┌───┐  │  ║  ┌───┐  ║  │  ┌───┐  │  │  ┌───┐  │  │  ┌───┐  │  │   +   │     │
│  │  │   │  │  ║  │   │  ║  │  │   │  │  │  │   │  │  │  │   │  │  │  NEW  │     │
│  │  └───┘  │  ║  └───┘  ║  │  └───┘  │  │  └───┘  │  │  └───┘  │  │       │     │
│  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  └───────┘     │
│  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘                │
│                    ↑                                                            │
│               selected                             ←── native scroll-x ──→      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Dragging State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────┐  ┌·········┐  ┌─────────┐  ┌ · · · · ┐  ┌─────────┐  ┌───────┐     │
│  │   0-1   │  · dragging·  │   4-5   │  · drop    ·  │   8-9   │  │  NEW  │     │
│  └─────────┘  └·········┘  └─────────┘  · target  ·  └─────────┘  └───────┘     │
│                                         └ · · · · ┘                             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              ┌─────────────────┐                                │
│                              │        +        │                                │
│                              │    Add First    │                                │
│                              │     Spread      │                                │
│                              └─────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý quan trọng:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - **KHÔNG** thiết kế store integration tại đây — child component tự thiết kế trong file riêng

### 3.1 SpreadThumbnail

📄 **Doc:** [`component/editor-page/04-02-03-01-spread-thumbnail.md`](component/editor-page/04-02-03-01-spread-thumbnail.md)

**Mục đích:** Thumbnail preview của một spread, dùng chung cho cả horizontal và grid layouts.

```typescript
interface SpreadThumbnailProps {
  spreadId: string;                      // ID-based, uses store internally
  mode: SpreadViewMode;
  dummyId?: string;                      // Required for dummy mode
  isSelected: boolean;
  displayField: DisplayField;
  isDragging?: boolean;
  isDropTarget?: boolean;
  isDragEnabled?: boolean;
  size: 'small' | 'medium';              // small=100×80, medium=dynamic

  onClick: () => void;
  onDragStart?: () => void;
  onDragOver?: () => void;
  onDragEnd?: () => void;
}
```

**Visual States:**

```
Normal:              Selected:            Dragging:           DropTarget:
┌─────────┐          ╔═════════╗          ┌·········┐         ┌ · · · · ┐
│  ┌───┐  │          ║  ┌───┐  ║          ·  ┌───┐  ·         ·  ┌───┐  ·
│  │   │  │          ║  │   │  ║          ·  │   │  ·         ·  │   │  ·
│  └───┘  │          ║  └───┘  ║          ·  └───┘  ·         ·  └───┘  ·
│   0-1   │          ║   2-3   ║          ·   4-5   ·         ·   6-7   ·
└─────────┘          ╚═════════╝          └·········┘         └ · · · · ┘
```

---

### 3.2 NewSpreadButton

**Mục đích:** Button để add new spread, hiển thị cuối danh sách thumbnails.

```typescript
interface NewSpreadButtonProps {
  size: 'small' | 'medium';
  onClick: () => void;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| ID-based subscription | `useDummySpreadIds` / `useSpreadIds` | Shallow compare → only re-render when IDs change |
| ID-based selection | `selectedId: string` instead of index | Stable across reorder |
| Native scroll | `overflow-x: auto` + `scroll-snap` | Simpler than arrow buttons, native touch support |
| Drag-drop library | `@dnd-kit/sortable` | Accessible, touch-friendly |
| Unified layout prop | `layout: 'horizontal' \| 'grid'` | Same logic, different CSS |

### 4.2 Scroll Behavior

- **Horizontal:** native `overflow-x: auto`, `scroll-snap-type: x mandatory`, thin scrollbar
- **Grid:** CSS grid `repeat(columnsPerRow, 1fr)`, `overflow-y: auto`
- **Auto-scroll:** On `selectedId` change, scroll thumbnail into view using `scrollIntoView()`

### 4.3 Drag-Drop Behavior

- Library: `@dnd-kit/sortable`
- On drag start → set `draggedId`
- On drag over → set `dropTargetId`
- On drag end → call `reorderSpreads` / `reorderDummySpreads`, clear state

### 4.4 Accessibility

| Element | Role/ARIA |
|---------|-----------|
| Container | `role="listbox"`, `aria-orientation` based on layout |
| Thumbnail | `role="option"`, `aria-selected`, `aria-label="Spread N, pages X-Y"` |

**Keyboard:** `ArrowLeft/Right` navigate, `Home/End` first/last, `Space/Enter` confirm

---
