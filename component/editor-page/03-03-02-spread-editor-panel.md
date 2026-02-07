# SpreadEditorPanel: Component Design

> **Note:** Replaces `SpreadEditModal`. Inline editor panel thay vì modal, hiển thị khi có spread được select.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                            SpreadCanvas                                   │  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         LeftPage            │         RightPage           │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │  EditableImage  │      │      │ EditableTextbox │    │            │  │
│  │  │    │  ╔═══════════╗  │      │      │  ┌───────────┐  │    │            │  │
│  │  │    │  ║ ┌───────┐ ║  │      │      │  │ Text      │  │    │            │  │
│  │  │    │  ║ │content│ ║  │      │      │  │ content   │  │    │            │  │
│  │  │    │  ║ └───────┘ ║  │      │      │  └───────────┘  │    │            │  │
│  │  │    │  ╚═══════════╝  │      │      └─────────────────┘    │            │  │
│  │  │    │    ↑ selected   │      │                             │            │  │
│  │  │    └─────────────────┘      │                             │            │  │
│  │  │                             │                             │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  Local State: selectedElement, isDragging, isResizing, dragOffset          │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                              │                                                  │
│                              ▼                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                           SpreadCanvas                                     │ │
│  │                                │                                           │ │
│  │     ┌──────────────────────────┼──────────────────────────────┐            │ │
│  │     ▼                          ▼                              ▼            │ │
│  │  ┌────────────┐          ┌────────────┐          ┌────────────────┐        │ │
│  │  │ LeftPage   │          │ RightPage  │          │ SelectionFrame │        │ │
│  │  │ Props:     │          │ Props:     │          │ (when selected)│        │ │
│  │  │ • images   │          │ • images   │          │ Props:         │        │ │
│  │  │ • textboxes│          │ • textboxes│          │ • geometry     │        │ │
│  │  │ Callbacks: │          │ Callbacks: │          │ • showHandles  │        │ │
│  │  │ • onSelect │          │ • onSelect │          │ Callbacks:     │        │ │
│  │  │ • onDrag   │          │ • onDrag   │          │ • onDrag       │        │ │
│  │  │ • onResize │          │ • onResize │          │ • onResize     │        │ │
│  │  └────────────┘          └────────────┘          └────────────────┘        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
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
type SelectedElementType = 'image' | 'textbox' | null;
type ResizeHandle = 'n' | 's' | 'e' | 'w' | 'nw' | 'ne' | 'sw' | 'se';

interface SelectedElement {
  type: SelectedElementType;
  index: number;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SpreadEditorPanelProps {
  spreadId: string;
  mode: SpreadViewMode;
  dummyType?: DummyType;           // Required when mode === 'dummy'
  // currentLanguage via useCurrentLanguage() - no prop drilling
  zoomLevel: number;               // 50-200
  isEditable: boolean;
  displayField: 'art_note' | 'visual_description';
}

interface SpreadEditorPanelState {
  selectedElement: SelectedElement | null;
  isDragging: boolean;
  isResizing: boolean;
  dragOffset: { x: number; y: number };
  resizeHandle: ResizeHandle | null;
}
```

**Store Integration:**

```typescript
// EditorSettingsStore (global UI state)
currentLanguage = useCurrentLanguage();  // ⚡ no prop drilling
langCode = currentLanguage.code;

// SnapshotStore Selectors (mode-based)
spread = mode === 'dummy'
  ? useDummySpreadById(dummyType, spreadId)
  : useSpreadById(spreadId);

// SnapshotStore Actions
const {
  updateSpreadImage,
  updateSpreadTextbox,
  updateDummySpread,
} = useSnapshotActions();

// Handler mappings
handleImageGeometryChange(imageIndex, newGeometry):
  IF mode === 'dummy':
    updateDummySpread(dummyType, spreadId, { images: updatedImages })
  ELSE:
    updateSpreadImage(spreadId, imageIndex, { geometry: newGeometry })

handleTextboxGeometryChange(textboxId, newGeometry):
  IF mode === 'dummy':
    updateDummySpread(dummyType, spreadId, { textboxes: updatedTextboxes })
  ELSE:
    updateSpreadTextbox(spreadId, textboxId, { [langCode]: { geometry: newGeometry } })

handleTextboxTextChange(textboxId, newText):
  IF mode === 'dummy':
    updateDummySpread(dummyType, spreadId, { textboxes: updatedTextboxes })
  ELSE:
    updateSpreadTextbox(spreadId, textboxId, { [langCode]: { text: newText } })
```

### 2.3 Render Logic (pseudo)

```
SpreadEditorPanel:
  spread = useSpreadById/useDummySpreadById based on mode
  scaledSize = calculateScaledSize(containerSize, zoomLevel)

  RENDER Container (flex, center, overflow-auto):
    RENDER SpreadCanvas với:
      - style: { width: scaledSize.width, height: scaledSize.height }
      - onClick: handleCanvasClick → deselect

      RENDER SpreadFrame với leftPageNumber, rightPageNumber

      FOR EACH page IN [left, right]:
        FOR EACH image WHERE isOnPage(image, page):
          RENDER EditableImage với index, isSelected, callbacks
        FOR EACH textbox WHERE isOnPage(textbox, page):
          RENDER EditableTextbox với content[langCode], callbacks

      IF selectedElement && isEditable:
        RENDER SelectionFrame với geometry, handles, callbacks
```

### 2.4 Visual

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

**Image Selected:**

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
│  ● = resize handles | Cursor: move (element), resize (handles)                  │
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

### 3.1 SpreadCanvas

📄 **Doc:** [`component/editor-page/03-03-02-01-spread-canvas.md`](component/editor-page/03-03-02-01-spread-canvas.md)

**Mục đích:** Container canvas cho spread content, handles zoom scaling và mouse events.

**Props & Callbacks:**

```typescript
interface SpreadCanvasProps {
  spreadId: string;
  mode: SpreadViewMode;
  dummyType?: DummyType;
  zoomLevel: number;
  // currentLanguage via useCurrentLanguage() - no prop drilling
  isEditable: boolean;
  displayField: 'art_note' | 'visual_description';
  selectedElement: SelectedElement | null;

  onElementSelect: (element: SelectedElement | null) => void;
}
```

---

### 3.2 EditableImage

📄 **Doc:** [`component/editor-page/03-03-02-02-editable-image.md`](component/editor-page/03-03-02-02-editable-image.md)

**Mục đích:** Draggable/resizable image placeholder trong canvas.

**Props & Callbacks:**

```typescript
interface EditableImageProps {
  image: SpreadViewImage;
  index: number;
  isSelected: boolean;
  displayField: 'art_note' | 'visual_description';
  isEditable: boolean;

  onSelect: () => void;
  onDrag: (newGeometry: Geometry) => void;
  onResize: (newGeometry: Geometry) => void;
}
```

---

### 3.3 EditableTextbox

📄 **Doc:** [`component/editor-page/03-03-02-03-editable-textbox.md`](component/editor-page/03-03-02-03-editable-textbox.md)

**Mục đích:** Draggable/resizable/editable textbox trong canvas.

**Special Impact:** ✅ **BỊ ẢNH HƯỞNG** — Content theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface EditableTextboxProps {
  textbox: SpreadViewTextbox;
  content: { text: string; geometry: Geometry; typography: Typography };
  index: number;
  isSelected: boolean;
  isEditable: boolean;

  onSelect: () => void;
  onDrag: (newGeometry: Geometry) => void;
  onResize: (newGeometry: Geometry) => void;
  onTextChange: (text: string) => void;
}
```

---

### 3.4 SelectionFrame

📄 **Doc:** [`component/editor-page/03-03-02-04-selection-frame.md`](component/editor-page/03-03-02-04-selection-frame.md)

**Mục đích:** Visual selection overlay với 8 resize handles.

**Props & Callbacks:**

```typescript
interface SelectionFrameProps {
  geometry: Geometry;
  showHandles: boolean;

  onDragStart: () => void;
  onDrag: (delta: { x: number; y: number }) => void;
  onDragEnd: () => void;
  onResizeStart: (handle: ResizeHandle) => void;
  onResize: (handle: ResizeHandle, delta: { x: number; y: number }) => void;
  onResizeEnd: () => void;
}
```

**Visual:**

```
┌───●───┬───●───┐
│  nw   │   n   │  ne
├───────┼───────┤
●   w   │       ●  e
├───────┼───────┤
│  sw   │   s   │  se
└───●───┴───●───┘
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Inline vs Modal | Inline panel | Context preserved, better continuous editing UX |
| Zoom | CSS transform scale | Simple, performant, maintains vector quality |
| Coordinate System | Percentages (0-100) | Responsive, independent of canvas size |
| Store Access | ID-based selector | Only re-renders when THIS spread changes |
| Mutations | Store actions (not callbacks) | Stable references, no prop drilling |

### 4.2 Mode-based Store Access

| Mode | Selector | Update Action |
|------|----------|---------------|
| `dummy` | `useDummySpreadById(type, id)` | `updateDummySpread(type, id, partial)` |
| `spread` | `useSpreadById(id)` | `updateSpreadImage()`, `updateSpreadTextbox()` |

### 4.3 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Escape` | Deselect element |
| `Delete` | Delete selected (with confirmation) |
| `Arrow keys` | Nudge selected by 1% |
| `Shift + Arrow` | Nudge by 5% |
| `Enter` | Edit textbox (when textbox selected) |

---

