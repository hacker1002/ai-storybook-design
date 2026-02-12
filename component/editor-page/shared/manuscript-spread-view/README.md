# ManuscriptSpreadView: Component Design

> **Note:** Props-driven component với render props pattern. Data-agnostic - consumer cung cấp data và render functions.

**Screenshots:**
- Edit mode: `component/editor-page/screenshots/manuscript-edit-view.png`
- Grid mode: `component/editor-page/screenshots/manuscript-grid-view.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         ManuscriptSpreadView<TSpread>                            │
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
Consumer Component
       │
       │ passes spreads[], callbacks, render props
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    ManuscriptSpreadView<TSpread>                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Props:                                                    │  │
│  │  • spreads: TSpread[]                                      │  │
│  │  • selectedSpreadId?: string                               │  │
│  │  • onSelectSpread, onUpdateSpread, etc.                    │  │
│  │  • renderImageItem, renderTextItem, etc.                   │  │
│  │  • canAddSpread, canDeleteSpread, canReorderSpread         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐              │
│         ▼                    ▼                    ▼              │
│  ┌───────────────┐   ┌─────────────────┐   ┌──────────────────┐  │
│  │SpreadViewHeader│  │SpreadEditorPanel│   │SpreadThumbnailList│ │
│  │               │   │                 │   │                  │  │
│  │ viewMode,     │   │ spreadId,       │   │ spreads,         │  │
│  │ zoom, columns │   │ renderItems,    │   │ selectedId,      │  │
│  │               │   │ render*Item,    │   │ render*Item      │  │
│  │               │   │ render*Toolbar  │   │                  │  │
│  └───────────────┘   └─────────────────┘   └──────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Props-driven spread view với render props pattern cho flexible item rendering. Consumer controls data source và item appearance.

**Key Characteristics:**
1. Generic type `<TSpread>` cho reusable với different spread types
2. Render props cho items với full context (item + spreadId + index + isSelected + callbacks)
3. Optional toolbars - null if not provided
4. Feature flags control add/delete/reorder capabilities

**Shared Types:**

```typescript
type ViewMode = 'edit' | 'grid';
type ItemType = 'image' | 'text' | 'object' | 'animation';

// Minimum spread structure required
interface BaseSpread {
  id: string;
  left_page: { number: number };
  right_page: { number: number };
  images: SpreadImage[];
  textboxes: SpreadTextbox[];
  objects?: SpreadObject[];
  animations?: SpreadAnimation[];
}
```

### 2.2 Interface

**Props:**

```typescript
interface ManuscriptSpreadViewProps<TSpread extends BaseSpread> {
  // === Data (required) ===
  spreads: TSpread[];

  // === Selection ===
  selectedSpreadId?: string;
  onSelectSpread?: (spreadId: string) => void;

  // === Spread-level callbacks ===
  onUpdateSpread?: (spreadId: string, updates: Partial<TSpread>) => void;
  onReorderSpread?: (spreadId: string, newIndex: number) => void;
  onAddSpread?: () => void;
  onDeleteSpread?: (spreadId: string) => void;

  // === Render configuration (required) ===
  renderItems: ItemType[];  // ['image', 'text', 'object', 'animation']

  // === Item render functions (required if item type in renderItems) ===
  renderImageItem?: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem?: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // === Toolbar render functions (optional, null if not provided) ===
  renderImageToolbar?: (context: ImageToolbarContext<TSpread>) => ReactNode;
  renderTextToolbar?: (context: TextToolbarContext<TSpread>) => ReactNode;
  renderObjectToolbar?: (context: ObjectToolbarContext<TSpread>) => ReactNode;
  renderAnimationToolbar?: (context: AnimationToolbarContext<TSpread>) => ReactNode;

  // === Feature flags ===
  canAddSpread?: boolean;     // default: false
  canDeleteSpread?: boolean;  // default: false
  canReorderSpread?: boolean; // default: false
}
```

**Local State:**

```typescript
interface ManuscriptSpreadViewState {
  // Layout
  viewMode: ViewMode;                 // 'edit' | 'grid'

  // View controls (dual-purpose slider)
  zoomLevel: number;                  // 25-200, default 100 (edit mode)
  columnsPerRow: number;              // 1-6, default 4 (grid mode)

  // NOTE: Drag state (draggedId, dropTargetId) managed internally by SpreadThumbnailList
}
```

### 2.3 Context Interfaces

**Base Context:**

```typescript
interface BaseItemContext<TSpread> {
  item: unknown;              // Item data (image/text/object/animation)
  itemIndex: number;          // Index within items array
  spreadId: string;           // Parent spread ID
  spread: TSpread;            // Full spread data
  isSelected: boolean;        // Item selection state
  isSpreadSelected: boolean;  // Spread selection state
}
```

**Item-Specific Contexts:**

```typescript
interface ImageItemContext<TSpread> extends BaseItemContext<TSpread> {
  item: SpreadImage;
  onSelect: () => void;
  onUpdate: (updates: Partial<SpreadImage>) => void;
  onDelete: () => void;
}

interface TextItemContext<TSpread> extends BaseItemContext<TSpread> {
  item: SpreadTextbox;
  onSelect: () => void;
  onTextChange: (text: string) => void;
  onUpdate: (updates: Partial<SpreadTextbox>) => void;
  onDelete: () => void;
}

interface ObjectItemContext<TSpread> extends BaseItemContext<TSpread> {
  item: SpreadObject;
  onSelect: () => void;
  onUpdate: (updates: Partial<SpreadObject>) => void;
  onDelete: () => void;
}

interface AnimationItemContext<TSpread> extends BaseItemContext<TSpread> {
  item: SpreadAnimation;
  onSelect: () => void;
  onUpdate: (updates: Partial<SpreadAnimation>) => void;
  onDelete: () => void;
}
```

**Toolbar Contexts:**

```typescript
interface ImageToolbarContext<TSpread> extends ImageItemContext<TSpread> {
  onGenerateImage: () => void;
  onReplaceImage: () => void;
}

interface TextToolbarContext<TSpread> extends TextItemContext<TSpread> {
  onFormatText: (format: TextFormat) => void;
}

interface ObjectToolbarContext<TSpread> extends ObjectItemContext<TSpread> {
  onEditObject: () => void;
}

interface AnimationToolbarContext<TSpread> extends AnimationItemContext<TSpread> {
  onPlayAnimation: () => void;
  onEditAnimation: () => void;
}
```

### 2.4 Render Logic (pseudo)

```
ManuscriptSpreadView<TSpread>:
  // Data from props
  spreads = props.spreads
  selectedSpread = spreads.find(s => s.id === selectedSpreadId)

  IF spreads.length === 0:
    RENDER EmptyState với:
      - "No spreads yet."
      - IF canAddSpread: Add button
    RETURN

  RENDER SpreadViewHeader với:
    - viewMode, zoomLevel, columnsPerRow
    - onViewModeToggle, onZoomChange, onColumnsChange

  IF viewMode === 'edit':
    IF selectedSpread:
      RENDER SpreadEditorPanel với:
        - spread: selectedSpread
        - spreadIndex: spreads.findIndex(s => s.id === selectedSpreadId)
        - zoomLevel, isEditable: true
        - renderItems, render*Item, render*Toolbar
        - onUpdate*

    RENDER SpreadThumbnailList với:
      - spreads, selectedId: selectedSpreadId
      - layout: 'horizontal'
      - renderItems, render*Item
      - canAdd, canReorder
      - onSpreadClick, onReorderSpread, onAddSpread

  ELSE (viewMode === 'grid'):
    RENDER SpreadThumbnailList với:
      - spreads, selectedId: selectedSpreadId
      - layout: 'grid', columnsPerRow
      - renderItems, render*Item
      - canAdd, canReorder
      - onSpreadClick, onReorderSpread, onAddSpread

  // Handlers
  handleSpreadClick(id):
    onSelectSpread?.(id)
    IF viewMode === 'grid':
      setViewMode('edit')

  toggleViewMode():
    setViewMode(viewMode === 'edit' ? 'grid' : 'edit')
    IF switching to 'edit' AND !selectedSpreadId AND spreads.length > 0:
      onSelectSpread?.(spreads[0].id)
```

### 2.5 Visual

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
│                │  │   │ (rendered │     │  │  Textbox    │    │ │               │
│                │  │   │  by prop) │     │  │ (rendered   │    │ │               │
│                │  │   └───────────┘     │  │  by prop)   │    │ │               │
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
│  └─────────┘  ╚═════════╝  └─────────┘  └─────────┘  └─────────┘   (if canAdd)  │
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
│  │ │(render │  │  │ │(render │  │  │ │(render │  │  │ │(render │  │             │
│  │ │ props) │  │  │ │ props) │  │  │ │ props) │  │  │ │ props) │  │             │
│  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │  │ └────────┘  │             │
│  │     text    │  │     text    │  │     text    │  │     text    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘             │
│       ↕ drag-drop (if canReorder)                                               │
│  ┌─────────────┐  ┌───────────────────┐                                         │
│  │  8  │  9    │  │        +          │  ← NewSpreadButton (if canAdd)          │
│  │ ┌────────┐  │  │   Add New Spread  │                                         │
│  │ │(render │  │  │                   │                                         │
│  │ │ props) │  │  └───────────────────┘                                         │
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
│                         │          +          │  ← Only if canAddSpread         │
│                         │   Add First Spread  │                                 │
│                         └─────────────────────┘                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Note:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - Child components receive render props from parent

### 3.1 SpreadViewHeader

📄 **Doc:** [01-spread-view-header.md](./01-spread-view-header.md)

**Mục đích:** Header toolbar với toggle button và dual-purpose slider (zoom/columns).

**Props & Callbacks:**

```typescript
interface SpreadViewHeaderProps {
  viewMode: ViewMode;                    // 'edit' | 'grid'
  zoomLevel: number;                     // 25-200, default 100 (edit mode)
  columnsPerRow: number;                 // 1-6, default 4 (grid mode)

  onViewModeToggle: () => void;
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}
```

---

### 3.2 SpreadEditorPanel

📄 **Doc:** [02-spread-editor-panel.md](./02-spread-editor-panel.md)

**Mục đích:** Inline editor panel cho selected spread với render props.

**Props & Callbacks:**

```typescript
interface SpreadEditorPanelProps<TSpread extends BaseSpread> {
  spread: TSpread;
  spreadIndex: number;
  zoomLevel: number;
  isEditable: boolean;

  // Render configuration
  renderItems: ItemType[];
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // Toolbar render (optional)
  renderImageToolbar?: (context: ImageToolbarContext<TSpread>) => ReactNode;
  renderTextToolbar?: (context: TextToolbarContext<TSpread>) => ReactNode;
  renderObjectToolbar?: (context: ObjectToolbarContext<TSpread>) => ReactNode;
  renderAnimationToolbar?: (context: AnimationToolbarContext<TSpread>) => ReactNode;

  // Callbacks
  onUpdateSpread: (updates: Partial<TSpread>) => void;
  onUpdateImage: (imageIndex: number, updates: Partial<SpreadImage>) => void;
  onUpdateTextbox: (textboxIndex: number, updates: Partial<SpreadTextbox>) => void;
  onDeleteImage?: (imageIndex: number) => void;
  onDeleteTextbox?: (textboxIndex: number) => void;
}
```

---

### 3.3 SpreadThumbnailList

📄 **Doc:** [03-spread-thumbnail-list.md](./03-spread-thumbnail-list.md)

**Mục đích:** Thumbnails container cho spread navigation và reorder.

**Props & Callbacks:**

```typescript
interface SpreadThumbnailListProps<TSpread extends BaseSpread> {
  spreads: TSpread[];
  selectedId: string | null;

  // Layout
  layout: ThumbnailListLayout;           // 'horizontal' | 'grid'
  columnsPerRow?: number;                // Grid only, default 4

  // Render configuration (same as parent)
  renderItems: ItemType[];
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;
  renderObjectItem?: (context: ObjectItemContext<TSpread>) => ReactNode;
  renderAnimationItem?: (context: AnimationItemContext<TSpread>) => ReactNode;

  // Feature flags
  canAdd: boolean;
  canReorder: boolean;
  canDelete: boolean;

  // Callbacks
  onSpreadClick: (spreadId: string) => void;
  onReorderSpread?: (fromIndex: number, toIndex: number) => void;
  onAddSpread?: () => void;
  onDeleteSpread?: (spreadId: string) => void;
}
```

---

### 3.4 SpreadThumbnail

📄 **Doc:** [03-01-spread-thumbnail.md](./03-01-spread-thumbnail.md)

**Mục đích:** Thumbnail preview của một spread.

**Props & Callbacks:**

```typescript
interface SpreadThumbnailProps<TSpread extends BaseSpread> {
  spread: TSpread;
  spreadIndex: number;
  isSelected: boolean;
  size: 'small' | 'medium';

  // Render configuration (same as parent, view-only mode)
  renderItems: ItemType[];
  renderImageItem: (context: ImageItemContext<TSpread>) => ReactNode;
  renderTextItem: (context: TextItemContext<TSpread>) => ReactNode;

  // Drag state
  isDragEnabled?: boolean;
  isDragging?: boolean;
  isDropTarget?: boolean;

  // Callbacks
  onClick: () => void;
  onDragStart?: () => void;
  onDragOver?: () => void;
  onDragEnd?: () => void;
}
```

---

## 4. Utility Components

> **Note:** Optional helper components consumers can import and use inside render props.

### 4.1 EditableImage

📄 **Doc:** [02-01-editable-image.md](./02-01-editable-image.md)

Default image renderer. Consumer can use in `renderImageItem`:

```typescript
renderImageItem={(context) => (
  <EditableImage
    image={context.item}
    index={context.itemIndex}
    isSelected={context.isSelected}
    isEditable={true}
    onSelect={context.onSelect}
  />
)}
```

### 4.2 EditableTextbox

📄 **Doc:** [02-02-editable-textbox.md](./02-02-editable-textbox.md)

Default text renderer. Consumer can use in `renderTextItem`:

```typescript
renderTextItem={(context) => (
  <EditableTextbox
    text={context.item.text}
    geometry={context.item.geometry}
    typography={context.item.typography}
    index={context.itemIndex}
    isSelected={context.isSelected}
    isEditable={true}
    onSelect={context.onSelect}
    onTextChange={context.onTextChange}
    onEditingChange={(editing) => { /* notify parent */ }}
  />
)}
```

---

## 5. Technical Notes

### 5.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Props-driven | Data via props | No store coupling, reusable |
| Generic type | `<TSpread>` | Works with any spread structure |
| Render props | Context objects | Full control, extensible |
| Feature flags | Boolean props | Explicit capability control |
| Toolbars optional | Null if missing | Progressive enhancement |

### 5.2 Layout Constants

| Element | Value | Note |
|---------|-------|------|
| Header height | 48px | Fixed |
| Filmstrip height | 120px | Fixed when editor visible |
| Editor min height | 300px | Minimum |
| Filmstrip thumbnail | 100×80px | Fixed size |
| Grid thumbnail | Dynamic | Based on columns và container width |

### 5.3 State Persistence

Persist view preferences to localStorage với key `spread-view-prefs`:

```typescript
interface ViewPreferences {
  viewMode: ViewMode;                    // 'edit' | 'grid'
  zoomLevel: number;                     // 25-200
  columnsPerRow: number;                 // 1-6
}
```

**Defaults:** `viewMode: 'edit'`, `zoomLevel: 100`, `columnsPerRow: 4`

### 5.4 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `←` / `→` | Navigate prev/next spread |
| `Home` / `End` | First/last spread |
| `G` | Toggle view mode |
| `+` / `-` | Zoom in/out (edit) or adjust columns (grid) |
| `Delete` | Delete selected spread (if canDeleteSpread) |

### 5.5 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Filmstrip | `listbox` | `aria-label="Spread thumbnails"`, `aria-orientation="horizontal"` |
| Thumbnail | `option` | `aria-selected`, `aria-label="Spread {n}, pages {x}-{y}"` |
| Editor panel | `region` | `aria-label="Spread editor"`, `aria-live="polite"` |

---
