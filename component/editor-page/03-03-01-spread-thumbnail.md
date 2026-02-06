# SpreadThumbnail: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        SpreadThumbnail                           │
│  ┌───────────────────────┬───────────────────────┐              │
│  │      LeftPage         │      RightPage        │              │
│  │  ┌─────────────────┐  │  ┌─────────────────┐  │              │
│  │  │ PageNumber (0)  │  │  │ PageNumber (1)  │  │              │
│  │  │                 │  │  │                 │  │              │
│  │  │  [ImagePreview] │  │  │  [TextPreview]  │  │              │
│  │  │       ▒▒▒       │  │  │   "text..."     │  │              │
│  │  │                 │  │  │                 │  │              │
│  │  └─────────────────┘  │  └─────────────────┘  │              │
│  └───────────────────────┴───────────────────────┘              │
│                    SpreadLabel: "Page 1-2"                       │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                        SpreadThumbnail                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Props: spread, index, isSelected, currentLanguage         │  │
│  │  Callback: onClick                                         │  │
│  └────────────────────────────────────────────────────────────┘  │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Derived:                                                   │ │
│  │  • leftPageNum = spread.left_page.number                    │ │
│  │  • rightPageNum = spread.right_page.number                  │ │
│  │  • images = spread.images (positions on left/right)         │ │
│  │  • texts = spread.textboxes[currentLanguage.code]           │ │
│  │  • label = `Page ${leftPageNum + 1}-${rightPageNum + 1}`    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Thumbnail preview của một spread trong grid. Hiển thị 2 pages side-by-side với page numbers và simplified content preview (images as placeholders, text truncated).

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content lấy từ `textbox[currentLanguage.code].text`

### 2.2 Interface

```typescript
interface SpreadThumbnailProps {
  spread: DummySpread;
  index: number;
  isSelected: boolean;
  currentLanguage: Language;
  onClick: () => void;
}
```

### 2.3 Render Logic (pseudo)

```
SpreadThumbnail:
  leftPageNum = spread.left_page.number
  rightPageNum = spread.right_page.number
  label = `Page ${leftPageNum + 1}-${rightPageNum + 1}`

  containerClass = isSelected ? 'selected' : 'normal'

  RENDER Container onClick={onClick}:
    RENDER SpreadPages:
      RENDER LeftPage:
        RENDER PageNumber: leftPageNum
        FOR EACH image IN spread.images WHERE onLeftPage(image):
          RENDER ImagePlaceholder (scaled position)
        FOR EACH textbox IN spread.textboxes WHERE onLeftPage(textbox):
          text = textbox[currentLanguage.code]?.text
          RENDER TextPreview: truncate(text, 20)

      RENDER RightPage:
        RENDER PageNumber: rightPageNum
        FOR EACH image IN spread.images WHERE onRightPage(image):
          RENDER ImagePlaceholder (scaled position)
        FOR EACH textbox IN spread.textboxes WHERE onRightPage(textbox):
          text = textbox[currentLanguage.code]?.text
          RENDER TextPreview: truncate(text, 20)

    RENDER SpreadLabel: label
```

### 2.4 Visual

**Normal State:**

```
┌─────────────┐
│ 0   │   1   │  ← Page numbers (small, corner)
│ ┌─┐ │       │
│ │▒│ │ Once  │  ← Image placeholder, Text preview
│ └─┘ │ upon..│
└─────────────┘
   Page 1-2       ← Label
```

**Selected State:**

```
╔═════════════╗   ← Highlighted border (blue/primary color)
║ 0   │   1   ║
║ ┌─┐ │       ║
║ │▒│ │ Once  ║
║ └─┘ │ upon..║
╚═════════════╝
   Page 1-2
```

**Hover State:**

```
┌─────────────┐
│ 0   │   1   │  ← Subtle shadow/lift effect
│ ┌─┐ │       │
│ │▒│ │ Once  │
│ └─┘ │ upon..│
└─────────────┘   ← cursor: pointer
   Page 1-2
```

**Content Variations:**

```
Text only:           Image only:          Mixed:               Empty:
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ 0   │   1   │      │ 0   │   1   │      │ 0   │   1   │      │ 0   │   1   │
│     │       │      │ ┌───────┐   │      │ ┌─┐ │       │      │     │       │
│ The │ brave │      │ │  ▒▒▒  │   │      │ │▒│ │ text  │      │     │       │
│ cat │ dog.. │      │ └───────┘   │      │ └─┘ │       │      │     │       │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** SpreadThumbnail chỉ chứa các element cơ bản.
> Không cần thiết kế grandchild components (level 4).

### 3.1 Elements (không phải components riêng)

| Element | Type | Notes |
|---------|------|-------|
| Container | `<div>` | Clickable wrapper với border styles |
| SpreadPages | `<div>` | Flex container cho 2 pages |
| LeftPage / RightPage | `<div>` | Page container với relative positioning |
| PageNumber | `<span>` | Small number ở góc trong |
| ImagePlaceholder | `<div>` | Gray box đại diện cho image |
| TextPreview | `<span>` | Truncated text preview |
| SpreadLabel | `<span>` | "Page X-Y" label |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Simplified Content Preview**
Images hiển thị là gray placeholder boxes, không load actual image. Text được truncate. Lý do: Performance với nhiều spreads.

**Scaled Geometry**
Image/text positions được scale từ actual geometry xuống thumbnail size. Lý do: Maintain relative positions.

**Page Number Position**
Page numbers hiển thị ở góc trong (left page → top-left, right page → top-right). Lý do: Mimic book spread.

**Language-aware Text**
Text lấy từ `textbox[currentLanguage.code]`, fallback empty nếu không có. Lý do: Consistent với global language setting.

### 4.2 Geometry Scaling

```typescript
// Scale factor from actual page size to thumbnail
const THUMBNAIL_WIDTH = 120; // px per page
const ACTUAL_PAGE_WIDTH = 210; // mm (A4)
const SCALE = THUMBNAIL_WIDTH / ACTUAL_PAGE_WIDTH;

function scaleGeometry(geometry: Geometry): ScaledGeometry {
  return {
    x: geometry.x * SCALE,
    y: geometry.y * SCALE,
    w: geometry.w * SCALE,
    h: geometry.h * SCALE,
  };
}
```

### 4.3 Accessibility

```
Container:
  role="button"
  tabIndex={0}
  aria-selected={isSelected}
  aria-label={`Spread ${index + 1}, ${label}`}
  onKeyDown: Enter/Space → onClick
```
