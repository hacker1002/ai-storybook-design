# SpreadEditorPanel: Component Design

> **Note:** Replaces `SpreadEditModal`. Inline editor panel thay vì modal, hiển thị khi có spread được select.
>
> **Merged:** SpreadCanvas đã được merge vào component này để đơn giản hóa architecture.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                         Canvas Container                                  │  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         LeftPage            │         RightPage           │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │  EditableImage  │      │      │ EditableTextbox │    │            │  │
│  │  │    │  ┌───────────┐  │      │      │  ┌───────────┐  │    │            │  │
│  │  │    │  │  content  │  │      │      │  │   text    │  │    │            │  │
│  │  │    │  └───────────┘  │      │      │  └───────────┘  │    │            │  │
│  │  │    └─────────────────┘      │      └─────────────────┘    │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  │                                                                           │  │
│  │  ╔═══════════════════════════════════════════════════════════════════╗    │  │
│  │  ║                    SelectionFrame (overlay)                       ║    │  │
│  │  ║  ●────────●────────●  (when element selected)                     ║    │  │
│  │  ╚═══════════════════════════════════════════════════════════════════╝    │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  Local State:                                                              │ │
│  │  • selectedElement: SelectedElement | null                                 │ │
│  │  • isDragging, isResizing: boolean                                         │ │
│  │  • activeHandle: ResizeHandle | null                                       │ │
│  │  • dragStartPos: Point                                                     │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                  │
│         ┌────────────────────┼────────────────────┐                             │
│         ▼                    ▼                    ▼                             │
│  ┌─────────────┐     ┌─────────────────┐  ┌─────────────────┐                   │
│  │EditableImage│     │EditableTextbox  │  │ SelectionFrame  │                   │
│  │ Props:      │     │ Props:          │  │ Props:          │                   │
│  │ • image     │     │ • textbox       │  │ • geometry      │                   │
│  │ • index     │     │ • content       │  │ • zoomLevel     │                   │
│  │ • isSelected│     │ • isSelected    │  │ • showHandles   │                   │
│  │             │     │                 │  │ • activeHandle  │                   │
│  │ Callbacks:  │     │ Callbacks:      │  │                 │                   │
│  │ • onSelect  │     │ • onSelect      │  │ Callbacks:      │                   │
│  │             │     │ • onTextChange  │  │ • onDragStart   │                   │
│  └─────────────┘     └─────────────────┘  │ • onDrag/End    │                   │
│                                           │ • onResizeStart │                   │
│                                           │ • onResize/End  │                   │
│                                           └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │   SnapshotStore         │
                          │  ┌───────────────────┐  │
                          │  │ useSpreadById     │  │
                          │  │ useDummySpreadById│  │
                          │  │ useSnapshotActions│  │
                          │  └───────────────────┘  │
                          └─────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Inline editor panel cho selected spread. Cho phép select, drag, resize images/textboxes và edit textbox content inline.

**Shared Types:**

```typescript
type SpreadViewMode = 'dummy' | 'finalize';
type DummyType = 'prose' | 'poetry';
type SelectedElementType = 'image' | 'textbox';
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

interface SpreadViewImage {
  geometry: Geometry;
  art_note?: string;
  visual_description?: string;
  generated_image_url?: string;
}

interface SpreadViewTextbox {
  [langCode: string]: TextboxContent;  // Keyed by language code
}

interface TextboxContent {
  text: string;
  geometry: Geometry;
  typography?: Typography;
}
```

### 2.2 Interface

**Props:**

```typescript
interface SpreadEditorPanelProps {
  spreadId: string;
  mode: SpreadViewMode;
  dummyType?: DummyType;           // Required when mode === 'dummy'
  zoomLevel: number;               // 50-200
  isEditable: boolean;
  displayField: 'art_note' | 'visual_description';
  // currentLanguage via useCurrentLanguage() - no prop drilling
}
```

**Local State:**

```typescript
interface SpreadEditorPanelState {
  // Selection
  selectedElement: SelectedElement | null;
  isTextboxEditing: boolean;         // Hide SelectionFrame handles when true

  // Drag/Resize (managed here, passed to SelectionFrame)
  isDragging: boolean;
  isResizing: boolean;
  activeHandle: ResizeHandle | null;
  dragStartPos: Point;
  originalGeometry: Geometry | null;
}
```

**Store Integration:**

```typescript
// EditorSettingsStore (global UI state)
currentLanguage = useCurrentLanguage();  // ⚡ no prop drilling
langCode = currentLanguage.code;

// SnapshotStore Selectors (mode-based) - SINGLE source of data
spread = mode === 'dummy'
  ? useDummySpreadById(dummyType, spreadId)
  : useSpreadById(spreadId);

// SnapshotStore Actions
const {
  updateSpreadImage,
  updateSpreadTextbox,
  updateDummySpread,
} = useSnapshotActions();
```

### 2.3 Coordinate System

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

### 2.4 Geometry Derivation

```typescript
// Get geometry of selected element
selectedGeometry = useMemo(() => {
  if (!selectedElement || !spread) return null;

  if (selectedElement.type === 'image') {
    return spread.images[selectedElement.index]?.geometry;
  }

  if (selectedElement.type === 'textbox') {
    const textbox = spread.textboxes[selectedElement.index];
    return textbox?.[langCode]?.geometry;
  }

  return null;
}, [selectedElement, spread, langCode]);
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

// Store update
updateElementGeometry(newGeometry: Geometry):
  IF selectedElement.type === 'image':
    IF mode === 'dummy':
      updateDummySpread(dummyType, spreadId, { images: updatedImages })
    ELSE:
      updateSpreadImage(spreadId, selectedElement.index, { geometry: newGeometry })

  IF selectedElement.type === 'textbox':
    IF mode === 'dummy':
      updateDummySpread(dummyType, spreadId, { textboxes: updatedTextboxes })
    ELSE:
      updateSpreadTextbox(spreadId, selectedElement.index, { [langCode]: { geometry: newGeometry } })

// Text change (called by EditableTextbox)
handleTextChange(textboxIndex: number, newText: string):
  IF mode === 'dummy':
    updateDummySpread(dummyType, spreadId, { textboxes: updatedTextboxes })
  ELSE:
    updateSpreadTextbox(spreadId, textboxIndex, { [langCode]: { text: newText } })

// Editing state change (called by EditableTextbox)
handleEditingChange(isEditing: boolean):
  setIsTextboxEditing(isEditing)
```

### 2.6 Render Logic (pseudo)

```
SpreadEditorPanel:
  // Store data (SINGLE source)
  spread = useSpreadById/useDummySpreadById based on mode
  langCode = useCurrentLanguage().code
  canvasRef = useRef()

  // Canvas sizing
  scaledWidth = BASE_WIDTH * (zoomLevel / 100)
  scaledHeight = BASE_HEIGHT * (zoomLevel / 100)

  // Derive selected geometry
  selectedGeometry = getSelectedGeometry(selectedElement, spread, langCode)

  RENDER OuterContainer (flex, center, overflow-auto):
    RENDER CanvasContainer (position: relative):
      style: { width: scaledWidth, height: scaledHeight }
      onClick: handleCanvasClick
      ref: canvasRef

      // Page divider
      RENDER Divider at x=50%

      // Page numbers
      RENDER PageNumber left: spread.leftPageNumber
      RENDER PageNumber right: spread.rightPageNumber

      // Images (selection only, no drag/resize callbacks)
      FOR EACH (image, index) IN spread.images:
        RENDER EditableImage với:
          - image, index
          - isSelected: selectedElement?.type === 'image' && selectedElement?.index === index
          - displayField, isEditable
          - onSelect: () => handleElementSelect({ type: 'image', index })

      // Textboxes (selection + text change, no drag/resize)
      FOR EACH (textbox, index) IN spread.textboxes:
        content = textbox[langCode]
        RENDER EditableTextbox với:
          - textbox, content, index
          - isSelected: selectedElement?.type === 'textbox' && selectedElement?.index === index
          - isEditable
          - onSelect: () => handleElementSelect({ type: 'textbox', index })
          - onTextChange: (text) => handleTextChange(index, text)
          - onEditingChange: handleEditingChange

      // SelectionFrame overlay (handles ALL drag/resize)
      // Hide handles when dragging OR when textbox is in editing mode
      IF selectedElement && selectedGeometry && isEditable:
        RENDER SelectionFrame với:
          - geometry: selectedGeometry
          - zoomLevel
          - showHandles: !isDragging && !isTextboxEditing
          - activeHandle
          - onDragStart, onDrag, onDragEnd
          - onResizeStart, onResize, onResizeEnd
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
│  │  │    │   ┌─────────┐   │      │      │  Once upon a    │    │            │  │
│  │  │    │   │"A cat   │   │      │      │  time, there    │    │            │  │
│  │  │    │   │sitting..│   │      │      │  was a brave... │    │            │  │
│  │  │    │   └─────────┘   │      │      └─────────────────┘    │            │  │
│  │  │    └─────────────────┘      │                             │            │  │
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
│  │    ●│                 │●    │      │  Once upon a    │    │                  │
│  │    ║│   ┌─────────┐   │║    │      │  time...        │    │                  │
│  │    ║│   │"A cat   │   │║    │      └─────────────────┘    │                  │
│  │    ●│   │sitting..│   │●    │                             │                  │
│  │    ║│   └─────────┘   │║    │                             │                  │
│  │    ║●────────●────────●║    │                             │                  │
│  │    ╚═══════════════════╝    │             3               │                  │
│  └─────────────────────────────┴─────────────────────────────┘                  │
│  ● = resize handles (via SelectionFrame) | Cursor: move/resize                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Textbox Selected (inline editing):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────┬─────────────────────────────┐                  │
│  │         Left Page           │         Right Page          │                  │
│  │    ┌─────────────────┐      │    ╔═══════════════════╗    │                  │
│  │    │     Image       │      │    ║●────────●────────●║    │                  │
│  │    │   ┌─────────┐   │      │    ●│                 │●    │                  │
│  │    │   │"A cat   │   │      │    ║│  Once upon a    │║    │                  │
│  │    │   │sitting..│   │      │    ║│  time, there█   │║← cursor               │
│  │    │   └─────────┘   │      │    ●│  was a brave... │●    │                  │
│  │    └─────────────────┘      │    ║●────────●────────●║    │                  │
│  │           2                 │    ╚═══════════════════╝    │                  │
│  └─────────────────────────────┴─────────────────────────────┘                  │
│  Double-click to edit text inline                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Note:** Drag/resize được handle bởi SelectionFrame, không phải EditableImage/EditableTextbox.

### 3.1 EditableImage

📄 **Doc:** [`component/editor-page/04-02-02-01-editable-image.md`](component/editor-page/04-02-02-01-editable-image.md)

**Mục đích:** Image placeholder trong canvas. Selection only, không handle drag/resize.

**Props & Callbacks:**

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

---

### 3.2 EditableTextbox

📄 **Doc:** [`component/editor-page/04-02-02-02-editable-textbox.md`](component/editor-page/04-02-02-02-editable-textbox.md)

**Mục đích:** Editable textbox trong canvas. Selection và text editing only.

**Special Impact:** ✅ **BỊ ẢNH HƯỞNG** — Content theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface EditableTextboxProps {
  textbox: SpreadViewTextbox;
  content: TextboxContent;           // Pre-extracted for current language
  index: number;
  isSelected: boolean;
  isEditable: boolean;

  onSelect: () => void;
  onTextChange: (text: string) => void;
  onEditingChange: (isEditing: boolean) => void;  // Notify parent to hide handles
  // NO onDrag/onResize - handled by SelectionFrame
}
```

---

### 3.3 SelectionFrame

📄 **Doc:** [`component/editor-page/04-02-02-03-selection-frame.md`](component/editor-page/04-02-02-03-selection-frame.md)

**Mục đích:** Visual selection overlay với 8 resize handles. Handles ALL drag/resize interactions.

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

**Visual:**

```
╔═══●═══╤═══●═══╗
●               ●
╟───────┼───────╢
●               ●
╚═══●═══╧═══●═══╝
● = resize handles (8 total)
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Inline vs Modal | Inline panel | Context preserved, better continuous editing UX |
| Merged Canvas | Single component | Simpler architecture, single store subscription |
| Zoom | CSS dimensions | Simple, performant, maintains vector quality |
| Coordinate System | Percentages (0-100) | Responsive, independent of canvas size |
| Store Access | ID-based selector | Only re-renders when THIS spread changes |
| Drag/Resize | SelectionFrame only | Single interaction handler, cleaner separation |

### 4.2 Mode-based Store Access

| Mode | Selector | Update Action |
|------|----------|---------------|
| `dummy` | `useDummySpreadById(type, id)` | `updateDummySpread(type, id, partial)` |
| `finalize` | `useSpreadById(id)` | `updateSpreadImage()`, `updateSpreadTextbox()` |

### 4.3 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Escape` | Deselect element |
| `Delete` | Delete selected (with confirmation) |
| `Arrow keys` | Nudge selected by 1% |
| `Shift + Arrow` | Nudge by 5% |
| `Enter` | Edit textbox (delegated to EditableTextbox when focused) |

### 4.4 Canvas Constants

```typescript
const BASE_WIDTH = 800;
const BASE_HEIGHT = 600;
const ASPECT_RATIO = 4/3;
const MIN_ELEMENT_SIZE = 5;  // percentage
const NUDGE_STEP = 1;        // percentage
const NUDGE_STEP_SHIFT = 5;  // percentage
```

### 4.5 Performance

- Single store subscription (no redundant fetches)
- Memoize coordinate transformations
- `will-change: transform` for smooth zoom
- Event delegation for canvas clicks

### 4.6 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Canvas | `application` | `aria-label="Spread editor"` |
| Image | `img` | `aria-label={displayField content}` |
| Textbox | `textbox` | `aria-label="Text content"` |
| SelectionFrame | `group` | `aria-label="Selection controls"` |

---
