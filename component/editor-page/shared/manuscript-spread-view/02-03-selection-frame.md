# SelectionFrame: Component Design

> **Parent:** [SpreadEditorPanel](component/editor-page/shared/manuscript-spread-view/02-spread-editor-panel.md)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        SelectionFrame                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     FrameBorder                           │  │
│  │  ┌───●───┬───●───┐                                        │  │
│  │  │  nw   │   n   │  ne                                    │  │
│  │  ●───────┼───────●                                        │  │
│  │  │   w   │       │   e    ← ResizeHandle (x8)             │  │
│  │  ├───────┼───────┤                                        │  │
│  │  │  sw   │   s   │  se                                    │  │
│  │  └───●───┴───●───┘                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        SelectionFrame                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Props: geometry, zoomLevel, showHandles, activeHandle    │ │
│  │  Callbacks: onDragStart, onDrag, onDragEnd,               │ │
│  │             onResizeStart, onResize, onResizeEnd          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│         ┌────────────────────┴────────────────────┐             │
│         ▼                                         ▼             │
│  ┌─────────────┐                          ┌─────────────┐      │
│  │ FrameBody   │                          │ResizeHandle │      │
│  │ mousedown → │                          │ mousedown → │      │
│  │ onDragStart │                          │onResizeStart│      │
│  └─────────────┘                          └─────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │  SpreadEditorPanel  │
                   │  (manages state)    │
                   └─────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Visual selection overlay với 8 resize handles. Handles ALL drag/resize interactions for selected elements. Rendered as overlay by SpreadEditorPanel when element is selected.

> **Note:** Stateless component. All state (isDragging, isResizing, activeHandle) managed by parent SpreadEditorPanel.

**Shared Types:**

```typescript
type ResizeHandle = 'n' | 's' | 'e' | 'w' | 'nw' | 'ne' | 'sw' | 'se';

interface Geometry {
  x: number;      // percentage
  y: number;      // percentage
  width: number;  // percentage
  height: number; // percentage
}

interface Point {
  x: number;
  y: number;
}
```

### 2.2 Interface

**Props:**

```typescript
interface SelectionFrameProps {
  geometry: Geometry;
  zoomLevel: number;               // For accurate delta calculation
  showHandles: boolean;            // false during drag
  activeHandle: ResizeHandle | null;  // Highlight active handle

  onDragStart: () => void;
  onDrag: (delta: Point) => void;
  onDragEnd: () => void;
  onResizeStart: (handle: ResizeHandle) => void;
  onResize: (handle: ResizeHandle, delta: Point) => void;
  onResizeEnd: () => void;
}
```

**Local State:** None (pure presentational, state managed by parent)

**Store Integration:** None

### 2.3 Handle Configuration

```typescript
const HANDLE_SIZE = 8;  // pixels

const HANDLE_POSITIONS: Record<ResizeHandle, { x: string; y: string }> = {
  nw: { x: '0%',   y: '0%'   },
  n:  { x: '50%',  y: '0%'   },
  ne: { x: '100%', y: '0%'   },
  w:  { x: '0%',   y: '50%'  },
  e:  { x: '100%', y: '50%'  },
  sw: { x: '0%',   y: '100%' },
  s:  { x: '50%',  y: '100%' },
  se: { x: '100%', y: '100%' },
};

const HANDLE_CURSORS: Record<ResizeHandle, string> = {
  nw: 'nwse-resize',
  n:  'ns-resize',
  ne: 'nesw-resize',
  w:  'ew-resize',
  e:  'ew-resize',
  sw: 'nesw-resize',
  s:  'ns-resize',
  se: 'nwse-resize',
};
```

### 2.4 Delta Calculation

```typescript
// Called internally, uses zoomLevel from props
// startPos is stored as canvas-relative coordinates (not clientX/Y)
calculateDelta(event, startPos, canvasRect): Point
  // Convert current mouse to canvas-relative, zoom-adjusted
  currentX = (event.clientX - canvasRect.left) / (zoomLevel / 100)
  currentY = (event.clientY - canvasRect.top) / (zoomLevel / 100)

  // Delta in percentage (startPos already canvas-relative)
  deltaX = ((currentX - startPos.x) / canvasRect.width) * 100
  deltaY = ((currentY - startPos.y) / canvasRect.height) * 100

  return { x: deltaX, y: deltaY }
```

### 2.5 Render Logic (pseudo)

```
SelectionFrame:
  frameRef = useRef()
  startPosRef = useRef()          // Canvas-relative start position
  currentHandleRef = useRef()     // Avoids race condition with activeHandle prop
  isDraggingRef = useRef(false)   // Track drag vs resize mode

  handleFrameMouseDown(e):
    e.stopPropagation()
    e.preventDefault()
    canvasRect = frameRef.current.parentElement.getBoundingClientRect()
    // Store canvas-relative coords (not raw clientX/Y)
    startPosRef.current = {
      x: (e.clientX - canvasRect.left) / (zoomLevel / 100),
      y: (e.clientY - canvasRect.top) / (zoomLevel / 100)
    }
    onDragStart()

    // Attach global listeners
    document.addEventListener('mousemove', handleMouseMove)
    document.addEventListener('mouseup', handleMouseUp)

  handleHandleMouseDown(e, handle):
    e.stopPropagation()
    e.preventDefault()
    canvasRect = frameRef.current.parentElement.getBoundingClientRect()
    startPosRef.current = {
      x: (e.clientX - canvasRect.left) / (zoomLevel / 100),
      y: (e.clientY - canvasRect.top) / (zoomLevel / 100)
    }
    // Store handle locally to avoid race condition with prop update
    currentHandleRef.current = handle
    onResizeStart(handle)

    document.addEventListener('mousemove', handleMouseMove)
    document.addEventListener('mouseup', handleMouseUp)

  handleMouseMove(e):
    canvasRect = frameRef.current.parentElement.getBoundingClientRect()
    delta = calculateDelta(e, startPosRef.current, canvasRect)

    // Use ref instead of prop to avoid race condition
    IF currentHandleRef.current:
      onResize(currentHandleRef.current, delta)
    ELSE:
      onDrag(delta)

  handleMouseUp():
    IF currentHandleRef.current:
      onResizeEnd()
      currentHandleRef.current = null
    ELSE:
      onDragEnd()

    document.removeEventListener('mousemove', handleMouseMove)
    document.removeEventListener('mouseup', handleMouseUp)

  // Cleanup on unmount
  useEffect cleanup:
    document.removeEventListener('mousemove', handleMouseMove)
    document.removeEventListener('mouseup', handleMouseUp)

  RENDER FrameContainer (position: absolute):
    ref: frameRef
    style:
      left: geometry.x + '%'
      top: geometry.y + '%'
      width: geometry.width + '%'
      height: geometry.height + '%'
      border: '2px solid #2196F3'
      cursor: 'move'
      zIndex: 100
      pointerEvents: 'auto'

    onMouseDown: handleFrameMouseDown

    IF showHandles:
      FOR EACH handle IN HANDLE_POSITIONS:
        isActive = handle === activeHandle
        RENDER HandleDot với:
          style:
            position: 'absolute'
            left: HANDLE_POSITIONS[handle].x
            top: HANDLE_POSITIONS[handle].y
            transform: 'translate(-50%, -50%)'
            width: isActive ? HANDLE_SIZE + 2 : HANDLE_SIZE
            height: isActive ? HANDLE_SIZE + 2 : HANDLE_SIZE
            background: isActive ? ACTIVE_HANDLE_COLOR : SELECTION_COLOR
            borderRadius: '50%'
            cursor: HANDLE_CURSORS[handle]
            zIndex: 101

          onMouseDown: (e) => handleHandleMouseDown(e, handle)
```

### 2.6 Visual

**Handles Visible:**

```
╔═══●═══╤═══●═══╗
●               ●
╟───────┼───────╢
●               ●
╚═══●═══╧═══●═══╝
● = handle (8px, blue, interactive)
```

**Dragging (handles hidden):**

```
╔═══════════════╗
║               ║
║    Moving     ║
║               ║
╚═══════════════╝
No handles, solid border
```

**Resizing (active handle highlighted):**

```
╔═══○═══╤═══○═══╗
○               ●← active (larger)
╟───────┼───────╢
○               ○
╚═══○═══╧═══○═══╝
● = active handle (highlighted, 10px)
○ = inactive handles (normal, 8px)
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Handle Count | 8 | Standard resize pattern |
| Handle Shape | Circle | Clear affordance, no rotation ambiguity |
| Border Style | Solid 2px | Visible, not distracting |
| Z-Index | 100/101 | Above elements, handles above frame |
| State Location | Parent (SpreadEditorPanel) | Single source of truth |
| Mouse Tracking | Global listeners | Continues working outside frame bounds |

### 3.2 Constants

```typescript
const MIN_SIZE = 5;               // Minimum 5% width/height
const HANDLE_SIZE = 8;            // pixels
const ACTIVE_HANDLE_SIZE = 10;    // pixels
const BORDER_WIDTH = 2;           // pixels
const SELECTION_COLOR = '#2196F3';
const ACTIVE_HANDLE_COLOR = '#1976D2';

const Z_INDEX = {
  frame: 100,
  handle: 101,
};
```

### 3.3 Edge Cases

| Case | Behavior |
|------|----------|
| Resize below minimum | Clamp to MIN_SIZE |
| Drag outside canvas | Clamp to canvas bounds (0-100) |
| Rapid mouse movement | Handled by global listeners |
| Mouse up outside window | Cleanup via document listener |

### 3.4 Accessibility

```typescript
<div
  role="group"
  aria-label="Selection controls"
  aria-roledescription="Drag to move, use handles to resize"
>
  {handles.map(handle => (
    <div
      role="slider"
      aria-label={`Resize ${handle}`}
      tabIndex={0}
      onKeyDown={handleKeyResize}
    />
  ))}
</div>
```

### 3.5 Performance

- Use `will-change: transform` during drag/resize
- Throttle `handleMouseMove` (16ms / 60fps) via `requestAnimationFrame`
- Cleanup global listeners on unmount via `useEffect` return
- Use refs for interaction state to avoid stale closures

---
