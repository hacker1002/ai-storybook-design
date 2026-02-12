# EditableImage: Utility Component Design

> **Parent:** [SpreadEditorPanel](./02-spread-editor-panel.md)
>
> **Role:** Optional utility component. Consumer can import and use inside `renderImageItem` render prop.

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

### 1.2 Usage Pattern

```typescript
// Consumer uses inside renderImageItem prop
<ManuscriptSpreadView
  spreads={spreads}
  renderItems={['image', 'text']}
  renderImageItem={(context) => (
    <EditableImage
      image={context.item}
      index={context.itemIndex}
      isSelected={context.isSelected}
      isEditable={true}
      onSelect={context.onSelect}
    />
  )}
  // ...
/>
```

---

## 2. Component Design

### 2.1 Overview

**Mục đích:** Default image renderer for ManuscriptSpreadView. Displays image or placeholder with description text.

> **Note:** Drag/resize handled by SelectionFrame, không phải component này.

**Shared Types:**

```typescript
interface SpreadImage {
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
  image: SpreadImage;
  index: number;
  isSelected: boolean;
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

### 2.3 Display Logic

```typescript
getDisplayContent(image: SpreadImage): string | null
  IF image.generated_image_url:
    return null  // Will render actual image

  // Use art_note or visual_description based on availability
  return image.art_note || image.visual_description || 'No description'
```

### 2.4 Render Logic (pseudo)

```
EditableImage:
  displayContent = getDisplayContent(image)

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

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Utility component | Optional import | Consumer can use custom |
| Display Priority | generated_image > description | Show actual image when available |
| Placeholder Style | Dashed border, gray bg | Clear visual distinction |
| Text Truncation | Ellipsis after 3 lines | Prevent overflow |

### 3.2 Image Loading

```typescript
// Reset states when generated_image_url changes
useEffect(() => {
  IF image.generated_image_url:
    setIsLoading(true)
    setHasError(false)
}, [image.generated_image_url])
```

### 3.3 Accessibility

```typescript
<div
  role="img"
  aria-label={displayContent || `Image ${index + 1}`}
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
