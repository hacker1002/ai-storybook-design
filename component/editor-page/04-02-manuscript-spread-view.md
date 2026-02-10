# ManuscriptSpreadView: Component Design

> **Note:** Unified component thay thế cả `ManuscriptDummyView` và `ManuscriptFinalizationView`. Used by Prose Dummy, Poetry Dummy, và Finalization steps.

**Screenshots:**
- Edit mode: `component/editor-page/screenshots/manuscript-edit-view.png`
- Grid mode: `component/editor-page/screenshots/manuscript-grid-view.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              ManuscriptSpreadView                                │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  SpreadViewHeader                                                          │  │
│  │  [⚏]                                       ─ ●────── + 100% (or 4 cols)   │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  SWITCH viewMode:                                                          │  │
│  │                                                                            │  │
│  │  'edit': SpreadEditorPanel + SpreadThumbnailList (horizontal)              │  │
│  │  'grid': SpreadThumbnailList (grid layout)                                 │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                    ┌─────────────────────┐
                    │    SnapshotStore    │
                    │   (Zustand global)  │
                    └──────────┬──────────┘
                               │
                     ┌─────────┼─────────┐
                     │         │         │
                     ▼         ▼         ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              ManuscriptSpreadView                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: mode, dummyType?                                                    │  │
│  │  Local State: viewMode, selectedSpreadId, zoomLevel, columnsPerRow          │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                              │                              │           │
│         ▼                              ▼                              ▼           │
│  ┌──────────────────┐          ┌─────────────────────┐        ┌─────────────────┐ │
│  │ SpreadViewHeader │          │  SpreadEditorPanel  │        │SpreadThumbnailList│
│  │                  │          │  (edit mode only)   │        │                 │ │
│  │ Props:           │          │                     │        │ Props:          │ │
│  │ • viewMode       │          │ Props:              │        │ • mode          │ │
│  │ • zoomLevel      │          │ • spreadId          │        │ • dummyType?    │ │
│  │ • columnsPerRow  │          │ • mode, dummyType?  │        │ • selectedId    │ │
│  │ • spreadViewMode │          │ • zoomLevel         │        │ • layout        │ │
│  │                  │          │ • isEditable        │        │ • columnsPerRow │ │
│  │ Callbacks:       │          │ • displayField      │        │ • canAdd        │ │
│  │ • onViewModeToggle│         │                     │        │                 │ │
│  │ • onZoomChange   │          │ (uses both stores)  │        │ Callbacks:      │ │
│  │ • onColumnsChange│          └─────────────────────┘        │ • onSpreadClick │ │
│  └──────────────────┘                                         │ (uses stores)   │ │
│                                                               └─────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Mode Comparison

| Feature | mode='dummy' | mode='finalize' |
|---------|--------------|-----------------|
| Selector | `useDummySpreads(dummyId)` | `useSpreads()` |
| Add Action | `addDummySpread(dummyId, spread)` | `addSpread(spread)` |
| Update Action | `updateDummySpread(dummyId, spreadId, updates)` | `updateSpread(id, updates)` |
| Reorder Action | `reorderDummySpreads(dummyId, from, to)` | `reorderSpreads(from, to)` |
| Purpose | Drafting & layout planning | Final assets for export |
| Image display | `art_note` | `visual_description` |
| Drag-drop | ✅ Yes | ✅ Yes |
| Add spread | ✅ Yes | ✅ Yes |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Unified spread view cho cả Dummy và Finalization steps. Hiển thị spread grid/filmstrip với inline editor panel, thay thế modal-based editing.

**Key Changes từ Design cũ:**
1. Gộp `ManuscriptDummyView` và `ManuscriptFinalizationView` thành 1 component
2. Click spread → inline `SpreadEditorPanel` thay vì modal
3. Dual-purpose slider: Zoom (edit mode) hoặc Columns (grid mode)
4. Toggle button với tooltip "Show full spread view"

**Shared Types:**

```typescript
type SpreadViewMode = 'dummy' | 'finalize';
type ViewMode = 'edit' | 'grid';
type DisplayField = 'art_note' | 'visual_description';

// DummySpread, DummyImage, DummyTextbox defined in parent (03-manuscript-creative-space.md)
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface ManuscriptSpreadViewProps {
  mode: SpreadViewMode;               // 'dummy' | 'finalize'
  dummyId?: string;                   // Required for dummy mode (UUID)
}

interface ManuscriptSpreadViewState {
  // Layout
  viewMode: ViewMode;                 // 'edit' | 'grid'

  // Selection (ID-based for optimized re-renders)
  selectedSpreadId: string | null;

  // View controls (dual-purpose slider)
  zoomLevel: number;                  // 25-200, default 100 (edit mode)
  columnsPerRow: number;              // 1-6, default 4 (grid mode)

  // Drag state
  draggedId: string | null;
  dropTargetId: string | null;
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors (mode-conditional)
dummySpreads = useDummySpreads(dummyId);       // For dummy mode
snapshotSpreads = useSpreads();                 // For finalize mode
spreads = mode === 'dummy' ? dummySpreads : snapshotSpreads;

// ID selectors for optimized list rendering (shallow compare)
dummySpreadIds = useDummySpreadIds(dummyId);   // For dummy mode
snapshotSpreadIds = useSpreadIds();             // For finalize mode
spreadIds = mode === 'dummy' ? dummySpreadIds : snapshotSpreadIds;

// SnapshotStore Actions
{ addDummySpread, updateDummySpread, deleteDummySpread, reorderDummySpreads,
  addSpread, updateSpread, deleteSpread, reorderSpreads } = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
ManuscriptSpreadView:
  // Data from SnapshotStore (mode-conditional)
  spreads = mode === 'dummy' ? useDummySpreads(dummyId) : useSpreads()
  spreadIds = mode === 'dummy' ? useDummySpreadIds(dummyId) : useSpreadIds()
  actions = useSnapshotActions()

  isEditable = true  // Both modes editable
  canAdd = true
  displayField = mode === 'dummy' ? 'art_note' : 'visual_description'

  IF spreadIds.length === 0:
    RENDER EmptyState với:
      - mode === 'dummy' → "No spreads yet. Click + to add."
      - mode === 'finalize' → "No spreads. Run Generate Art Direction."
    RETURN

  RENDER SpreadViewHeader với:
    - viewMode, zoomLevel, columnsPerRow, spreadViewMode: mode
    - onViewModeToggle, onZoomChange, onColumnsChange

  IF viewMode === 'edit':
    IF selectedSpreadId !== null:
      RENDER SpreadEditorPanel với:
        - spreadId: selectedSpreadId
        - mode, dummyId, zoomLevel
        - isEditable, displayField

    RENDER SpreadThumbnailList với:
      - mode, dummyId, selectedId: selectedSpreadId
      - displayField, layout: 'horizontal'
      - isDragEnabled: true, canAdd
      - onSpreadClick: handleSpreadClick

  ELSE (viewMode === 'grid'):
    RENDER SpreadThumbnailList với:
      - mode, dummyId, selectedId: selectedSpreadId
      - displayField, layout: 'grid', columnsPerRow
      - isDragEnabled: true, canAdd
      - onSpreadClick: handleSpreadClick

  // Handlers
  handleSpreadClick(id):
    setSelectedSpreadId(id)
    IF viewMode === 'grid':
      setViewMode('edit')  // Auto-switch to edit

  toggleViewMode():
    setViewMode(viewMode === 'edit' ? 'grid' : 'edit')
    IF switching to 'edit' AND selectedSpreadId === null AND spreadIds.length > 0:
      setSelectedSpreadId(spreadIds[0])
```

### 2.4 Visual

**Edit Mode (viewMode: 'edit'):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── + 100%  │
│   ↑ toggle icon                                             └→ Zoom (25%-200%)  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                ┌────────────────────────────────────────────────┐               │
│                │                 SpreadEditorPanel              │               │
│                │  ┌─────────────────────┬─────────────────────┐ │               │
│                │  │     Left Page       │     Right Page      │ │               │
│                │  │   ┌───────────┐     │                     │ │               │
│                │  │   │  [Image]  │     │  ┌─────────────┐    │ │               │
│                │  │   │           │     │  │  Textbox    │    │ │               │
│                │  │   │ "art_note"│     │  │   ╔═══╗     │    │ │               │
│                │  │   └───────────┘     │  │   ║ T ║     │    │ │               │
│                │  │                     │  │   ╚═══╝     │    │ │               │
│                │  │     2               │  └─────────────┘  3 │ │               │
│                │  └─────────────────────┴─────────────────────┘ │               │
│                └────────────────────────────────────────────────┘               │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │  ┌───┐  │  ║  ┌───┐  ║  │  ┌───┐  │  │  ┌───┐  │  │  ┌───┐  │  │    +    │   │
│  │  │   │  │  ║  │   │  ║  │  │   │  │  │  │   │  │  │  │   │  │  │   NEW   │   │
│  │  └───┘  │  ║  └───┘  ║  │  └───┘  │  │  └───┘  │  │  └───┘  │  │         │   │
│  │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  └─────────┘   │
│  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘    ↑ add       │
│                   ↑ selected (blue border)                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Grid Mode (viewMode: 'grid'):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── +   4  │
│   ↑ toggle (tooltip: "Show full spread view")               └→ Columns (1-6)    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  0  │  1    │  │  2  │  3    │  │  4  │  5    │  │  6  │  7    │             │
│  │ ┌────────┐  │  │ ┌────────┐  │  │ ┌────────┐  │  │ ┌────────┐  │             │
│  │ │"art    │  │  │ │"art    │  │  │ │"art    │  │  │ │"art    │  │             │
│  │ │note"   │  │  │ │note"   │  │  │ │note"   │  │  │ │note"   │  │             │
│  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │             │
│  │     text    │  │     text    │  │     text    │  │     text    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘             │
│       ↕ drag-drop                                                               │
│  ┌─────────────┐  ┌───────────────────┐                                         │
│  │  8  │  9    │  │        +          │  ← NewSpreadButton                      │
│  │ ┌────────┐  │  │   Add New Spread  │                                         │
│  │ │"art    │  │  │                   │                                         │
│  │ │note"   │  │  └───────────────────┘                                         │
│  │ └────────┘  │                                                                │
│  └─────────────┘                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── + 100%  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                             📄 No spreads yet                                   │
│                                                                                 │
│                         ┌─────────────────────┐                                 │
│                         │          +          │                                 │
│                         │   Add First Spread  │  ← dummy mode                   │
│                         └─────────────────────┘                                 │
│                                                                                 │
│                         OR                                                      │
│                                                                                 │
│                         📄 No spreads in snapshot                               │
│                         Run "Generate Art Direction"   ← finalize mode          │
│                         to create spreads.                                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý quan trọng:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - **KHÔNG** thiết kế store integration tại đây — child component tự thiết kế trong file riêng
> - State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng

### 3.1 SpreadViewHeader

📄 **Doc:** [`component/editor-page/04-02-01-spread-view-header.md`](component/editor-page/04-02-01-spread-view-header.md)

**Mục đích:** Header toolbar với toggle button và dual-purpose slider (zoom/columns).

**Props & Callbacks:**

```typescript
interface SpreadViewHeaderProps {
  viewMode: ViewMode;                    // 'edit' | 'grid'
  zoomLevel: number;                     // 25-200, default 100 (edit mode)
  columnsPerRow: number;                 // 1-6, default 4 (grid mode)
  spreadViewMode: SpreadViewMode;        // 'dummy' | 'finalize'

  onViewModeToggle: () => void;
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}
```

**Visual:**

```
Edit Mode:
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── + 100%  │
│   ↑ toggle icon                                             └→ Zoom (25%-200%)  │
└─────────────────────────────────────────────────────────────────────────────────┘

Grid Mode:
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── +   4  │
│   ↑ toggle (tooltip: "Show full spread view")               └→ Columns (1-6)    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.2 SpreadEditorPanel

📄 **Doc:** [`component/editor-page/04-02-02-spread-editor-panel.md`](component/editor-page/04-02-02-spread-editor-panel.md)

**Mục đích:** Inline editor panel cho selected spread, thay thế SpreadEditModal.

**Props & Callbacks:**

```typescript
interface SpreadEditorPanelProps {
  spreadId: string;                  // ID-based subscription
  mode: SpreadViewMode;
  dummyId?: string;                  // Required for dummy mode
  zoomLevel: number;
  isEditable: boolean;
  displayField: DisplayField;        // 'art_note' | 'visual_description'
  // No spread prop - uses store selectors internally
  // No callbacks - uses store actions directly
}
```

**Store Integration (inside component):**

```typescript
// SnapshotStore - mode conditional
spread = mode === 'dummy'
  ? useDummySpreadById(dummyId, spreadId)
  : useSpreadById(spreadId);
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadEditorPanel                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                            SpreadCanvas                                   │  │
│  │  ┌─────────────────────────────┬─────────────────────────────┐            │  │
│  │  │         Left Page           │         Right Page          │            │  │
│  │  │                             │                             │            │  │
│  │  │    ┌─────────────────┐      │      ┌─────────────────┐    │            │  │
│  │  │    │     Image       │      │      │     Textbox     │    │            │  │
│  │  │    │   "art_note"    │      │      │    ╔═══════╗    │    │            │  │
│  │  │    └─────────────────┘      │      │    ║   T   ║    │    │            │  │
│  │  │                             │      │    ╚═══════╝    │    │            │  │
│  │  │           2                 │             3               │            │  │
│  │  └─────────────────────────────┴─────────────────────────────┘            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│  Features: Click to select, drag to move, resize handles, inline text edit      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 SpreadThumbnailList

📄 **Doc:** [`component/editor-page/04-02-03-spread-thumbnail-list.md`](component/editor-page/04-02-03-spread-thumbnail-list.md)

**Mục đích:** Unified thumbnails container cho cả horizontal filmstrip và vertical grid.

**Props & Callbacks:**

```typescript
type ThumbnailListLayout = 'horizontal' | 'grid';

interface SpreadThumbnailListProps {
  mode: SpreadViewMode;
  dummyId?: string;                        // Required for dummy mode
  selectedId: string | null;               // ID-based selection
  displayField: DisplayField;
  isDragEnabled: boolean;
  canAdd: boolean;

  // Layout configuration
  layout: ThumbnailListLayout;             // 'horizontal' = filmstrip, 'grid' = vertical
  columnsPerRow?: number;                  // Only for grid layout, default 4

  // Selection callback only (parent manages selection state)
  onSpreadClick: (id: string) => void;
  // Note: Add/Reorder handled via store actions internally
}
```

**Store Integration (inside component):**

```typescript
// SnapshotStore - mode conditional
spreadIds = mode === 'dummy'
  ? useDummySpreadIds(dummyId)
  : useSpreadIds();
```

**Visual (layout='horizontal'):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ◀ ┌─────────┐  ╔═════════╗  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────┐ ▶ │
│    │   0-1   │  ║   2-3   ║  │   4-5   │  │   6-7   │  │   8-9   │  │  NEW  │   │
│    └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘  └───────┘   │
│                     ↑ selected                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Visual (layout='grid'):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │     0-1     │  │     2-3     │  │     4-5     │  │     6-7     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘             │
│  ┌─────────────┐  ┌─────────────────┐                                           │
│  │     8-9     │  │       + NEW     │                                           │
│  └─────────────┘  └─────────────────┘                                           │
│                        ↓ vertical scroll                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 3.4 SpreadThumbnail

📄 **Doc:** [`component/editor-page/04-02-03-01-spread-thumbnail.md`](component/editor-page/04-02-03-01-spread-thumbnail.md)

**Mục đích:** Thumbnail preview của một spread, dùng chung cho cả horizontal và grid layouts.

**Props & Callbacks:**

```typescript
interface SpreadThumbnailProps {
  spreadId: string;                      // ID-based, uses store internally
  mode: SpreadViewMode;
  dummyId?: string;                      // Required for dummy mode
  isSelected: boolean;
  displayField: DisplayField;
  isDragging?: boolean;
  isDropTarget?: boolean;
  isDragEnabled?: boolean;
  size?: 'small' | 'medium';             // small for horizontal, medium for grid

  onClick: () => void;
}
```

**Store Integration (inside component):**

```typescript
// SnapshotStore
spread = mode === 'dummy'
  ? useDummySpreadById(dummyId, spreadId)
  : useSpreadById(spreadId);
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Unified Component**
Gộp DummyView và FinalizationView thành 1 component với `mode` prop. Lý do: Cả 2 có cùng data structure (`spreads[]`), cùng UI pattern, chỉ khác selector/action và display field.

**Inline Editor thay Modal**
Thay SpreadEditModal bằng SpreadEditorPanel inline. Lý do: UX tốt hơn, không bị context switch, dễ so sánh với filmstrip.

**ID-based Subscription**
SpreadEditorPanel và SpreadThumbnail nhận `spreadId` và fetch data từ store. Lý do: Tối ưu re-render - chỉ component nào có data thay đổi mới re-render.

**Dual-Purpose Slider**
Slider behavior khác nhau dựa trên viewMode:
- Edit mode: Zoom level (25%-200%)
- Grid mode: Columns per row (1-6)

Lý do: Space-efficient, cùng 1 control phục vụ 2 mục đích.

### 4.2 Store Integration Notes

**ID Selectors:** Component uses `useDummySpreadIds(type)` và `useSpreadIds()` for optimized list rendering. These selectors use shallow comparison để minimize re-renders khi chỉ spread order/content thay đổi mà không thay đổi IDs.

**Mode-conditional Actions:**

```typescript
const handleAdd = () => {
  const newSpread = createEmptySpread();
  mode === 'dummy'
    ? addDummySpread(dummyId!, newSpread)
    : addSpread(newSpread);
};

const handleReorder = (fromIdx: number, toIdx: number) => {
  mode === 'dummy'
    ? reorderDummySpreads(dummyId!, fromIdx, toIdx)
    : reorderSpreads(fromIdx, toIdx);
};
```

### 4.3 Layout Constants

| Element | Value | Note |
|---------|-------|------|
| Header height | 48px | Fixed |
| Filmstrip height | 120px | Fixed when editor visible |
| Editor min height | 300px | Minimum |
| Filmstrip thumbnail | 100×80px | Fixed size |
| Grid thumbnail | Dynamic | Based on columns và container width |

### 4.4 State Persistence

Persist view preferences to localStorage với key `spread-view-prefs`:

```typescript
interface ViewPreferences {
  viewMode: ViewMode;                    // 'edit' | 'grid'
  zoomLevel: number;                     // 25-200
  columnsPerRow: number;                 // 1-6
}
```

**Defaults:** `viewMode: 'edit'`, `zoomLevel: 100`, `columnsPerRow: 4`

### 4.5 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `←` / `→` | Navigate prev/next spread |
| `Home` / `End` | First/last spread |
| `G` | Toggle view mode |
| `+` / `-` | Zoom in/out (edit) or adjust columns (grid) |
| `Delete` | Delete selected spread (with confirmation) |
| `Ctrl+Z` | Undo last change |

### 4.6 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Filmstrip | `listbox` | `aria-label="Spread thumbnails"`, `aria-orientation="horizontal"` |
| Thumbnail | `option` | `aria-selected`, `aria-label="Spread {n}, pages {x}-{y}"` |
| Editor panel | `region` | `aria-label="Spread editor"`, `aria-live="polite"` |

---
