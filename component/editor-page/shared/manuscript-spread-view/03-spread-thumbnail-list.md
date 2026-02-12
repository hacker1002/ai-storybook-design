# SpreadThumbnailList: Component Design

> **Note:** Props-driven component. Receives spreads array và render props from parent. Supports horizontal (filmstrip) và grid layouts.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
Layout: Horizontal (filmstrip with native scroll-x)
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         SpreadThumbnailList<TSpread>                            │
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
ManuscriptSpreadView<TSpread>
       │
       │ passes spreads[], canAdd, canReorder, renderItems, render*Item, callbacks
       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                         SpreadThumbnailList<TSpread>                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: spreads[], selectedId, layout, renderItems, render*Item             │  │
│  │  Local State: draggedId, dropTargetId                                       │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                        │                │
│         ▼                                                        ▼                │
│  ┌─────────────────────┐                                 ┌─────────────────┐      │
│  │  SpreadThumbnail    │                                 │ NewSpreadButton │      │
│  │  (for each spread)  │                                 │                 │      │
│  │                     │                                 │ Props:          │      │
│  │ Props:              │                                 │ • size          │      │
│  │ • spread            │                                 │                 │      │
│  │ • spreadIndex       │                                 │ Callbacks:      │      │
│  │ • isSelected        │                                 │ • onClick       │      │
│  │ • renderItems       │                                 └─────────────────┘      │
│  │ • render*Item       │                                                          │
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

**Mục đích:** Thumbnails container (horizontal filmstrip hoặc vertical grid) cho spread navigation và reorder. Props-driven - receives spreads from parent.

**Shared Types:**

```typescript
type ThumbnailListLayout = 'horizontal' | 'grid';
type ItemType = 'image' | 'text' | 'object' | 'animation';
```

### 2.2 Interface

**Props:**

```typescript
interface SpreadThumbnailListProps<TSpread extends BaseSpread> {
  // === Data (from parent) ===
  spreads: TSpread[];
  selectedId: string | null;

  // === Layout configuration ===
  layout: ThumbnailListLayout;           // 'horizontal' | 'grid'
  columnsPerRow?: number;                // Grid only, default 4, range 2-6

  // === Render configuration (same as parent) ===
  renderItems: ItemType[];

  // === Item render functions (same as parent, used for thumbnail display) ===
  // Thumbnails render items with disabled interactions (view-only)
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // === Feature flags ===
  canAdd: boolean;
  canReorder: boolean;
  canDelete: boolean;

  // === Callbacks ===
  onSpreadClick: (spreadId: string) => void;
  onReorderSpread?: (fromIndex: number, toIndex: number) => void;
  onAddSpread?: () => void;
  onDeleteSpread?: (spreadId: string) => void;
}
```

**Local State:**

```typescript
interface SpreadThumbnailListState {
  // Drag-drop only (scroll handled by CSS)
  draggedId: string | null;
  dropTargetId: string | null;
}
```

### 2.3 Context Building for Thumbnails

Thumbnails use same render props as main panel but with disabled interactions:

```typescript
buildThumbnailImageContext(image: SpreadImage, index: number, spread: TSpread): ImageItemContext<TSpread>
  return {
    item: image,
    itemIndex: index,
    spreadId: spread.id,
    spread: spread,
    isSelected: false,          // Items inside thumbnail not selectable
    isSpreadSelected: false,    // Thumbnail is view-only
    onSelect: () => {},         // No-op
    onUpdate: () => {},         // No-op
    onDelete: () => {},         // No-op
  }

buildThumbnailTextContext(textbox: SpreadTextbox, index: number, spread: TSpread): TextItemContext<TSpread>
  return {
    item: textbox,
    itemIndex: index,
    spreadId: spread.id,
    spread: spread,
    isSelected: false,
    isSpreadSelected: false,
    onSelect: () => {},
    onTextChange: () => {},
    onUpdate: () => {},
    onDelete: () => {},
  }
```

### 2.4 Render Logic (pseudo)

```
SpreadThumbnailList<TSpread>:
  spreads = props.spreads
  thumbnailSize = layout === 'horizontal' ? 'small' : 'medium'
  gridColumns = columnsPerRow ?? 4

  RENDER ScrollContainer (native scroll, scroll-snap):
    style:
      IF layout === 'horizontal':
        display: flex, overflowX: auto, scrollSnapType: x mandatory
      ELSE:
        display: grid, gridTemplateColumns: repeat(gridColumns, 1fr)

    FOR EACH (spread, index) IN spreads:
      RENDER SpreadThumbnail với:
        spread: spread
        spreadIndex: index
        isSelected: spread.id === selectedId
        size: thumbnailSize
        isDragEnabled: canReorder
        isDragging: spread.id === draggedId
        isDropTarget: spread.id === dropTargetId
        renderItems, renderImageItem, renderTextItem
        onClick: () => onSpreadClick(spread.id)
        onDragStart: () => setDraggedId(spread.id)
        onDragOver: () => setDropTargetId(spread.id)
        onDragEnd: handleDragEnd

    IF canAdd:
      RENDER NewSpreadButton onClick={onAddSpread}

  ON selectedId change → scrollThumbnailIntoView(selectedId)

  handleDragEnd(activeId, overId):
    IF activeId !== overId:
      fromIndex = spreads.findIndex(s => s.id === activeId)
      toIndex = spreads.findIndex(s => s.id === overId)
      onReorderSpread?.(fromIndex, toIndex)
    setDraggedId(null)
    setDropTargetId(null)
```

### 2.5 Visual

**Normal State (horizontal):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐     │
│  │  ┌───┐  │  ║  ┌───┐  ║  │  ┌───┐  │  │  ┌───┐  │  │  ┌───┐  │  │   +   │     │
│  │  │   │  │  ║  │   │  ║  │  │   │  │  │  │   │  │  │  │   │  │  │  NEW  │     │
│  │  └───┘  │  ║  └───┘  ║  │  └───┘  │  │  └───┘  │  │  └───┘  │  │       │     │
│  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  └───────┘     │
│  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘   (if canAdd)  │
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
│                              │    Add First    │  (if canAdd)                   │
│                              │     Spread      │                                │
│                              └─────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

### 3.1 SpreadThumbnail

📄 **Doc:** [03-01-spread-thumbnail.md](./03-01-spread-thumbnail.md)

**Mục đích:** Thumbnail preview của một spread với render props.

```typescript
interface SpreadThumbnailProps<TSpread extends BaseSpread> {
  spread: TSpread;
  spreadIndex: number;
  isSelected: boolean;
  size: 'small' | 'medium';

  // Render configuration (same as parent, view-only)
  renderItems: ItemType[];
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;

  // Drag state
  isDragEnabled?: boolean;
  isDragging?: boolean;
  isDropTarget?: boolean;

  // Callbacks
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

**Mục đích:** Button để add new spread.

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
| Props-driven | spreads[] via props | No store coupling |
| Same render props | Reuse from parent | Consistent rendering |
| View-only contexts | Disabled handlers | Thumbnails not editable |
| Native scroll | `overflow-x: auto` + `scroll-snap` | Simple, native touch |
| Unified layout prop | `layout: 'horizontal' \| 'grid'` | Same logic, different CSS |

### 4.2 Scroll Behavior

- **Horizontal:** native `overflow-x: auto`, `scroll-snap-type: x mandatory`
- **Grid:** CSS grid `repeat(columnsPerRow, 1fr)`, `overflow-y: auto`
- **Auto-scroll:** On `selectedId` change, scroll thumbnail into view

### 4.3 Drag-Drop Behavior

- Library: `@dnd-kit/sortable`
- On drag start → set `draggedId`
- On drag over → set `dropTargetId`
- On drag end → call `onReorderSpread`, clear state

### 4.4 Accessibility

| Element | Role/ARIA |
|---------|-----------|
| Container | `role="listbox"`, `aria-orientation` based on layout |
| Thumbnail | `role="option"`, `aria-selected`, `aria-label="Spread N, pages X-Y"` |

**Keyboard:** `ArrowLeft/Right` navigate, `Home/End` first/last, `Space/Enter` select

---
