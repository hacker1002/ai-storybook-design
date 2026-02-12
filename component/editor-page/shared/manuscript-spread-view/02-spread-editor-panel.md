# SpreadEditorPanel: Component Design

> **Note:** Props-driven editor panel với render props. Receives spread data và render functions from parent.
>
> **Merged:** SpreadCanvas đã được merge vào component này.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         SpreadEditorPanel<TSpread>                              │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                         Canvas Container                                  │  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         LeftPage            │         RightPage           │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │ renderImageItem │      │      │ renderTextItem  │    │            │  │
│  │  │    │  (consumer's)   │      │      │  (consumer's)   │    │            │  │
│  │  │    │                 │      │      │                 │    │            │  │
│  │  │    └─────────────────┘      │      └─────────────────┘    │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  │                                                                           │  │
│  │  ╔═══════════════════════════════════════════════════════════════════╗    │  │
│  │  ║                    SelectionFrame (overlay)                       ║    │  │
│  │  ║  ●────────●────────●  (when element selected)                     ║    │  │
│  │  ╚═══════════════════════════════════════════════════════════════════╝    │  │
│  │                                                                           │  │
│  │  ┌───────────────────────────────────────────────────────────────────┐    │  │
│  │  │  Toolbar (if selected & render*Toolbar provided)                  │    │  │
│  │  └───────────────────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
ManuscriptSpreadView<TSpread>
       │
       │ passes spread, renderItems, render*Item, render*Toolbar, callbacks
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  SpreadEditorPanel<TSpread>                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Props:                                                    │  │
│  │  • spread: TSpread (data from parent)                      │  │
│  │  • renderImageItem, renderTextItem, etc.                   │  │
│  │  • renderImageToolbar, renderTextToolbar, etc.             │  │
│  │  • onUpdateImage, onUpdateTextbox, etc.                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  For each item in renderItems:                             │  │
│  │                                                            │  │
│  │  1. Build context with item data + callbacks               │  │
│  │  2. Call render*Item(context)                              │  │
│  │  3. If selected + toolbar exists, call render*Toolbar      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐              │
│         ▼                    ▼                    ▼              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐          │
│  │ Consumer's   │   │ Consumer's   │   │ Selection    │          │
│  │ ImageItem    │   │ TextItem     │   │ Frame        │          │
│  │ Component    │   │ Component    │   │ (built-in)   │          │
│  └──────────────┘   └──────────────┘   └──────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Inline editor panel cho selected spread. Cho phép select, drag, resize items via SelectionFrame. Render items via render props from parent.

**Shared Types:**

```typescript
type ItemType = 'image' | 'text' | 'object' | 'animation';
type SelectedElementType = 'image' | 'textbox' | 'object' | 'animation';
type ResizeHandle = 'n' | 's' | 'e' | 'w' | 'nw' | 'ne' | 'sw' | 'se';

interface SelectedElement {
  type: SelectedElementType;
  index: number;
}

interface Point {
  x: number;
  y: number;
}

interface Geometry {
  x: number;      // percentage 0-100
  y: number;      // percentage 0-100
  width: number;  // percentage 0-100
  height: number; // percentage 0-100
}
```

### 2.2 Interface

**Props:**

```typescript
interface SpreadEditorPanelProps<TSpread extends BaseSpread> {
  // === Data (from parent) ===
  spread: TSpread;
  spreadIndex: number;

  // === View config ===
  zoomLevel: number;               // 50-200
  isEditable: boolean;

  // === Render configuration ===
  renderItems: ItemType[];         // ['image', 'text', 'object', 'animation']

  // === Item render functions ===
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // === Toolbar render functions (optional) ===
  renderImageToolbar?: (context: ImageToolbarContext<TSpread>) => ReactNode;
  renderTextToolbar?: (context: TextToolbarContext<TSpread>) => ReactNode;
  renderObjectToolbar?: (context: ObjectToolbarContext<TSpread>) => ReactNode;
  renderAnimationToolbar?: (context: AnimationToolbarContext<TSpread>) => ReactNode;

  // === Callbacks ===
  onUpdateSpread: (updates: Partial<TSpread>) => void;
  onUpdateImage: (imageIndex: number, updates: Partial<SpreadImage>) => void;
  onUpdateTextbox: (textboxIndex: number, updates: Partial<SpreadTextbox>) => void;
  onUpdateObject?: (objectIndex: number, updates: Partial<SpreadObject>) => void;
  onUpdateAnimation?: (animIndex: number, updates: Partial<SpreadAnimation>) => void;
  onDeleteImage?: (imageIndex: number) => void;
  onDeleteTextbox?: (textboxIndex: number) => void;
}
```

**Local State:**

```typescript
interface SpreadEditorPanelState {
  // Selection
  selectedElement: SelectedElement | null;
  isTextboxEditing: boolean;         // Hide SelectionFrame handles when true

  // Drag/Resize
  isDragging: boolean;
  isResizing: boolean;
  activeHandle: ResizeHandle | null;
  dragStartPos: Point;
  originalGeometry: Geometry | null;
}
```

### 2.3 Context Building

Panel builds context objects for render props:

```typescript
buildImageContext(image: SpreadImage, index: number): ImageItemContext<TSpread>
  return {
    item: image,
    itemIndex: index,
    spreadId: spread.id,
    spread: spread,
    isSelected: selectedElement?.type === 'image' && selectedElement.index === index,
    isSpreadSelected: true,
    onSelect: () => handleElementSelect({ type: 'image', index }),
    onUpdate: (updates) => onUpdateImage(index, updates),
    onDelete: () => onDeleteImage?.(index),
  }

buildTextContext(textbox: SpreadTextbox, index: number): TextItemContext<TSpread>
  return {
    item: textbox,
    itemIndex: index,
    spreadId: spread.id,
    spread: spread,
    isSelected: selectedElement?.type === 'textbox' && selectedElement.index === index,
    isSpreadSelected: true,
    onSelect: () => handleElementSelect({ type: 'textbox', index }),
    onTextChange: (text) => onUpdateTextbox(index, { text }),
    onUpdate: (updates) => onUpdateTextbox(index, updates),
    onDelete: () => onDeleteTextbox?.(index),
  }

// Toolbar contexts extend item contexts
buildImageToolbarContext(image, index): ImageToolbarContext<TSpread>
  return {
    ...buildImageContext(image, index),
    onGenerateImage: () => { /* consumer implements via callback */ },
    onReplaceImage: () => { /* consumer implements via callback */ },
  }
```

### 2.4 Coordinate System

```typescript
// Canvas Constants
const BASE_WIDTH = 800;
const BASE_HEIGHT = 600;
const ASPECT_RATIO = 4/3;

// Percentage → Pixel (for rendering)
toPixel(percent: number, dimension: number): number
  return (percent / 100) * dimension

// Pixel → Percentage (for storage)
toPercent(pixel: number, dimension: number): number
  return (pixel / dimension) * 100

// Mouse event → Canvas percentage (accounts for zoom)
mouseToCanvasPercent(event, canvasRect, zoomLevel): Point
  x = ((event.clientX - canvasRect.left) / (zoomLevel / 100)) / canvasRect.width * 100
  y = ((event.clientY - canvasRect.top) / (zoomLevel / 100)) / canvasRect.height * 100
  return { x, y }

// Page detection
isOnLeftPage(geometry: Geometry): boolean
  return geometry.x + geometry.width / 2 < 50

isOnRightPage(geometry: Geometry): boolean
  return geometry.x + geometry.width / 2 >= 50
```

### 2.5 Handler Mappings

```typescript
// Selection
handleElementSelect(element: SelectedElement | null):
  setSelectedElement(element)
  resetDragState()

handleCanvasClick(e):
  IF e.target === canvasRef.current:
    handleElementSelect(null)  // Deselect

// Drag handlers (called by SelectionFrame)
handleDragStart():
  setIsDragging(true)
  setDragStartPos(currentMousePos)
  setOriginalGeometry(selectedGeometry)

handleDrag(delta: Point):
  IF !isDragging || !originalGeometry RETURN

  newGeometry = {
    ...originalGeometry,
    x: clamp(originalGeometry.x + delta.x, 0, 100 - originalGeometry.width),
    y: clamp(originalGeometry.y + delta.y, 0, 100 - originalGeometry.height),
  }
  updateElementGeometry(newGeometry)

handleDragEnd():
  setIsDragging(false)

// Resize handlers (called by SelectionFrame)
handleResizeStart(handle: ResizeHandle):
  setIsResizing(true)
  setActiveHandle(handle)
  setDragStartPos(currentMousePos)
  setOriginalGeometry(selectedGeometry)

handleResize(handle: ResizeHandle, delta: Point):
  IF !isResizing || !originalGeometry RETURN

  newGeometry = calculateResizedGeometry(originalGeometry, handle, delta)
  updateElementGeometry(newGeometry)

handleResizeEnd():
  setIsResizing(false)
  setActiveHandle(null)

// Store update via callbacks
updateElementGeometry(newGeometry: Geometry):
  IF selectedElement.type === 'image':
    onUpdateImage(selectedElement.index, { geometry: newGeometry })

  IF selectedElement.type === 'textbox':
    onUpdateTextbox(selectedElement.index, { geometry: newGeometry })

// Editing state change (called by consumer's component)
handleEditingChange(isEditing: boolean):
  setIsTextboxEditing(isEditing)
```

### 2.6 Render Logic (pseudo)

```
SpreadEditorPanel<TSpread>:
  // Data from props
  spread = props.spread
  canvasRef = useRef()

  // Canvas sizing
  scaledWidth = BASE_WIDTH * (zoomLevel / 100)
  scaledHeight = BASE_HEIGHT * (zoomLevel / 100)

  // Derive selected geometry from local selection state
  selectedGeometry = getSelectedGeometry(selectedElement, spread)

  RENDER OuterContainer (flex, center, overflow-auto):
    RENDER CanvasContainer onClick={handleCanvasClick} ref={canvasRef}:
      style: { width: scaledWidth, height: scaledHeight }

      // Page divider and numbers
      RENDER Divider at x=50%
      RENDER PageNumber left: spread.left_page.number
      RENDER PageNumber right: spread.right_page.number

      // Render items based on renderItems config
      IF renderItems.includes('image'):
        FOR EACH (image, index) IN spread.images:
          context = buildImageContext(image, index)
          RENDER props.renderImageItem(context)

      IF renderItems.includes('text'):
        FOR EACH (textbox, index) IN spread.textboxes:
          context = buildTextContext(textbox, index)
          RENDER props.renderTextItem(context)

      IF renderItems.includes('object') AND spread.objects:
        FOR EACH (obj, index) IN spread.objects:
          context = buildObjectContext(obj, index)
          RENDER props.renderObjectItem?.(context) ?? null

      IF renderItems.includes('animation') AND spread.animations:
        FOR EACH (anim, index) IN spread.animations:
          context = buildAnimationContext(anim, index)
          RENDER props.renderAnimationItem?.(context) ?? null

      // SelectionFrame (built-in, always available)
      IF selectedElement && selectedGeometry && isEditable:
        RENDER SelectionFrame với:
          geometry: selectedGeometry
          zoomLevel
          showHandles: !isDragging && !isTextboxEditing
          onDragStart, onDrag, onDragEnd
          onResizeStart, onResize, onResizeEnd

      // Toolbar (render if item selected and toolbar render function exists)
      IF selectedElement && isEditable:
        toolbarContext = buildToolbarContext(selectedElement)
        toolbarRenderer = getToolbarRenderer(selectedElement.type)
        IF toolbarRenderer:
          RENDER toolbarRenderer(toolbarContext)
```

### 2.7 Visual States

**Normal State (nothing selected):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │     Image       │      │      │     Textbox     │    │            │  │
│  │  │    │   (rendered by  │      │      │  (rendered by   │    │            │  │
│  │  │    │   consumer)     │      │      │   consumer)     │    │            │  │
│  │  │    │                 │      │      │                 │    │            │  │
│  │  │    └─────────────────┘      │      └─────────────────┘    │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│  Cursor: default | Click element to select                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Image Selected (SelectionFrame visible):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────┬─────────────────────────────┐                  │
│  │         Left Page           │         Right Page          │                  │
│  │    ╔═══════════════════╗    │      ┌─────────────────┐    │                  │
│  │    ║●────────●────────●║    │      │     Textbox     │    │                  │
│  │    ●│                 │●    │      │                 │    │                  │
│  │    ║│   (consumer's   │║    │      │                 │    │                  │
│  │    ║│    component)   │║    │      │                 │    │                  │
│  │    ●│                 │●    │      │                 │    │                  │
│  │    ║●────────●────────●║    │                             │                  │
│  │    ╚═══════════════════╝    │             3               │                  │
│  └─────────────────────────────┴─────────────────────────────┘                  │
│  ● = resize handles (via SelectionFrame) | Cursor: move/resize                  │
│  [Toolbar appears if renderImageToolbar provided]                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Note:** Drag/resize được handle bởi SelectionFrame, không phải consumer's render components.

### 3.1 SelectionFrame

📄 **Doc:** [02-03-selection-frame.md](./02-03-selection-frame.md)

**Mục đích:** Visual selection overlay với 8 resize handles. Internal component, not customizable.

**Props & Callbacks:**

```typescript
interface SelectionFrameProps {
  geometry: Geometry;
  zoomLevel: number;               // For accurate delta calculation
  showHandles: boolean;            // false during drag
  activeHandle: ResizeHandle | null;

  onDragStart: () => void;
  onDrag: (delta: Point) => void;
  onDragEnd: () => void;
  onResizeStart: (handle: ResizeHandle) => void;
  onResize: (handle: ResizeHandle, delta: Point) => void;
  onResizeEnd: () => void;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Props-driven | Spread via props | No store coupling |
| Render props | Consumer provides | Full customization |
| SelectionFrame | Built-in | Consistent drag/resize |
| Toolbar optional | Null if missing | Progressive enhancement |
| Context objects | Full data + callbacks | Extensible |

### 4.2 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Escape` | Deselect element |
| `Delete` | Delete selected (calls onDelete callback) |
| `Arrow keys` | Nudge selected by 1% |
| `Shift + Arrow` | Nudge by 5% |

### 4.3 Canvas Constants

```typescript
const BASE_WIDTH = 800;
const BASE_HEIGHT = 600;
const ASPECT_RATIO = 4/3;
const MIN_ELEMENT_SIZE = 5;  // percentage
const NUDGE_STEP = 1;        // percentage
const NUDGE_STEP_SHIFT = 5;  // percentage
```

### 4.4 Performance

- Memoize context builders with `useMemo`
- `will-change: transform` for smooth zoom
- Event delegation for canvas clicks
- Render only items in renderItems array

### 4.5 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Canvas | `application` | `aria-label="Spread editor"` |
| SelectionFrame | `group` | `aria-label="Selection controls"` |

---
