# SpreadEditorPanel: Component Design

> **Note:** Replaces `SpreadEditModal`. Inline editor panel thay vì modal, hiển thị khi có spread được select.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                            SpreadCanvas                                    │  │
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
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                    │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: spread, currentLanguage, mode, zoomLevel, isEditable, displayField │  │
│  │  State: selectedElement, isDragging, isResizing                            │  │
│  │  Callbacks: onSpreadUpdate                                                  │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │                           SpreadCanvas                                      │  │
│  │                                │                                            │  │
│  │     ┌──────────────────────────┼──────────────────────────┐                │  │
│  │     ▼                          ▼                          ▼                │  │
│  │  ┌────────────┐          ┌────────────┐          ┌────────────────┐        │  │
│  │  │ LeftPage   │          │ RightPage  │          │ SelectionFrame │        │  │
│  │  │            │          │            │          │ (when selected)│        │  │
│  │  │ • Images   │          │ • Images   │          │                │        │  │
│  │  │ • Textboxes│          │ • Textboxes│          │ • Handles      │        │  │
│  │  └────────────┘          └────────────┘          │ • Controls     │        │  │
│  │                                                  └────────────────┘        │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Inline editor panel cho selected spread, thay thế modal-based editing. Cho phép:
- Select, drag, resize images và textboxes
- Edit textbox content inline
- Visual preview với zoom support

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content theo `currentLanguage.code`

### 2.2 Interface

```typescript
type SelectedElementType = 'image' | 'textbox' | null;

interface SelectedElement {
  type: SelectedElementType;
  index: number;
}

interface SpreadEditorPanelProps {
  spread: SpreadViewSpread;
  spreadIndex: number;
  currentLanguage: Language;
  mode: SpreadViewMode;
  zoomLevel: number;                     // 50-200
  isEditable: boolean;
  displayField: 'art_note' | 'visual_description';

  onSpreadUpdate: (updatedSpread: SpreadViewSpread) => void;
}

interface SpreadEditorPanelState {
  selectedElement: SelectedElement | null;
  isDragging: boolean;
  isResizing: boolean;
  dragOffset: { x: number; y: number };
  resizeHandle: ResizeHandle | null;
}

type ResizeHandle = 'n' | 's' | 'e' | 'w' | 'nw' | 'ne' | 'sw' | 'se';
```

### 2.3 Render Logic (pseudo)

```
SpreadEditorPanel:
  scaledSize = calculateScaledSize(containerSize, zoomLevel)

  RENDER Container (flex, center, overflow-auto):
    RENDER SpreadCanvas với:
      - style: { width: scaledSize.width, height: scaledSize.height }
      - onClick: (e) => handleCanvasClick(e)

      // Background spread frame
      RENDER SpreadFrame với:
        - leftPageNumber: spread.left_page.number
        - rightPageNumber: spread.right_page.number

      // Left Page content
      RENDER LeftPage với:
        FOR EACH image IN spread.images WHERE isOnLeftPage(image):
          RENDER EditableImage với:
            - image
            - index: imageIndex
            - isSelected: selectedElement?.type === 'image' && selectedElement.index === imageIndex
            - displayField
            - isEditable
            - onSelect: () => selectElement('image', imageIndex)
            - onDrag: handleImageDrag
            - onResize: handleImageResize

        FOR EACH textbox IN spread.textboxes WHERE isOnLeftPage(textbox):
          langContent = textbox[currentLanguage.code]
          RENDER EditableTextbox với:
            - textbox
            - content: langContent
            - index: textboxIndex
            - isSelected: selectedElement?.type === 'textbox' && selectedElement.index === textboxIndex
            - isEditable
            - onSelect: () => selectElement('textbox', textboxIndex)
            - onDrag: handleTextboxDrag
            - onResize: handleTextboxResize
            - onTextChange: handleTextChange

      // Right Page content (same pattern)
      RENDER RightPage với: [...]

      // Selection overlay
      IF selectedElement !== null AND isEditable:
        element = getElement(selectedElement)
        RENDER SelectionFrame với:
          - geometry: element.geometry
          - showHandles: true
          - onResizeStart: (handle) => startResize(handle)

  handleCanvasClick(e):
    IF e.target === canvas:
      setSelectedElement(null)  // Deselect when clicking empty area

  selectElement(type, index):
    setSelectedElement({ type, index })

  handleImageDrag(index, newGeometry):
    updatedSpread = updateImageGeometry(spread, index, newGeometry)
    onSpreadUpdate(updatedSpread)

  handleTextboxDrag(index, newGeometry):
    updatedSpread = updateTextboxGeometry(spread, index, currentLanguage.code, newGeometry)
    onSpreadUpdate(updatedSpread)

  handleTextChange(index, newText):
    updatedSpread = updateTextboxText(spread, index, currentLanguage.code, newText)
    onSpreadUpdate(updatedSpread)
```

### 2.4 Visual

**Normal State (nothing selected):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                   │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                            SpreadCanvas                                    │  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │                             │            │  │
│  │  │    │     Image       │      │      ┌─────────────────┐    │            │  │
│  │  │    │                 │      │      │     Textbox     │    │            │  │
│  │  │    │   ┌─────────┐   │      │      │                 │    │            │  │
│  │  │    │   │"A cat   │   │      │      │  Once upon a    │    │            │  │
│  │  │    │   │sitting..│   │      │      │  time, there    │    │            │  │
│  │  │    │   └─────────┘   │      │      │  was a brave... │    │            │  │
│  │  │    │                 │      │      │                 │    │            │  │
│  │  │    └─────────────────┘      │      └─────────────────┘    │            │  │
│  │  │                             │                             │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  Cursor: default                                                                │
│  Click on element to select                                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Image Selected:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │                             │                             │            │  │
│  │  │    ╔═══════════════════╗    │                             │            │  │
│  │  │    ║●────────●────────●║    │      ┌─────────────────┐    │            │  │
│  │  │    ║│                 │║    │      │     Textbox     │    │            │  │
│  │  │    ║│   ┌─────────┐   │║    │      │                 │    │            │  │
│  │  │    ●│   │"A cat   │   │●    │      │  Once upon a    │    │            │  │
│  │  │    ║│   │sitting..│   │║    │      │  time...        │    │            │  │
│  │  │    ║│   └─────────┘   │║    │      │                 │    │            │  │
│  │  │    ║│                 │║    │      └─────────────────┘    │            │  │
│  │  │    ║●────────●────────●║    │                             │            │  │
│  │  │    ╚═══════════════════╝    │                             │            │  │
│  │  │      ↑ selection frame      │                             │            │  │
│  │  │        with handles         │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  Cursor: move (on element), resize (on handles)                                 │
│  Drag to move, drag handles to resize                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Textbox Selected (with inline editing):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │    ╔═══════════════════╗    │            │  │
│  │  │    │     Image       │      │    ║●────────●────────●║    │            │  │
│  │  │    │                 │      │    ║│                 │║    │            │  │
│  │  │    │   ┌─────────┐   │      │    ●│  Once upon a    │●    │            │  │
│  │  │    │   │"A cat   │   │      │    ║│  time, there█   │║← cursor        │  │
│  │  │    │   │sitting..│   │      │    ║│  was a brave... │║    │            │  │
│  │  │    │   └─────────┘   │      │    ║│                 │║    │            │  │
│  │  │    └─────────────────┘      │    ║●────────●────────●║    │            │  │
│  │  │                             │    ╚═══════════════════╝    │            │  │
│  │  │           2                 │      ↑ editable textarea    │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  Double-click to edit text inline                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Zoomed View (150%):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                   │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                         (scrollable container)                             │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    SpreadCanvas (scaled 150%)                        │  │  │
│  │  │  ┌──────────────────────────────┬──────────────────────────────┐    │  │  │
│  │  │  │                              │                              │    │  │  │
│  │  │  │       ┌─────────────────┐    │                              │    │  │  │
│  │  │  │       │                 │    │                              │    │  │  │
│  │  │  │       │                 │    │                              │    │  │  │
│  │  │  │       │   ┌───────────┐ │    │   (right page scrolled out)  │    │  │  │
│  │  │  │       │   │ "A cat    │ │    │                              │    │  │  │
│  │  │  │       │   │ sitting on│ │    │                              │    │  │  │
│  │  │  │       │   │ a cozy    │ │    │                              │    │  │  │
│  │  │  │       │   │ blanket"  │ │    │                              │    │  │  │
│  │  │  │       │   └───────────┘ │    │                              │    │  │  │
│  │  │  │       │                 │    │                              │    │  │  │
│  │  │  │       └─────────────────┘    │                              │    │  │  │
│  │  │  │              2               │              3               │    │  │  │
│  │  │  └──────────────────────────────┴──────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  Scroll to pan when zoomed in                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Read-Only Mode (isEditable: false):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │     Image       │      │      │     Textbox     │    │            │  │
│  │  │    │                 │      │      │                 │    │            │  │
│  │  │    │   ┌─────────┐   │      │      │  Once upon a    │    │            │  │
│  │  │    │   │"visual  │   │      │      │  time...        │    │            │  │
│  │  │    │   │desc"    │   │      │      │                 │    │            │  │
│  │  │    │   └─────────┘   │      │      └─────────────────┘    │            │  │
│  │  │    └─────────────────┘      │                             │            │  │
│  │  │                             │                             │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  Cursor: default (not pointer)                                                  │
│  No selection, no drag, no edit                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

### 3.1 SpreadCanvas

**Mục đích:** Container canvas cho spread content, handles zoom scaling và mouse events.

**Props:**

```typescript
interface SpreadCanvasProps {
  spread: SpreadViewSpread;
  zoomLevel: number;
  currentLanguage: Language;
  isEditable: boolean;
  displayField: 'art_note' | 'visual_description';
  selectedElement: SelectedElement | null;

  onElementSelect: (element: SelectedElement | null) => void;
  onImageUpdate: (index: number, geometry: Geometry) => void;
  onTextboxUpdate: (index: number, geometry: Geometry) => void;
  onTextChange: (index: number, text: string) => void;
}
```

### 3.2 EditableImage

**Mục đích:** Draggable/resizable image placeholder trong canvas.

**Props:**

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

### 3.3 EditableTextbox

**Mục đích:** Draggable/resizable/editable textbox trong canvas.

**Props:**

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

### 3.4 SelectionFrame

**Mục đích:** Visual selection overlay với resize handles.

**Props:**

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

**Inline vs Modal**
Thay modal bằng inline panel. Lý do:
- Context không bị mất khi edit
- Dễ so sánh với filmstrip bên dưới
- UX tốt hơn cho continuous editing

**Zoom Implementation**
Canvas được scale bằng CSS transform. Container có overflow scroll để pan khi zoomed in. Lý do: Simple, performant, maintains vector quality.

**Coordinate System**
All positions/sizes in percentages (0-100). Lý do: Responsive, independent of canvas size.

### 4.2 Geometry Update Helpers

```typescript
function updateImageGeometry(
  spread: SpreadViewSpread,
  index: number,
  newGeometry: Geometry
): SpreadViewSpread {
  return {
    ...spread,
    images: spread.images.map((img, i) =>
      i === index ? { ...img, geometry: newGeometry } : img
    ),
  };
}

function updateTextboxGeometry(
  spread: SpreadViewSpread,
  index: number,
  langCode: string,
  newGeometry: Geometry
): SpreadViewSpread {
  return {
    ...spread,
    textboxes: spread.textboxes.map((tb, i) =>
      i === index
        ? {
            ...tb,
            [langCode]: { ...tb[langCode], geometry: newGeometry },
          }
        : tb
    ),
  };
}

function updateTextboxText(
  spread: SpreadViewSpread,
  index: number,
  langCode: string,
  newText: string
): SpreadViewSpread {
  return {
    ...spread,
    textboxes: spread.textboxes.map((tb, i) =>
      i === index
        ? {
            ...tb,
            [langCode]: { ...tb[langCode], text: newText },
          }
        : tb
    ),
  };
}
```

### 4.3 Drag Constraints

```typescript
function constrainToPage(
  geometry: Geometry,
  page: 'left' | 'right'
): Geometry {
  const minX = page === 'left' ? 0 : 50;
  const maxX = page === 'left' ? 50 : 100;

  return {
    x: Math.max(minX, Math.min(maxX - geometry.w, geometry.x)),
    y: Math.max(0, Math.min(100 - geometry.h, geometry.y)),
    w: geometry.w,
    h: geometry.h,
  };
}

function isOnLeftPage(geometry: Geometry): boolean {
  return geometry.x + geometry.w / 2 < 50;
}
```

### 4.4 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Escape` | Deselect element |
| `Delete` | Delete selected element (with confirmation) |
| `Arrow keys` | Nudge selected by 1% |
| `Shift + Arrow` | Nudge by 5% |
| `Enter` | Edit textbox (when textbox selected) |

### 4.5 Accessibility

```typescript
const canvasA11y = {
  role: 'application',
  'aria-label': `Spread editor, pages ${spread.left_page.number}-${spread.right_page.number}`,
  tabIndex: 0,
};

const elementA11y = (type: 'image' | 'textbox', index: number, isSelected: boolean) => ({
  role: 'button',
  tabIndex: isSelected ? 0 : -1,
  'aria-selected': isSelected,
  'aria-label': `${type} ${index + 1}`,
  'aria-describedby': `${type}-${index}-description`,
});

// Announce selection changes
function announceSelection(element: SelectedElement | null) {
  const message = element
    ? `Selected ${element.type} ${element.index + 1}`
    : 'Selection cleared';
  ariaLive.announce(message);
}
```
