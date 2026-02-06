# SpreadThumbnail: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SpreadThumbnail                                │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  SpreadContainer (clickable in DummyView, not in FinalizationView) │  │
│  │  ┌─────────────────────────┬─────────────────────────┐            │  │
│  │  │      LeftPage           │        RightPage        │            │  │
│  │  │  ┌─────────────────┐    │   ┌─────────────────┐   │            │  │
│  │  │  │ PageNumber (0)  │    │   │ PageNumber (1)  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  │  ┌───────────┐  │    │   │  ┌───────────┐  │   │            │  │
│  │  │  │  │ DummyImage│  │    │   │  │ DummyImage│  │   │            │  │
│  │  │  │  │ ┌───────┐ │  │    │   │  │ ┌───────┐ │  │   │            │  │
│  │  │  │  │ │"art   │ │  │    │   │  │ │"visual│ │  │   │            │  │
│  │  │  │  │ │note"  │ │  │    │   │  │ │desc"  │ │  │   │            │  │
│  │  │  │  │ └───────┘ │  │    │   │  │ └───────┘ │  │   │            │  │
│  │  │  │  └───────────┘  │    │   │  └───────────┘  │   │            │  │
│  │  │  │                 │    │   │                 │   │            │  │
│  │  │  │  ┌───────────┐  │    │   │  ┌───────────┐  │   │            │  │
│  │  │  │  │ Textbox   │  │    │   │  │ Textbox   │  │   │            │  │
│  │  │  │  │ (overlay) │  │    │   │  │ (overlay) │  │   │            │  │
│  │  │  │  └───────────┘  │    │   │  └───────────┘  │   │            │  │
│  │  │  └─────────────────┘    │   └─────────────────┘   │            │  │
│  │  └─────────────────────────┴─────────────────────────┘            │  │
│  │                     SpreadLabel: "Page 1-2"                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Layer Stack (Z-Index)

```
┌─────────────────────────────────────────────┐
│  Layer 3: Textboxes (overlay on images)     │  ← z-index: 2
├─────────────────────────────────────────────┤
│  Layer 2: Dummy Images (rectangles w/ text) │  ← z-index: 1
├─────────────────────────────────────────────┤
│  Layer 1: Page Background                   │  ← z-index: 0
└─────────────────────────────────────────────┘
```

### 1.3 Data Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                           SpreadThumbnail                             │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Props: spread, index, isSelected, currentLanguage,            │  │
│  │         displayMode, isDragEnabled, onClick                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Derived:                                                       │ │
│  │  • leftPageNum = spread.left_page.number                        │ │
│  │  • rightPageNum = spread.right_page.number                      │ │
│  │  • label = `Page ${leftPageNum + 1}-${rightPageNum + 1}`        │ │
│  │                                                                 │ │
│  │  FOR EACH image IN spread.images:                               │ │
│  │    • scaledGeometry = scaleGeometry(image.geometry)             │ │
│  │    • displayText = displayMode === 'dummy'                      │ │
│  │        ? image.art_note                                         │ │
│  │        : image.visual_description                               │ │
│  │                                                                 │ │
│  │  FOR EACH textbox IN spread.textboxes:                          │ │
│  │    • langContent = textbox[currentLanguage.code]                │ │
│  │    • scaledGeometry = scaleGeometry(langContent.geometry)       │ │
│  │    • scaledTypography = scaleTypography(langContent.typography) │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Thumbnail preview của một spread trong grid. Hiển thị:
- Dummy images dạng khung chữ nhật với text bên trong (art_note hoặc visual_description)
- Images được đặt theo đúng geometry (vị trí, kích thước)
- Textboxes overlay lên trên images, theo đúng geometry và typography

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content lấy từ `textbox[currentLanguage.code]`

**Display Modes:**
- `'dummy'`: Image placeholder hiển thị `art_note`
- `'finalize'`: Image placeholder hiển thị `visual_description`

### 2.2 Interface

```typescript
interface SpreadThumbnailProps {
  spread: DummySpread;
  index: number;
  isSelected: boolean;
  currentLanguage: Language;
  displayMode: 'dummy' | 'finalize';
  isDragEnabled?: boolean;
  isDragging?: boolean;
  onClick?: () => void;
  onDragStart?: () => void;
  onDragEnd?: (event: DragEndEvent) => void;
}

// Geometry in percentage (0-100)
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

### 2.3 Render Logic (pseudo)

```
SpreadThumbnail:
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
            RENDER DummyImagePlaceholder:
              - position: absolute
              - left: scaledGeo.x%
              - top: scaledGeo.y%
              - width: scaledGeo.w%
              - height: scaledGeo.h%
              - content: displayText (truncated)

        // Layer 2: Textboxes (overlay)
        FOR EACH textbox IN spread.textboxes:
          langContent = textbox[currentLanguage.code]
          IF langContent AND isOnLeftPage(langContent.geometry):
            scaledGeo = scaleToThumbnail(langContent.geometry, 'left')
            scaledTypo = scaleTypography(langContent.typography)
            RENDER TextboxOverlay:
              - position: absolute
              - left: scaledGeo.x%
              - top: scaledGeo.y%
              - width: scaledGeo.w%
              - height: scaledGeo.h%
              - fontSize: scaledTypo.size
              - fontWeight: scaledTypo.weight
              - color: scaledTypo.color
              - textAlign: scaledTypo.text_align
              - content: langContent.text (truncated)

      RENDER RightPage (position: relative, width: 50%):
        // Same pattern as LeftPage but for right side
        ...

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
│ │  │ ┌───────────┐ │              │ │
│ │  │ │  "A cat   │ │  ┌─────────┐ │ │  ← DummyImage placeholder
│ │  │ │  sitting  │ │  │ Once    │ │ │    with art_note text
│ │  │ │  on..."   │ │  │ upon a  │ │ │
│ │  │ └───────────┘ │  │ time... │ │ │  ← Textbox overlay
│ │  └───────────────┘  └─────────┘ │ │
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
│ │  │ ┌───────────┐ │              │ │
│ │  │ │ "Detailed │ │  ┌─────────┐ │ │  ← visual_description
│ │  │ │  scene of │ │  │ Once    │ │ │
│ │  │ │  tabby..."│ │  │ upon a  │ │ │
│ │  │ └───────────┘ │  │ time... │ │ │
│ │  └───────────────┘  └─────────┘ │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
          Page 1-2
```

**Geometry Layout Example:**

```
Image geometry: { x: 10, y: 10, w: 40, h: 60 }
Textbox geometry: { x: 55, y: 20, w: 35, h: 40 }

┌─────────────────────────────────────┐
│ 0           │           1           │
│ ┌───────────────────────────────────┐
│ │                                   │
│ │  ┌──────────┐   ┌───────────┐     │
│ │  │          │   │ Textbox   │     │  ← Positioned by geometry %
│ │  │  Dummy   │   │ text here │     │
│ │  │  Image   │   │           │     │
│ │  │ "art_    │   └───────────┘     │
│ │  │  note"   │                     │
│ │  │          │                     │
│ │  └──────────┘                     │
│ │   ↑                               │
│ │   x:10%, y:10%, w:40%, h:60%      │
│ └───────────────────────────────────┘
└─────────────────────────────────────┘
```

**States:**

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

Hover (clickable):          Read-only (FinalizationView):
┌───────────────────┐       ┌───────────────────┐
│ 0     │     1     │       │ 0     │     1     │
│ ┌───┐ │           │       │ ┌───┐ │           │
│ │art│ │  text     │       │ │vis│ │  text     │  ← visual_description
│ └───┘ │           │       │ └───┘ │           │
└───────────────────┘       └───────────────────┘
   Page 1-2                    Page 1-2
   ↑ Shadow/lift               ↑ No hover effect
   ↑ cursor: pointer           ↑ cursor: default
```

**Content Variations:**

```
Full spread image:           Text-heavy:               Multiple images:
┌───────────────────┐       ┌───────────────────┐     ┌───────────────────┐
│ 0     │     1     │       │ 0     │     1     │     │ 0     │     1     │
│ ┌───────────────┐ │       │       ┌─────────┐ │     │ ┌───┐ │ ┌───┐     │
│ │ "Full spread  │ │       │       │  Long   │ │     │ │img│ │ │img│     │
│ │  scene of..." │ │       │       │  text   │ │     │ │ 1 │ │ │ 2 │     │
│ └───────────────┘ │       │       │  here   │ │     │ └───┘ │ └───┘     │
│                   │       │       └─────────┘ │     │   ┌─────────┐     │
└───────────────────┘       └───────────────────┘     │   │ textbox │     │
                                                      └───────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** SpreadThumbnail chứa các element cơ bản, không phải full components.

### 3.1 Elements (không phải components riêng)

| Element | Type | Description |
|---------|------|-------------|
| Container | `<div>` | Clickable/draggable wrapper với border styles |
| SpreadPages | `<div>` | Flex container cho 2 pages, position: relative |
| LeftPage / RightPage | `<div>` | Page container với relative positioning |
| PageNumber | `<span>` | Small number badge ở góc |
| DummyImagePlaceholder | `<div>` | Rectangle với background color, positioned absolute |
| DummyImageText | `<span>` | Truncated art_note/visual_description text |
| TextboxOverlay | `<div>` | Positioned absolute, with scaled typography |
| TextboxContent | `<span>` | Truncated text with scaled styles |
| SpreadLabel | `<span>` | "Page X-Y" label below spread |

### 3.2 DummyImagePlaceholder

**Purpose:** Hiển thị vị trí và kích thước của image slot dạng placeholder.

**Visual:**
```
┌───────────────────────┐
│ ┌───────────────────┐ │
│ │  "A cat sitting   │ │  ← Truncated art_note/visual_description
│ │   on a cozy..."   │ │
│ └───────────────────┘ │
└───────────────────────┘
  ↑ Background: #f5f5f5 (light gray)
  ↑ Border: 1px dashed #ccc
  ↑ Text: centered, small font, gray color
```

**Styling:**
```css
.dummy-image-placeholder {
  position: absolute;
  background-color: #f5f5f5;
  border: 1px dashed #ccc;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 4px;
  overflow: hidden;
}

.dummy-image-text {
  font-size: 8px;
  color: #666;
  text-align: center;
  line-height: 1.2;
  word-break: break-word;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
}
```

### 3.3 TextboxOverlay

**Purpose:** Hiển thị textbox với geometry và typography được scale xuống thumbnail size.

**Visual:**
```
┌─────────────┐
│ Once upon   │  ← Scaled font size, weight, color
│ a time...   │  ← Text alignment preserved
└─────────────┘
  ↑ Semi-transparent background for readability
  ↑ Positioned exactly according to geometry
```

**Styling:**
```css
.textbox-overlay {
  position: absolute;
  background-color: rgba(255, 255, 255, 0.85);
  border-radius: 2px;
  padding: 2px;
  overflow: hidden;
}

.textbox-content {
  /* Typography applied via inline styles from scaled values */
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Display Mode Determines Content**
- `'dummy'` mode: DummyImagePlaceholder shows `image.art_note`
- `'finalize'` mode: DummyImagePlaceholder shows `image.visual_description`

**Geometry-based Positioning**
All images and textboxes are positioned using CSS `position: absolute` with percentage values derived from geometry data. This preserves relative positions regardless of thumbnail size.

**Layered Rendering**
Images render first (z-index: 1), textboxes overlay on top (z-index: 2). This matches the actual spread composition.

**Typography Scaling**
Font sizes and other typography values are scaled proportionally to thumbnail size. A scale factor converts actual page sizes to thumbnail dimensions.

**Page Split Logic**
Items with `geometry.x < 50` belong to left page, `>= 50` belong to right page. Geometry coordinates are adjusted accordingly.

### 4.2 Geometry Calculations

```typescript
const THUMBNAIL_WIDTH = 240;  // Total width for spread
const PAGE_WIDTH = 120;       // Width per page
const ACTUAL_PAGE_WIDTH = 100; // Percentage base

// Scale geometry from percentage to thumbnail pixels
function scaleToThumbnail(geometry: Geometry, page: 'left' | 'right'): CSSGeometry {
  const xOffset = page === 'left' ? 0 : PAGE_WIDTH;
  const adjustedX = page === 'left'
    ? geometry.x
    : geometry.x - 50; // Adjust for right page

  return {
    left: `${adjustedX}%`,
    top: `${geometry.y}%`,
    width: `${geometry.w}%`,
    height: `${geometry.h}%`,
  };
}

// Scale typography for thumbnail display
function scaleTypography(typography: Typography): ScaledTypography {
  const SCALE_FACTOR = 0.5; // Thumbnail is 50% of reference

  return {
    fontSize: Math.max(6, typography.size * SCALE_FACTOR), // Min 6px
    fontWeight: typography.weight,
    color: typography.color,
    textAlign: typography.text_align,
    // ... other properties
  };
}

// Determine which page an item belongs to
function isOnLeftPage(geometry: Geometry): boolean {
  return geometry.x < 50;
}
```

### 4.3 Text Truncation

```typescript
function truncateText(text: string, maxLength: number): string {
  if (!text) return '';
  if (text.length <= maxLength) return text;
  return text.slice(0, maxLength - 3) + '...';
}

// Usage
const displayText = truncateText(
  displayMode === 'dummy' ? image.art_note : image.visual_description,
  30
);
```

### 4.4 Drag-Drop Support

```typescript
// Drag-drop attributes when isDragEnabled
const dragProps = isDragEnabled ? {
  draggable: true,
  onDragStart: (e) => {
    e.dataTransfer.setData('text/plain', index.toString());
    onDragStart?.();
  },
  onDragEnd: (e) => {
    onDragEnd?.({ oldIndex: index, newIndex: dropTargetIndex });
  },
  style: { cursor: isDragging ? 'grabbing' : 'grab' }
} : {};
```

### 4.5 Accessibility

```typescript
// Accessibility attributes
const a11yProps = {
  role: isDragEnabled ? 'listitem' : 'article',
  tabIndex: onClick ? 0 : -1,
  'aria-selected': isSelected,
  'aria-label': `Spread ${index + 1}, ${label}`,
  'aria-grabbed': isDragging,
  onKeyDown: (e) => {
    if (onClick && (e.key === 'Enter' || e.key === ' ')) {
      e.preventDefault();
      onClick();
    }
  }
};
```

### 4.6 Performance Considerations

**CSS containment:**
```css
.spread-thumbnail {
  contain: layout style paint;
}
```

**No image loading:**
Thumbnails use CSS-only placeholders, no actual images are loaded. This ensures fast rendering even with many spreads.

**Text truncation:**
All text is truncated on render, not dynamically. Prevents expensive re-layouts.
