# SketchViewer: Component Design

> **Parent:** [SketchCreativeSpace](component/editor-page/sketch-creative-space/00-sketch-creative-space.md)

**Screenshot:** `screenshots/manuscript-sketch-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               SketchViewer                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SheetPreview (main view area)                                           │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │                                                                    │  │   │
│  │  │                                                                    │  │   │
│  │  │                    [Large Sheet Image]                             │  │   │
│  │  │                                                                    │  │   │
│  │  │                    Character/Prop reference                        │  │   │
│  │  │                    sheet (zoomable)                                │  │   │
│  │  │                                                                    │  │   │
│  │  │                                                                    │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SheetThumbnailList (horizontal filmstrip)                               │   │
│  │  ┌──────┐ ╔══════╗ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐                 │   │
│  │  │  1   │ ║  2   ║ │  3   │ │  4   │ │  5   │ │ + NEW  │  ← scroll →     │   │
│  │  │      │ ║      ║ │      │ │      │ │      │ │        │                 │   │
│  │  └──────┘ ╚══════╝ └──────┘ └──────┘ └──────┘ └────────┘                 │   │
│  │              ↑ selected                                                  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
        ┌─────────────────────────────┐
        │     SketchCreativeSpace     │
        │     (parent component)      │
        └─────────────┬───────────────┘
                      │ (props)
                      ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                               SketchViewer                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: sketchType, selectedIndex, onIndexChange                            │  │
│  │  Store: sheets = useCharacterSheets() or usePropSheets()                    │  │
│  │  Actions: addCharacterSheet, removeCharacterSheet, etc.                     │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                              │                                          │
│         ▼                              ▼                                          │
│  ┌──────────────────────────┐  ┌───────────────────────────────────────────────┐  │
│  │     SheetPreview         │  │           SheetThumbnailList                  │  │
│  │                          │  │                                               │  │
│  │  Props:                  │  │  Props:                                       │  │
│  │  • src: string (URL)     │  │  • sheets: string[]                           │  │
│  │  • alt: string           │  │  • selectedIndex: number                      │  │
│  │  • isZoomable: boolean   │  │  • onSelect: (index) => void                  │  │
│  │                          │  │  • onAdd: () => void                          │  │
│  │  (stateless)             │  │  • canAdd: boolean                            │  │
│  └──────────────────────────┘  └───────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Image sheet viewer with large preview and horizontal filmstrip navigation. Used only for Characters and Props tabs (not Spreads).

**Shared Types:**

```typescript
type SheetSketchType = 'characters' | 'props';  // NOT 'spreads'
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SketchViewerProps {
  sketchType: SheetSketchType;      // 'characters' | 'props' (NOT 'spreads')
  selectedIndex: number;             // Currently selected sheet (0-based)
  onIndexChange: (index: number) => void;
}

interface SketchViewerState {
  // Zoom state for preview
  zoomLevel: number;                 // 1.0 = 100%, 2.0 = 200%, etc.
  isPanning: boolean;
  panOffset: { x: number; y: number };
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors (mode-conditional)
characterSheets = useCharacterSheets();  // string[] (URLs)
propSheets = usePropSheets();            // string[] (URLs)
sheets = sketchType === 'characters' ? characterSheets : propSheets;

// SnapshotStore Actions
{
  addCharacterSheet,
  removeCharacterSheet,
  reorderCharacterSheets,
  addPropSheet,
  removePropSheet,
  reorderPropSheets,
} = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
SketchViewer:
  // Props from parent
  { sketchType, selectedIndex, onIndexChange } = props

  // Store selectors (mode-conditional)
  characterSheets = useCharacterSheets()
  propSheets = usePropSheets()
  sheets = sketchType === 'characters' ? characterSheets : propSheets
  selectedSheet = sheets[selectedIndex]

  // Local state for zoom/pan
  [zoomLevel, setZoomLevel] = useState(1.0)
  [isPanning, setIsPanning] = useState(false)
  [panOffset, setPanOffset] = useState({ x: 0, y: 0 })

  // Actions
  { addCharacterSheet, addPropSheet } = useSnapshotActions()

  handleSheetSelect(index):
    onIndexChange(index)
    // Reset zoom when switching sheets
    setZoomLevel(1.0)
    setPanOffset({ x: 0, y: 0 })

  handleAddSheet():
    // Open file picker or trigger generate
    IF sketchType === 'characters':
      addCharacterSheet(newSheetUrl)
    ELSE:
      addPropSheet(newSheetUrl)

  handleZoom(delta):
    newZoom = clamp(zoomLevel + delta, 0.5, 3.0)
    setZoomLevel(newZoom)

  RENDER Container (flex column):

    // Main preview area
    RENDER SheetPreview với:
      - src: selectedSheet
      - alt: `${sketchType} sheet ${selectedIndex + 1}`
      - zoomLevel
      - panOffset
      - isPanning
      - onZoom: handleZoom
      - onPanStart: () => setIsPanning(true)
      - onPanMove: (offset) => setPanOffset(offset)
      - onPanEnd: () => setIsPanning(false)
      - onError: () => showPlaceholder()

    // Filmstrip navigation
    RENDER SheetThumbnailList với:
      - sheets
      - selectedIndex
      - onSelect: handleSheetSelect
      - onAdd: handleAddSheet
      - canAdd: true
```

### 2.4 Visual

**Normal State (with sheets):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│                                                                                 │
│                ┌───────────────────────────────────────┐                        │
│                │                                       │                        │
│                │     [Character Sheet Image]           │                        │
│                │                                       │                        │
│                │      ┌─────┐  ┌─────┐  ┌─────┐        │                        │
│                │      │Char1│  │Char2│  │Char3│        │                        │
│                │      └─────┘  └─────┘  └─────┘        │                        │
│                │                                       │                        │
│                └───────────────────────────────────────┘                        │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ╔═════╗  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────────┐                                │
│  ║ 1   ║  │ 2   │  │ 3   │  │ 4   │  │ + NEW   │  ← horizontal scroll           │
│  ║     ║  │     │  │     │  │     │  │         │                                │
│  ╚═════╝  └─────┘  └─────┘  └─────┘  └─────────┘                                │
│      ↑ selected                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State (no sheets):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│                                                                                 │
│                          📄 No character sheets yet                             │
│                                                                                 │
│                          Generate sheets from your dummy                        │
│                                                                                 │
│                    ┌─────────────────────────────────────────┐                  │
│                    │     ✨ Generate Character Sheets        │                  │
│                    └─────────────────────────────────────────┘                  │
│                                                                                 │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐                                                                │
│  │   + NEW     │                                                                │
│  └─────────────┘                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Zoomed State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [─] 150% [+]    [Reset]                            ← zoom controls             │
│                                                                                 │
│                ┌───────────────────────────────────────┐                        │
│                │                                       │                        │
│                │     [Zoomed portion of sheet]         │ ← drag to pan          │
│                │           ┌─────────────┐             │                        │
│                │           │    Char2    │             │                        │
│                │           │  (enlarged) │             │                        │
│                │           └─────────────┘             │                        │
│                │                                       │                        │
│                └───────────────────────────────────────┘                        │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────┐  ╔═════╗  ┌─────┐  ┌─────┐  ┌─────────┐                                │
│  │ 1   │  ║ 2   ║  │ 3   │  │ 4   │  │ + NEW   │                                │
│  └─────┘  ╚═════╝  └─────┘  └─────┘  └─────────┘                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks**.
> Sub-components có thể là elements đơn giản.

### 3.1 SheetPreview

**Mục đích:** Large preview area for selected sheet image. Supports zoom and pan.

**Props & Callbacks:**

```typescript
interface SheetPreviewProps {
  src: string | null;           // Image URL
  alt: string;                  // Accessibility label
  zoomLevel: number;            // 0.5 - 3.0
  panOffset: { x: number; y: number };
  isPanning: boolean;

  onZoom: (delta: number) => void;
  onPanStart: () => void;
  onPanMove: (offset: { x: number; y: number }) => void;
  onPanEnd: () => void;
  onError: () => void;
}
```

**Behavior:**
- Mouse wheel: Zoom in/out
- Click + drag: Pan (when zoomed)
- Double-click: Reset zoom to 1.0
- On error: Show placeholder image

**Visual:**

```
Normal:                         Zoomed (2.0x):
┌─────────────────────────┐     ┌─────────────────────────┐
│   [Full sheet image]    │     │  [Cropped, enlarged]    │
│                         │     │      portion            │
│      at 100% zoom       │     │                         │
└─────────────────────────┘     └─────────────────────────┘
                                      ↔ drag to pan
```

### 3.2 SheetThumbnailList

**Mục đích:** Horizontal filmstrip of sheet thumbnails with selection and add button.

**Props & Callbacks:**

```typescript
interface SheetThumbnailListProps {
  sheets: string[];              // Array of image URLs
  selectedIndex: number;
  onSelect: (index: number) => void;
  onAdd: () => void;
  canAdd: boolean;
}
```

**Visual:**

```
┌────────────────────────────────────────────────────────────────────────────────┐
│ ◀ ╔═════╗  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────────┐ ▶                   │
│   ║  1  ║  │  2  │  │  3  │  │  4  │  │  5  │  │ + NEW   │    ← horizontal     │
│   ║     ║  │     │  │     │  │     │  │     │  │         │      scroll         │
│   ╚═════╝  └─────┘  └─────┘  └─────┘  └─────┘  └─────────┘                     │
│       ↑ selected (blue border)                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 SheetThumbnail

**Mục đích:** Single thumbnail in the filmstrip.

**Props & Callbacks:**

```typescript
interface SheetThumbnailProps {
  src: string;                   // Image URL
  index: number;
  isSelected: boolean;
  onClick: () => void;
}
```

**Visual:**

```
Normal:                   Selected:
┌───────────────────┐     ╔═══════════════════╗
│                   │     ║                   ║
│    [thumbnail]    │     ║    [thumbnail]    ║
│                   │     ║                   ║
│        1          │     ║        1          ║
└───────────────────┘     ╚═══════════════════╝
                                 ↑ blue border
```

### 3.4 NewSheetButton

**Mục đích:** Button to add new sheet (trigger generate or file upload).

**Props & Callbacks:**

```typescript
interface NewSheetButtonProps {
  onClick: () => void;
  disabled?: boolean;
}
```

**Visual:**

```
┌───────────────────┐
│                   │
│        +          │
│       NEW         │
│                   │
└───────────────────┘
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Characters/Props Only**
SketchViewer is used ONLY for Characters and Props tabs. Spreads tab uses ManuscriptSpreadView instead.

**Image-based Content**
Sheets are image URLs (not structured data). Each sheet is a reference image generated by AI.

**Zoom/Pan for Detail**
Character/prop sheets often have detailed annotations. Zoom and pan allow users to inspect fine details.

**Add via Generate or Upload**
"+ NEW" button can:
1. Trigger AI generation (using prompt from sidebar)
2. Open file picker for manual upload

### 4.2 Layout Constants

| Element | Value |
|---------|-------|
| Preview max-height | 60vh |
| Preview padding | 24px |
| Filmstrip height | 100px |
| Thumbnail size | 80×80px |
| Thumbnail gap | 8px |
| Selected border | 2px solid primary |

### 4.3 Zoom Behavior

| Action | Result |
|--------|--------|
| Mouse wheel up | Zoom in (+0.25) |
| Mouse wheel down | Zoom out (-0.25) |
| Double-click | Reset to 1.0 |
| Pinch (touch) | Zoom in/out |
| Min zoom | 0.5x (50%) |
| Max zoom | 3.0x (300%) |

### 4.4 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `←` / `→` | Previous/next sheet |
| `Home` / `End` | First/last sheet |
| `+` / `-` | Zoom in/out |
| `0` | Reset zoom |
| `Space` | Toggle pan mode |

### 4.5 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Container | `region` | `aria-label="Sheet viewer"` |
| SheetPreview | `img` | `alt` with sheet description |
| SheetThumbnailList | `listbox` | `aria-label="Sheet thumbnails"`, `aria-orientation="horizontal"` |
| SheetThumbnail | `option` | `aria-selected` |
| NewSheetButton | `button` | `aria-label="Add new sheet"` |

### 4.6 Error States

| State | Visual | Recovery |
|-------|--------|----------|
| Image load failed | Placeholder with retry button | Click to retry |
| No sheets | Empty state with generate CTA | Click to generate |
| Network error | Error message | Auto-retry after 5s |

---
