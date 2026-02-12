# SpreadThumbnail: Component Design

> **Parent:** [SpreadThumbnailList](./03-spread-thumbnail-list.md)
>
> **Note:** Props-driven component. Receives spread data và render props from parent. Uses same render functions as main panel but in view-only mode.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SpreadThumbnail<TSpread>                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  SpreadContainer (clickable and draggable)                        │  │
│  │  ┌─────────────────────────┬─────────────────────────┐            │  │
│  │  │      LeftPage           │        RightPage        │            │  │
│  │  │  ┌─────────────────┐    │   ┌─────────────────┐   │            │  │
│  │  │  │ PageNumber (0)  │    │   │ PageNumber (1)  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  │  ┌───────────┐  │    │   │  ┌───────────┐  │   │            │  │
│  │  │  │  │ rendered  │  │    │   │  │ rendered  │  │   │            │  │
│  │  │  │  │ by render │  │    │   │  │ by render │  │   │            │  │
│  │  │  │  │ props     │  │    │   │  │ props     │  │   │            │  │
│  │  │  │  └───────────┘  │    │   │  └───────────┘  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  └─────────────────┘    │   └─────────────────┘   │            │  │
│  │  └─────────────────────────┴─────────────────────────┘            │  │
│  │                     SpreadLabel: "Page 1-2"                       │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
SpreadThumbnailList<TSpread>
       │
       │ passes spread, spreadIndex, renderItems, render*Item, callbacks
       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         SpreadThumbnail<TSpread>                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Props: spread, spreadIndex, isSelected, renderItems, render*Item  │  │
│  │  Local State: (none - read-only thumbnail)                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  For each item in renderItems:                                      │ │
│  │    - Build view-only context (isSelected=false, handlers=no-op)     │ │
│  │    - Call render*Item(context)                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Layer Stack (Z-Index)

```
┌─────────────────────────────────────────────┐
│  Layer 3: Textboxes (overlay on images)     │  ← z-index: 2
├─────────────────────────────────────────────┤
│  Layer 2: Images                            │  ← z-index: 1
├─────────────────────────────────────────────┤
│  Layer 1: Page Background                   │  ← z-index: 0
└─────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Thumbnail preview của một spread. Uses same render props as main panel but in view-only mode (disabled interactions).

**Shared Types:**

```typescript
type ItemType = 'image' | 'text' | 'object' | 'animation';

interface Geometry {
  x: number;      // Left position (%)
  y: number;      // Top position (%)
  width: number;  // Width (%)
  height: number; // Height (%)
}
```

### 2.2 Interface

**Props:**

```typescript
interface SpreadThumbnailProps<TSpread extends BaseSpread> {
  // === Data (from parent) ===
  spread: TSpread;
  spreadIndex: number;

  // === State ===
  isSelected: boolean;
  size: 'small' | 'medium';

  // === Render configuration (same as parent) ===
  renderItems: ItemType[];
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // === Drag state ===
  isDragEnabled?: boolean;
  isDragging?: boolean;
  isDropTarget?: boolean;

  // === Callbacks ===
  onClick: () => void;
  onDragStart?: () => void;
  onDragOver?: () => void;
  onDragEnd?: () => void;
}
```

**Local State:** None (read-only thumbnail)

### 2.3 Context Building (View-Only)

Thumbnail builds contexts with disabled interactions:

```typescript
buildViewOnlyImageContext(image: SpreadImage, index: number): ImageItemContext<TSpread>
  return {
    item: image,
    itemIndex: index,
    spreadId: spread.id,
    spread: spread,
    isSelected: false,          // Never selected in thumbnail
    isSpreadSelected: false,
    onSelect: () => {},         // No-op
    onUpdate: () => {},         // No-op
    onDelete: () => {},         // No-op
  }

buildViewOnlyTextContext(textbox: SpreadTextbox, index: number): TextItemContext<TSpread>
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
SpreadThumbnail<TSpread>:
  spread = props.spread
  label = `Page ${spread.left_page.number + 1}-${spread.right_page.number + 1}`

  containerClass = determineContainerClass(isSelected, isDragging, isDragEnabled)
  cursor = isDragEnabled ? 'grab' : 'pointer'

  // Scale factor for thumbnail
  scale = size === 'small' ? 0.15 : 0.25

  RENDER Container onClick={onClick} draggable={isDragEnabled}:
    RENDER SpreadPages (position: relative, transform: scale):

      RENDER LeftPage (position: relative, width: 50%):
        RENDER PageNumber: spread.left_page.number (corner badge)

        // Render items for left page
        IF renderItems.includes('image'):
          FOR EACH (image, index) IN spread.images:
            IF isOnLeftPage(image.geometry):
              context = buildViewOnlyImageContext(image, index)
              RENDER props.renderImageItem(context)

        IF renderItems.includes('text'):
          FOR EACH (textbox, index) IN spread.textboxes:
            IF isOnLeftPage(textbox.geometry):
              context = buildViewOnlyTextContext(textbox, index)
              RENDER props.renderTextItem(context)

      RENDER RightPage (same pattern for right side)

    RENDER SpreadLabel: label
```

### 2.5 Visual

**Normal State:**

```
┌───────────────────┐
│ 0     │     1     │
│ ┌───┐ │           │
│ │img│ │  text     │
│ └───┘ │           │
└───────────────────┘
   Page 1-2
```

**Selected:**

```
╔═══════════════════╗
║ 0     │     1     ║
║ ┌───┐ │           ║
║ │img│ │  text     ║
║ └───┘ │           ║
╚═══════════════════╝
   Page 1-2
   ↑ Blue border
```

**Dragging:**

```
┌·····················┐
· 0     │     1     ·
· ┌───┐ │           ·
· │img│ │  text     ·
· └───┘ │           ·
└·····················┘
   Page 1-2
   ↑ Opacity 0.5, dashed border
```

**Drop Target:**

```
┌ · · · · · · · · · ┐
· 0     │     1     ·
· ┌───┐ │           ·
· │img│ │  text     ·
· └───┘ │           ·
└ · · · · · · · · · ┘
   Page 1-2
   ↑ Highlighted dashed border
```

**Hover (clickable):**

```
┌───────────────────┐
│ 0     │     1     │  ← Shadow/lift effect
│ ┌───┐ │           │
│ │img│ │  text     │
│ └───┘ │           │
└───────────────────┘
   Page 1-2
   cursor: pointer
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Props-driven | spread via props | No store coupling |
| Same render props | Reuse from parent | Consistent appearance |
| View-only contexts | Disabled handlers | No editing in thumbnails |
| CSS transform scale | Performance | Hardware accelerated |

### 3.2 Geometry Scaling

```typescript
// Scale geometry for thumbnail display
// Items are rendered at actual positions but container is scaled down
const THUMBNAIL_SCALE = {
  small: 0.15,   // 100×80px result
  medium: 0.25,  // Dynamic based on grid
};
```

### 3.3 Helper Functions

```typescript
// Determine which page an item belongs to
isOnLeftPage(geometry: Geometry): boolean
  return geometry.x + geometry.width / 2 < 50

isOnRightPage(geometry: Geometry): boolean
  return geometry.x + geometry.width / 2 >= 50
```

### 3.4 Accessibility

| Attribute | Value |
|-----------|-------|
| role | `option` |
| tabIndex | `0` (clickable) |
| aria-selected | `isSelected` |
| aria-label | `Spread ${spreadIndex + 1}, pages ${left}-${right}` |
| aria-grabbed | `isDragging` (if drag enabled) |
| keyboard | Enter/Space triggers onClick |

### 3.5 Performance

- **CSS containment:** `contain: layout style paint` on container
- **Transform scale:** Use CSS transform instead of recalculating sizes
- **No image loading:** Consumer's render function handles image display
- **Memoize contexts:** Prevent unnecessary re-renders

---
