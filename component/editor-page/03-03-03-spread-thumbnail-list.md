# SpreadThumbnailList: Component Design

> **Note:** Component này hỗ trợ 2 layout modes: `horizontal` (filmstrip) và `grid` (vertical scroll). Cùng actions, chỉ khác scroll direction.

---

## 1. Diagrams

### 1.1 Component Hierarchy

**Layout: Horizontal (Filmstrip)**
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList                                     │
│  ┌───┐                                                                    ┌───┐ │
│  │ ◀ │  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌───────┐   │ ▶ │ │
│  │   │  │Thumbnail│  ║Thumbnail║  │Thumbnail│  │Thumbnail│  │ + NEW │   │   │ │
│  │   │  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │       │   │   │ │
│  │   │  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └───────┘   │   │ │
│  └───┘                   ↑                                               └───┘ │
│         ↑ scroll left   selected                              ↑ scroll right   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Layout: Grid (Vertical Scroll)**
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList (layout=grid)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  0  │  1    │  │  2  │  3    │  │  4  │  5    │  │  6  │  7    │            │
│  │ ┌────────┐  │  │ ┌────────┐  │  │ ┌────────┐  │  │ ┌────────┐  │            │
│  │ │"art    │  │  │ │"art    │  │  │ │"art    │  │  │ │"art    │  │            │
│  │ │note"   │  │  │ │note"   │  │  │ │note"   │  │  │ │note"   │  │            │
│  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │            │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘            │
│       ↕ drag          ↕ drag          ↕ drag          ↕ drag                   │
│  ┌─────────────┐  ┌───────────────────┐                                        │
│  │  8  │  9    │  │        +          │                                        │
│  │ ┌────────┐  │  │   Add New Spread  │                                        │
│  │ └────────┘  │  └───────────────────┘                                        │
│  └─────────────┘                                                               │
│                                    ↓ vertical scroll                            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: spreads[], selectedIndex, currentLanguage, mode, displayField,     │  │
│  │         isDragEnabled, canAdd                                               │  │
│  │  State: scrollPosition, draggedIndex, dropTargetIndex                       │  │
│  │  Callbacks: onSpreadClick, onAddSpread, onDragEnd                           │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │                      Horizontal Scroll Container                            │  │
│  │                                                                             │  │
│  │  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐            │  │
│  │  │SpreadThumb│   │SpreadThumb│   │SpreadThumb│   │NewSpread  │            │  │
│  │  │ nail[0]   │   │ nail[1]   │   │ nail[2]   │   │ Button    │            │  │
│  │  └───────────┘   └───────────┘   └───────────┘   └───────────┘            │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Horizontal scrollable thumbnails strip ở bottom, cho phép navigate spreads và reorder bằng drag-drop.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox preview theo `currentLanguage.code`

### 2.2 Interface

```typescript
type ThumbnailListLayout = 'horizontal' | 'grid';

interface SpreadThumbnailListProps {
  spreads: SpreadViewSpread[];
  selectedIndex: number | null;
  currentLanguage: Language;
  mode: SpreadViewMode;
  displayField: 'art_note' | 'visual_description';
  isDragEnabled: boolean;
  canAdd: boolean;

  // Layout configuration
  layout: ThumbnailListLayout;            // 'horizontal' = filmstrip, 'grid' = vertical scroll
  columnsPerRow?: number;             // Only for grid layout, default 4, range 2-6

  onSpreadClick: (index: number) => void;
  onAddSpread?: () => void;
  onDragEnd: (event: { oldIndex: number; newIndex: number }) => void;
  onColumnsChange?: (columns: number) => void;  // Only for grid layout
}

interface SpreadThumbnailListState {
  scrollPosition: number;
  draggedIndex: number | null;
  dropTargetIndex: number | null;

  // For horizontal layout
  canScrollLeft: boolean;
  canScrollRight: boolean;
}
```

### 2.3 Render Logic (pseudo)

```
SpreadThumbnailList:
  scrollRef = useRef()

  // Layout-specific calculations
  IF layout === 'horizontal':
    canScrollLeft = scrollPosition > 0
    canScrollRight = scrollPosition < maxScroll
    thumbnailSize = 'small'
  ELSE: // grid
    thumbnailSize = 'medium'
    gridColumns = columnsPerRow ?? 4

  RENDER Container (flex, relative):

    // Horizontal layout: scroll buttons
    IF layout === 'horizontal' AND canScrollLeft:
      RENDER ScrollButton left:
        - onClick: scrollLeft
        - icon: ◀

    // Thumbnails container
    RENDER ScrollContainer với:
      - IF layout === 'horizontal': horizontal scroll, flex row, fixed height
      - IF layout === 'grid': vertical scroll, grid layout, columnsPerRow

      FOR EACH spread, index IN spreads:
        isSelected = index === selectedIndex
        isDragging = index === draggedIndex
        isDropTarget = index === dropTargetIndex

        RENDER SpreadThumbnail với:
          - spread
          - index
          - isSelected
          - currentLanguage
          - displayField
          - isDragging
          - isDropTarget
          - isDragEnabled
          - size: thumbnailSize
          - onClick: () => onSpreadClick(index)
          - onDragStart: () => setDraggedIndex(index)
          - onDragOver: () => setDropTargetIndex(index)
          - onDragEnd: handleDragEnd

      // Add button (conditional)
      IF canAdd:
        RENDER NewSpreadButton với:
          - onClick: onAddSpread
          - size: thumbnailSize

    // Horizontal layout: right scroll button
    IF layout === 'horizontal' AND canScrollRight:
      RENDER ScrollButton right:
        - onClick: scrollRight
        - icon: ▶

  handleDragEnd(dropIndex):
    IF draggedIndex !== null AND draggedIndex !== dropIndex:
      onDragEnd({ oldIndex: draggedIndex, newIndex: dropIndex })
    setDraggedIndex(null)
    setDropTargetIndex(null)

  scrollLeft():
    scrollRef.current.scrollBy({ left: -THUMBNAIL_WIDTH * 2, behavior: 'smooth' })

  scrollRight():
    scrollRef.current.scrollBy({ left: THUMBNAIL_WIDTH * 2, behavior: 'smooth' })

  // Auto-scroll selected into view
  useEffect():
    IF selectedIndex !== null:
      scrollThumbnailIntoView(selectedIndex, layout)
```

### 2.4 Visual

**Normal State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadThumbnailList                                     │
│                                                                                 │
│     ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐ │
│     │  ┌───┐  │  ║  ┌───┐  ║  │  ┌───┐  │  │  ┌───┐  │  │  ┌───┐  │  │   +   │ │
│     │  │ 📷 │  │  ║  │ 📷 │  ║  │  │ 📷 │  │  │  │ 📷 │  │  │  │ 📷 │  │  │ NEW │ │
│     │  └───┘  │  ║  └───┘  ║  │  └───┘  │  │  └───┘  │  │  └───┘  │  │       │ │
│     │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  └───────┘ │
│     └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘    ↑       │
│                       ↑                                              canAdd    │
│                  selected (blue border, slight scale up)                        │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**With Scroll Buttons (when overflow):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ ┌───┐                                                                     ┌───┐ │
│ │ ◀ │  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐   │ ▶ │ │
│ │   │  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │   │   │ │
│ │   │  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘   │   │ │
│ └───┘                                                                     └───┘ │
│   ↑                                                                         ↑   │
│   scroll left                                                       scroll right│
│   (visible when can scroll left)                        (visible when can scroll)│
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Dragging State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│     ┌─────────┐  ┌·········┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐ │
│     │   0-1   │  ·   2-3   ·  │   4-5   │  │   6-7   │  │   8-9   │  │  NEW  │ │
│     └─────────┘  ·(dragging)·  └─────────┘  └─────────┘  └─────────┘  └───────┘ │
│                  └·········┘                                                    │
│                       │              ┌ · · · · · · ┐                            │
│                       │              · drop target ·                            │
│                       └──────────────· (blue dash) ·                            │
│                                      └ · · · · · · ┘                            │
│                                                                                 │
│  Ghost preview follows cursor                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Thumbnail Hover:**

```
Normal:                         Hover:
┌─────────┐                    ╔═════════╗
│  ┌───┐  │                    ║  ┌───┐  ║
│  │   │  │      ────→         ║  │   │  ║  ← slight elevation
│  └───┘  │                    ║  └───┘  ║     and border
│   0-1   │                    ║   0-1   ║
└─────────┘                    ╚═════════╝
```

**Selected Thumbnail:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│     ┌─────────┐  ╔══════════════╗  ┌─────────┐  ┌─────────┐                    │
│     │         │  ║              ║  │         │  │         │                    │
│     │   0-1   │  ║     2-3      ║  │   4-5   │  │   6-7   │                    │
│     │         │  ║   (selected) ║  │         │  │         │                    │
│     └─────────┘  ╚══════════════╝  └─────────┘  └─────────┘                    │
│                         ↑                                                       │
│                  • Blue border (2px solid primary)                              │
│                  • Slight scale up (transform: scale(1.05))                     │
│                  • Elevated shadow                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│                              ┌─────────────────┐                                │
│                              │                 │                                │
│                              │        +        │                                │
│                              │    Add First    │                                │
│                              │     Spread      │                                │
│                              │                 │                                │
│                              └─────────────────┘                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

### 3.1 SpreadThumbnail (shared)

📄 **Doc:** [03-03-05-spread-thumbnail.md](component/editor-page/03-03-05-spread-thumbnail.md)

**Mục đích:** Thumbnail preview của một spread.

**Props:**

```typescript
interface SpreadThumbnailProps {
  spread: SpreadViewSpread;
  index: number;
  isSelected: boolean;
  currentLanguage: Language;
  displayField: 'art_note' | 'visual_description';
  isDragging?: boolean;
  isDropTarget?: boolean;
  isDragEnabled?: boolean;
  size: 'small' | 'medium' | 'large';

  onClick: () => void;
  onDragStart?: () => void;
  onDragOver?: () => void;
  onDragEnd?: () => void;
}
```

### 3.2 NewSpreadButton

**Mục đích:** Button để add new spread.

**Props:**

```typescript
interface NewSpreadButtonProps {
  onClick: () => void;
}
```

**Visual:**

```
┌───────────────┐
│               │
│       +       │
│      NEW      │
│               │
└───────────────┘
```

### 3.3 ScrollButton

**Mục đích:** Arrow button để scroll filmstrip.

**Props:**

```typescript
interface ScrollButtonProps {
  direction: 'left' | 'right';
  disabled: boolean;
  onClick: () => void;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Horizontal Scroll**
Native horizontal scroll với scroll-snap cho smooth snapping. Buttons visible only when can scroll in that direction.

**Drag-Drop in Filmstrip**
Always enabled (cả dummy và finalize mode). Lý do: User cần flexibility để sắp xếp lại thứ tự spreads.

**Auto-scroll Selected**
Khi selected spread thay đổi, auto-scroll để đảm bảo visible. Lý do: UX tốt hơn khi navigate với keyboard.

**Fixed Thumbnail Size**
Thumbnails có fixed size trong filmstrip (không responsive như grid). Lý do: Consistent navigation, predictable layout.

### 4.2 Scroll Configuration

```typescript
const FILMSTRIP_CONFIG = {
  thumbnailWidth: 100,
  thumbnailHeight: 80,
  gap: 12,
  scrollAmount: 2,  // Number of thumbnails per scroll
  scrollBehavior: 'smooth' as const,
};

function scrollThumbnailIntoView(index: number, containerRef: RefObject<HTMLDivElement>) {
  const container = containerRef.current;
  if (!container) return;

  const thumbnail = container.children[index] as HTMLElement;
  if (!thumbnail) return;

  const containerRect = container.getBoundingClientRect();
  const thumbnailRect = thumbnail.getBoundingClientRect();

  if (thumbnailRect.left < containerRect.left) {
    // Scroll left
    container.scrollBy({
      left: thumbnailRect.left - containerRect.left - FILMSTRIP_CONFIG.gap,
      behavior: FILMSTRIP_CONFIG.scrollBehavior,
    });
  } else if (thumbnailRect.right > containerRect.right) {
    // Scroll right
    container.scrollBy({
      left: thumbnailRect.right - containerRect.right + FILMSTRIP_CONFIG.gap,
      behavior: FILMSTRIP_CONFIG.scrollBehavior,
    });
  }
}
```

### 4.3 Drag-Drop Implementation

```typescript
// Using @dnd-kit/sortable for drag-drop
interface DragState {
  activeId: string | null;
  overId: string | null;
}

function handleDragStart(event: DragStartEvent) {
  setDraggedIndex(Number(event.active.id));
}

function handleDragOver(event: DragOverEvent) {
  setDropTargetIndex(event.over ? Number(event.over.id) : null);
}

function handleDragEnd(event: DragEndEvent) {
  const { active, over } = event;

  if (over && active.id !== over.id) {
    const oldIndex = Number(active.id);
    const newIndex = Number(over.id);
    onDragEnd({ oldIndex, newIndex });
  }

  setDraggedIndex(null);
  setDropTargetIndex(null);
}
```

### 4.4 Styling

```css
/* ========== Base Container ========== */
.spread-thumbnail-list {
  display: flex;
  background: var(--surface-secondary);
}

/* ========== Horizontal Layout (Filmstrip) ========== */
.spread-thumbnail-list.layout-horizontal {
  align-items: center;
  height: 120px;
  border-top: 1px solid var(--border-color);
  padding: 0 8px;
}

.spread-thumbnail-list.layout-horizontal .scroll-container {
  display: flex;
  gap: 12px;
  overflow-x: auto;
  overflow-y: hidden;
  scroll-snap-type: x mandatory;
  scrollbar-width: none;
  -ms-overflow-style: none;
  padding: 8px 0;
}

.spread-thumbnail-list.layout-horizontal .scroll-container::-webkit-scrollbar {
  display: none;
}

.spread-thumbnail-list.layout-horizontal .thumbnail {
  flex-shrink: 0;
  width: 100px;
  height: 80px;
  scroll-snap-align: start;
}

/* ========== Grid Layout (Vertical Scroll) ========== */
.spread-thumbnail-list.layout-grid {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.spread-thumbnail-list.layout-grid .scroll-container {
  display: grid;
  grid-template-columns: repeat(var(--columns, 4), 1fr);
  gap: 16px;
}

.spread-thumbnail-list.layout-grid .thumbnail {
  aspect-ratio: 2 / 1.5;  /* Spread aspect ratio */
  width: 100%;
}

/* ========== Shared Thumbnail Styles ========== */
.thumbnail {
  cursor: pointer;
  border-radius: 8px;
  border: 2px solid transparent;
  transition: transform 0.15s, box-shadow 0.15s, border-color 0.15s;
}

.thumbnail:hover {
  border-color: var(--border-hover);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.thumbnail.selected {
  border-color: var(--primary);
  transform: scale(1.02);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.thumbnail.dragging {
  opacity: 0.5;
}

.thumbnail.drop-target {
  border: 2px dashed var(--primary);
}

/* ========== Scroll Buttons (Horizontal only) ========== */
.scroll-button {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--surface);
  border: 1px solid var(--border-color);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.scroll-button:hover {
  background: var(--surface-hover);
}

.scroll-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

### 4.5 Accessibility

```typescript
const filmstripA11y = {
  role: 'listbox',
  'aria-label': 'Spread thumbnails',
  'aria-orientation': 'horizontal',
};

const thumbnailA11y = (index: number, isSelected: boolean, pageRange: string) => ({
  role: 'option',
  'aria-selected': isSelected,
  'aria-label': `Spread ${index + 1}, pages ${pageRange}`,
  tabIndex: isSelected ? 0 : -1,
});

const scrollButtonA11y = (direction: 'left' | 'right', disabled: boolean) => ({
  'aria-label': `Scroll ${direction}`,
  'aria-disabled': disabled,
  tabIndex: disabled ? -1 : 0,
});

// Keyboard navigation
function handleKeyDown(e: KeyboardEvent) {
  switch (e.key) {
    case 'ArrowLeft':
      selectPreviousSpread();
      break;
    case 'ArrowRight':
      selectNextSpread();
      break;
    case 'Home':
      selectFirstSpread();
      break;
    case 'End':
      selectLastSpread();
      break;
  }
}
```

### 4.6 Touch Support

```typescript
// Swipe gestures for mobile
const swipeHandlers = useSwipeable({
  onSwipedLeft: () => scrollRight(),
  onSwipedRight: () => scrollLeft(),
  trackMouse: false,
  trackTouch: true,
  delta: 50,  // Minimum swipe distance
});
```
