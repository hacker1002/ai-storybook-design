# SelectionFrame: Internal Component Design

> **Parent:** [SpreadEditorPanel](./02-spread-editor-panel.md)
>
> **Role:** Internal component. Not customizable via render props. Handles ALL drag/resize interactions.

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

## 2. Component Design

### 2.1 Overview

**Mục đích:** Visual selection overlay với 8 resize handles. Handles ALL drag/resize interactions for selected elements.

> **Note:** Stateless component. All state managed by parent SpreadEditorPanel.

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
  showHandles: boolean;            // false during drag, text editing, or canResize=false
  activeHandle: ResizeHandle | null;  // Highlight active handle

  // Feature flags
  canDrag?: boolean;               // default: true - enable drag to move
  canResize?: boolean;             // default: true - enable resize handles

  // Drag callbacks (only called if canDrag=true)
  onDragStart: () => void;
  onDrag: (delta: Point) => void;
  onDragEnd: () => void;

  // Resize callbacks (only called if canResize=true)
  onResizeStart: (handle: ResizeHandle) => void;
  onResize: (handle: ResizeHandle, delta: Point) => void;
  onResizeEnd: () => void;
}
```

**Local State:** None (pure presentational)

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
calculateDelta(event, startPos, canvasRect): Point
  // Convert current mouse to canvas-relative, zoom-adjusted
  currentX = (event.clientX - canvasRect.left) / (zoomLevel / 100)
  currentY = (event.clientY - canvasRect.top) / (zoomLevel / 100)

  // Delta in percentage
  deltaX = ((currentX - startPos.x) / canvasRect.width) * 100
  deltaY = ((currentY - startPos.y) / canvasRect.height) * 100

  return { x: deltaX, y: deltaY }
```

### 2.5 Render Logic (pseudo)

```
SelectionFrame:
  frameRef = useRef()
  startPosRef = useRef()
  currentHandleRef = useRef()

  handleFrameMouseDown(e):
    IF !canDrag: RETURN  // Skip if drag disabled
    e.stopPropagation()
    e.preventDefault()
    canvasRect = frameRef.current.parentElement.getBoundingClientRect()
    startPosRef.current = {
      x: (e.clientX - canvasRect.left) / (zoomLevel / 100),
      y: (e.clientY - canvasRect.top) / (zoomLevel / 100)
    }
    onDragStart()

    document.addEventListener('mousemove', handleMouseMove)
    document.addEventListener('mouseup', handleMouseUp)

  handleHandleMouseDown(e, handle):
    IF !canResize: RETURN  // Skip if resize disabled
    e.stopPropagation()
    e.preventDefault()
    canvasRect = frameRef.current.parentElement.getBoundingClientRect()
    startPosRef.current = { ... }
    currentHandleRef.current = handle
    onResizeStart(handle)

    document.addEventListener('mousemove', handleMouseMove)
    document.addEventListener('mouseup', handleMouseUp)

  handleMouseMove(e):
    delta = calculateDelta(e, startPosRef.current, canvasRect)
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

    document.removeEventListener(...)

  RENDER FrameContainer (position: absolute):
    ref: frameRef
    style:
      left: geometry.x + '%'
      top: geometry.y + '%'
      width: geometry.width + '%'
      height: geometry.height + '%'
      border: '2px solid #2196F3'
      cursor: canDrag ? 'move' : 'default'
      zIndex: 100

    onMouseDown: canDrag ? handleFrameMouseDown : null

    IF showHandles && canResize:
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
            background: SELECTION_COLOR
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
● = active handle (10px)
○ = inactive handles (8px)
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Internal component | Not customizable | Consistent behavior |
| Stateless | Parent manages | Single source of truth |
| Global listeners | Document events | Works outside bounds |
| 8 Handles | Standard pattern | Intuitive resize |

### 3.2 Constants

```typescript
const MIN_SIZE = 5;               // Minimum 5% width/height
const HANDLE_SIZE = 8;            // pixels
const ACTIVE_HANDLE_SIZE = 10;    // pixels
const BORDER_WIDTH = 2;           // pixels
const SELECTION_COLOR = '#2196F3';

const Z_INDEX = {
  frame: 100,
  handle: 101,
};
```

### 3.3 Edge Cases

| Case | Behavior |
|------|----------|
| Resize below minimum | Clamp to MIN_SIZE |
| Drag outside canvas | Clamp to bounds (0-100) |
| Mouse up outside window | Cleanup via document listener |

### 3.4 Performance

- Use `will-change: transform` during drag/resize
- Throttle `handleMouseMove` via `requestAnimationFrame`
- Cleanup global listeners on unmount
- Use refs to avoid stale closures

---
