# EditableImage: Component Design

> **Parent:** [SpreadEditorPanel](./02-spread-editor-panel.md)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        EditableImage                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    ImageContainer                         │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  IF generated_image_url:                            │  │  │
│  │  │    <img src={url} />                                │  │  │
│  │  │  ELSE:                                              │  │  │
│  │  │    ImagePlaceholder                                 │  │  │
│  │  │    ┌───────────────────────────────────────────┐    │  │  │
│  │  │    │  🖼️ Icon                                  │    │  │  │
│  │  │    │  "A fluffy orange cat sitting..."         │    │  │  │
│  │  │    └───────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        EditableImage                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Props: image, index, isSelected, displayField, isEditable │ │
│  │  Callbacks: onSelect                                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Local State: isHovered                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Display Logic:                                            │ │
│  │  • IF generated_image_url → render <img>                   │ │
│  │  • ELSE → render placeholder with displayField content     │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Image placeholder trong canvas. Handles selection only. Displays art_note hoặc visual_description khi chưa có generated image.

> **Note:** Drag/resize được handle bởi SelectionFrame overlay, không phải component này.

**Shared Types:**

```typescript
interface SpreadViewImage {
  geometry: Geometry;
  art_note?: string;
  visual_description?: string;
  generated_image_url?: string;
}

interface Geometry {
  x: number;
  y: number;
  width: number;
  height: number;
}
```

### 2.2 Interface

**Props:**

```typescript
interface EditableImageProps {
  image: SpreadViewImage;
  index: number;
  isSelected: boolean;
  displayField: 'art_note' | 'visual_description';
  isEditable: boolean;

  onSelect: () => void;
  // NO onDrag/onResize - handled by SelectionFrame
}
```

**Local State:**

```typescript
interface EditableImageState {
  isHovered: boolean;
  isLoading: boolean;    // Image loading state
  hasError: boolean;     // Fallback to placeholder on error
}
```

**Store Integration:** None (receives data via props)

### 2.3 Display Logic

```typescript
getDisplayContent(image: SpreadViewImage, field: DisplayField): string | null
  IF image.generated_image_url:
    return null  // Will render actual image
  IF field === 'art_note':
    return image.art_note || 'No art note'
  IF field === 'visual_description':
    return image.visual_description || 'No visual description'
```

### 2.4 Render Logic (pseudo)

```
EditableImage:
  displayContent = getDisplayContent(image, displayField)

  handleClick(e):
    e.stopPropagation()
    IF isEditable:
      onSelect()

  RENDER ImageContainer (position: absolute):
    style:
      left: image.geometry.x + '%'
      top: image.geometry.y + '%'
      width: image.geometry.width + '%'
      height: image.geometry.height + '%'
      cursor: isEditable ? 'pointer' : 'default'
      outline: isHovered && !isSelected ? '1px dashed #bdbdbd' : 'none'

    onClick: handleClick
    onMouseEnter: () => setHovered(true)
    onMouseLeave: () => setHovered(false)

    IF image.generated_image_url && !hasError:
      IF isLoading:
        RENDER LoadingSpinner (centered)
      RENDER <img
        src={generated_image_url}
        style: { objectFit: 'contain', width: '100%', height: '100%' }
        loading="lazy"
        onLoad: () => setIsLoading(false)
        onError: () => { setHasError(true); setIsLoading(false) }
      />
    ELSE:
      RENDER ImagePlaceholder với:
        style:
          border: '2px dashed #e0e0e0'
          background: '#f5f5f5'
          display: 'flex', flexDirection: 'column'
          alignItems: 'center', justifyContent: 'center'
          padding: '8px'

        RENDER Icon (image placeholder)
        RENDER Text với displayContent (truncated, italic)
```

### 2.5 Visual

**No Generated Image (placeholder):**

```
┌─────────────────────────────────┐
│  ┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐  │
│  ┆       🖼️ Image            ┆  │
│  ┆  ─────────────────────    ┆  │
│  ┆  "A fluffy orange cat     ┆  │
│  ┆   sitting on a windowsill ┆  │
│  ┆   looking outside..."     ┆  │
│  └┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘  │
│     dashed border, gray bg      │
└─────────────────────────────────┘
```

**With Generated Image:**

```
┌─────────────────────────────────┐
│  ┌───────────────────────────┐  │
│  │                           │  │
│  │    [Actual Image]         │  │
│  │                           │  │
│  │                           │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**Hovered (not selected):**

```
┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐
┆  ┌───────────────────────────┐  ┆
┆  │       [Content]           │  ┆
┆  └───────────────────────────┘  ┆
└┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
   dashed outline (hover hint)
```

**Selected:**

```
╔═══════════════════════════════════╗
║  ┌───────────────────────────┐    ║
║  │       [Content]           │    ║
║  └───────────────────────────┘    ║
╚═══════════════════════════════════╝
   SelectionFrame rendered by parent
```

**Disabled (isEditable = false):**

```
┌─────────────────────────────────┐
│  ┌───────────────────────────┐  │
│  │       [Content]           │  │
│  │      (no interaction)     │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
   cursor: default, no hover effect
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Display Priority | generated_image > displayField | Show actual image when available |
| Placeholder Style | Dashed border, gray bg | Clear visual distinction |
| Text Truncation | Ellipsis after 3 lines | Prevent overflow |
| Drag/Resize | Delegated to SelectionFrame | Single interaction handler |

### 3.2 Image Loading

```typescript
// Initial state when generated_image_url exists
isLoading = true, hasError = false

// On successful load
onLoad: setIsLoading(false)

// On error → fallback to placeholder
onError: setHasError(true), setIsLoading(false)

// Reset states when generated_image_url changes
useEffect(() => {
  IF image.generated_image_url:
    setIsLoading(true)
    setHasError(false)
}, [image.generated_image_url])
```

### 3.3 Accessibility

```typescript
// Use displayField content for placeholder, "Image" for generated images
ariaLabel = image.generated_image_url && !hasError
  ? `Image ${index + 1}`
  : displayContent

<div
  role="img"
  aria-label={ariaLabel}
  tabIndex={isEditable ? 0 : -1}
  onKeyDown={(e) => e.key === 'Enter' && onSelect()}
>
```

### 3.4 Constants

```typescript
const PLACEHOLDER_BG = '#f5f5f5';
const PLACEHOLDER_BORDER = '#e0e0e0';
const HOVER_OUTLINE = '#bdbdbd';
const MAX_TEXT_LINES = 3;
```

---
