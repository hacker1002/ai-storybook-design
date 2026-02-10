# SpreadThumbnail: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SpreadThumbnail                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  SpreadContainer (clickable and drag & drop)                      │  │
│  │  ┌─────────────────────────┬─────────────────────────┐            │  │
│  │  │      LeftPage           │        RightPage        │            │  │
│  │  │  ┌─────────────────┐    │   ┌─────────────────┐   │            │  │
│  │  │  │ PageNumber (0)  │    │   │ PageNumber (1)  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  │  ┌───────────┐  │    │   │  ┌───────────┐  │   │            │  │
│  │  │  │  │ DummyImage│  │    │   │  │ DummyImage│  │   │            │  │
│  │  │  │  │ Placeholder│ │    │   │  │ Placeholder│ │   │            │  │
│  │  │  │  └───────────┘  │    │   │  └───────────┘  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  │  ┌───────────┐  │    │   │  ┌───────────┐  │   │            │  │
│  │  │  │  │ Textbox   │  │    │   │  │ Textbox   │  │   │            │  │
│  │  │  │  │ Overlay   │  │    │   │  │ Overlay   │  │   │            │  │
│  │  │  │  └───────────┘  │    │   │  └───────────┘  │   │            │  │
│  │  │  └─────────────────┘    │   └─────────────────┘   │            │  │
│  │  └─────────────────────────┴─────────────────────────┘            │  │
│  │                     SpreadLabel: "Page 1-2"                       │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                    ┌─────────────────────────────┐
                    │      SnapshotStore          │
                    │  ┌───────────────────────┐  │
                    │  │ useDummySpreadById()  │  │
                    │  │ useSpreadById()       │  │
                    │  └───────────────────────┘  │
                    └──────────────┬──────────────┘
                                   │ (selectors)
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           SpreadThumbnail                                │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Props: spreadId, mode, dummyType?, isSelected,                    │  │
│  │         displayMode, isDragEnabled, onClick, onDragStart, onDragEnd│  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Local State: (none - read-only thumbnail)                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  Derived (from selector data):                                      │ │
│  │  • leftPageNum, rightPageNum, label                                 │ │
│  │  • scaledGeometry per image/textbox                                 │ │
│  │  • displayText per image (art_note | visual_description)            │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Layer Stack (Z-Index)

```
┌─────────────────────────────────────────────┐
│  Layer 3: Textboxes (overlay on images)     │  ← z-index: 2
├─────────────────────────────────────────────┤
│  Layer 2: Dummy Images (rectangles w/ text) │  ← z-index: 1
├─────────────────────────────────────────────┤
│  Layer 1: Page Background                   │  ← z-index: 0
└─────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Thumbnail preview của một spread trong grid. Hiển thị dummy images dạng khung chữ nhật với text, textboxes overlay lên trên images, theo đúng geometry và typography.

**Shared Types:**

```typescript
type SpreadViewMode = 'dummy' | 'finalize';
type DummyType = 'prose' | 'poetry';

interface Geometry {
  x: number;      // Left position (%)
  y: number;      // Top position (%)
  w: number;      // Width (%)
  h: number;      // Height (%)
  rotation?: number;
}

interface Typography {
  size: number;
  weight: number;
  style: string;
  family: string;
  color: string;
  line_height: number;
  letter_spacing: number;
  decoration: string;
  text_align: string;
  text_transform: string;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SpreadThumbnailProps {
  spreadId: string;
  mode: SpreadViewMode;
  dummyId?: string;                      // Required when mode='dummy'
  isSelected: boolean;
  displayMode: 'dummy' | 'finalize';
  isDragEnabled?: boolean;
  isDragging?: boolean;
  onClick?: () => void;
  onDragStart?: () => void;
  onDragEnd?: (event: DragEndEvent) => void;
}

// No local state - read-only thumbnail
```

**Store Integration:**

```typescript
// SnapshotStore Selectors (conditional based on mode)
spread = mode === 'dummy'
  ? useDummySpreadById(dummyId!, spreadId)
  : useSpreadById(spreadId);

// Actions: none (read-only component)
```

### 2.3 Render Logic (pseudo)

```
SpreadThumbnail:
  // Get spread data from SnapshotStore
  spread = mode === 'dummy' ? useDummySpreadById(dummyId, spreadId) : useSpreadById(spreadId)

  leftPageNum = spread.left_page.number
  rightPageNum = spread.right_page.number
  label = `Page ${leftPageNum + 1}-${rightPageNum + 1}`

  containerClass = determineContainerClass(isSelected, isDragging, isDragEnabled)
  cursor = isDragEnabled ? 'grab' : (onClick ? 'pointer' : 'default')

  RENDER Container onClick={onClick} draggable={isDragEnabled}:
    RENDER SpreadPages (position: relative):

      RENDER LeftPage (position: relative, width: 50%):
        RENDER PageNumber: leftPageNum (corner badge)

        // Layer 1: Images
        FOR EACH image IN spread.images:
          IF isOnLeftPage(image.geometry):
            scaledGeo = scaleToThumbnail(image.geometry, 'left')
            displayText = getDisplayText(image, displayMode)
            RENDER DummyImagePlaceholder at scaledGeo with displayText

        // Layer 2: Textboxes (overlay)
        FOR EACH textbox IN spread.textboxes:
          IF isOnLeftPage(textbox.geometry):
            scaledGeo = scaleToThumbnail(textbox.geometry, 'left')
            scaledTypo = scaleTypography(textbox.typography)
            RENDER TextboxOverlay at scaledGeo with scaledTypo

      RENDER RightPage (same pattern as LeftPage for right side)

    RENDER SpreadLabel: label

  getDisplayText(image, mode):
    IF mode === 'dummy':
      RETURN truncate(image.art_note, 30)
    ELSE:
      RETURN truncate(image.visual_description || '', 30)
```

### 2.4 Visual

**Dummy Mode (art_note displayed):**

```
┌─────────────────────────────────────┐
│ 0           │           1           │  ← Page numbers
│ ┌─────────────────────────────────┐ │
│ │  ┌───────────────┐              │ │
│ │  │  "A cat       │  ┌─────────┐ │ │  ← DummyImage placeholder
│ │  │  sitting..."  │  │ Once    │ │ │    with art_note text
│ │  │               │  │ upon a  │ │ │
│ │  └───────────────┘  │ time... │ │ │  ← Textbox overlay
│ │                     └─────────┘ │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
          Page 1-2
```

**Finalize Mode (visual_description displayed):**

```
┌─────────────────────────────────────┐
│ 0           │           1           │
│ ┌─────────────────────────────────┐ │
│ │  ┌───────────────┐              │ │
│ │  │ "Detailed     │  ┌─────────┐ │ │  ← visual_description
│ │  │  scene of     │  │ Once    │ │ │
│ │  │  tabby..."    │  │ upon... │ │ │
│ │  └───────────────┘  └─────────┘ │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
          Page 1-2
```

**Visual States:**

```
Normal:                     Selected:                   Dragging:
┌───────────────────┐       ╔═══════════════════╗       ┌·····················┐
│ 0     │     1     │       ║ 0     │     1     ║       · 0     │     1     ·
│ ┌───┐ │           │       ║ ┌───┐ │           ║       · ┌───┐ │           ·
│ │art│ │  text     │       ║ │art│ │  text     ║       · │art│ │  text     ·
│ └───┘ │           │       ║ └───┘ │           ║       · └───┘ │           ·
└───────────────────┘       ╚═══════════════════╝       └·····················┘
   Page 1-2                    Page 1-2                    Page 1-2
                               ↑ Blue border               ↑ Opacity 0.5
                                                           ↑ Dashed border

Hover (clickable):          Read-only:
┌───────────────────┐       ┌───────────────────┐
│ 0     │     1     │       │ 0     │     1     │
│ ┌───┐ │           │       │ ┌───┐ │           │
│ │art│ │  text     │       │ │vis│ │  text     │
│ └───┘ │           │       │ └───┘ │           │
└───────────────────┘       └───────────────────┘
   Page 1-2                    Page 1-2
   ↑ Shadow/lift               ↑ No hover effect
   ↑ cursor: pointer           ↑ cursor: default
```

---

## 3. Child Components Interface

> **Lưu ý:** SpreadThumbnail chứa các DOM elements cơ bản, không phải full React components.
> Các elements dưới đây được render inline, không cần file design riêng.

### 3.1 DummyImagePlaceholder (element)

**Mục đích:** Hiển thị vị trí và kích thước của image slot dạng placeholder rectangle với text bên trong.

**Props (inline):**

```typescript
interface DummyImagePlaceholderProps {
  geometry: Geometry;          // Scaled to thumbnail
  displayText: string;         // art_note | visual_description (truncated)
}
```

**Visual:**

```
┌───────────────────────┐
│  "A cat sitting       │  ← Truncated text
│   on a cozy..."       │
└───────────────────────┘
  ↑ Background: light gray
  ↑ Border: dashed
  ↑ Text: centered, small
```

### 3.2 TextboxOverlay (element)

**Mục đích:** Hiển thị textbox với geometry và typography được scale xuống thumbnail size.

**Props (inline):**

```typescript
interface TextboxOverlayProps {
  geometry: Geometry;          // Scaled to thumbnail
  typography: Typography;      // Scaled to thumbnail
  text: string;                // langContent.text (truncated)
}
```

**Visual:**

```
┌─────────────┐
│ Once upon   │  ← Scaled font size, weight, color
│ a time...   │  ← Text alignment preserved
└─────────────┘
  ↑ Semi-transparent background
  ↑ Positioned by geometry
```

### 3.3 Elements Summary

| Element | Type | Description |
|---------|------|-------------|
| Container | `<div>` | Clickable/draggable wrapper |
| SpreadPages | `<div>` | Flex container cho 2 pages |
| LeftPage / RightPage | `<div>` | Page container với relative positioning |
| PageNumber | `<span>` | Small number badge ở góc |
| DummyImagePlaceholder | `<div>` | Rectangle positioned absolute |
| TextboxOverlay | `<div>` | Positioned absolute, scaled typography |
| SpreadLabel | `<span>` | "Page X-Y" label below spread |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**ID-based Store Subscription**
Component subscribe theo `spreadId` thay vì nhận `spread` object từ parent. Điều này đảm bảo mỗi thumbnail chỉ re-render khi dữ liệu của chính nó thay đổi, không phải khi sibling spreads thay đổi.

**Display Mode Determines Content**
- `'dummy'` mode: shows `image.art_note`
- `'finalize'` mode: shows `image.visual_description`

**Geometry-based Positioning**
All images và textboxes được position bằng CSS `position: absolute` với percentage values. Items với `geometry.x < 50` thuộc left page, `>= 50` thuộc right page.

**Typography Scaling**
Font sizes và typography values được scale proportionally xuống thumbnail size. Minimum font size: 6px để đảm bảo readability.

### 4.2 Geometry Interfaces

```typescript
interface CSSGeometry {
  left: string;   // e.g., "10%"
  top: string;
  width: string;
  height: string;
}

interface ScaledTypography {
  fontSize: number;      // Min 6px
  fontWeight: number;
  color: string;
  textAlign: string;
}
```

### 4.3 Helper Function Signatures

```typescript
// Scale geometry from percentage to thumbnail-relative percentage
scaleToThumbnail(geometry: Geometry, page: 'left' | 'right'): CSSGeometry

// Scale typography for thumbnail display
scaleTypography(typography: Typography): ScaledTypography

// Determine which page an item belongs to
isOnLeftPage(geometry: Geometry): boolean

// Truncate text for display
truncateText(text: string, maxLength: number): string
```

### 4.4 Accessibility

| Attribute | Value |
|-----------|-------|
| role | `listitem` (drag) / `article` (click) |
| tabIndex | `0` (clickable) / `-1` (not) |
| aria-selected | `isSelected` |
| aria-label | `Spread ${index + 1}, ${label}` |
| aria-grabbed | `isDragging` |
| keyboard | Enter/Space triggers onClick |

### 4.5 Performance

- **CSS containment:** `contain: layout style paint` trên container
- **No image loading:** Dùng CSS-only placeholders
- **Text truncation:** Truncate khi render, không dynamic
- **ID-based subscription:** Chỉ re-render khi spread data thay đổi

---

