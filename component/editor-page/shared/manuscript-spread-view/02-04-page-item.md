# PageItem: Component Design

> **Note:** Special item type for page backgrounds - renders beneath all content with lowest z-index. Provides toolbar for page properties when selected.

**Screenshots:**
- Double page spread toolbar: `component/editor-page/screenshots/double-page-spread-toolbar.png`
- Single page toolbar: `component/editor-page/screenshots/single-page-toolbar.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SpreadEditorPanel<TSpread>                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Canvas Container                         │  │
│  │  ┌─────────────────────────┬─────────────────────────┐        │  │
│  │  │     PageItem (L)        │     PageItem (R)        │        │  │
│  │  │  (z-index: -999)        │  (z-index: -999)        │        │  │
│  │  │  ┌───────────────────┐  │  ┌───────────────────┐  │        │  │
│  │  │  │ background:       │  │  │ background:       │  │        │  │
│  │  │  │  color + texture  │  │  │  color + texture  │  │        │  │
│  │  │  └───────────────────┘  │  └───────────────────┘  │        │  │
│  │  │          2              │          3              │        │  │
│  │  └─────────────────────────┴─────────────────────────┘        │  │
│  │         ↑ Lowest z-index layer (renders first)                │  │
│  │                                                               │  │
│  │  ┌───────────────────────┬───────────────────────┐            │  │
│  │  │ Content Items         │ Content Items         │            │  │
│  │  │ (images, textboxes)   │ (images, textboxes)   │            │  │
│  │  │ (z-index: 0+)         │ (z-index: 0+)         │            │  │
│  │  └───────────────────────┴───────────────────────┘            │  │
│  │         ↑ Content layer (renders on top)                      │  │
│  │                                                               │  │
│  │  ╔═══════════════════════════════════════════════════════╗    │  │
│  │  ║           SelectionFrame (when page selected)         ║    │  │
│  │  ╚═══════════════════════════════════════════════════════╝    │  │
│  │                                                               │  │
│  │  ┌───────────────────────────────────────────────────────┐    │  │
│  │  │ PageToolbar (if renderPageToolbar provided)           │    │  │
│  │  │ [Layout ▾] [Color ■ #FFFFFF] [Texture ▾]            │    │  │
│  │  └───────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
SpreadEditorPanel
       │
       │ reads spread.pages[]
       ▼
┌──────────────────────────────────────────────────────────────┐
│  For each page in spread.pages[]:                            │
│                                                              │
│  1. Render PageItem at z-index: -999                         │
│  2. Apply background.color + background.texture              │
│  3. If renderPageToolbar exists:                             │
│     • Make page selectable (click handler)                   │
│     • Show SelectionFrame when selected                      │
│     • Show PageToolbar when selected                         │
│  4. Else:                                                    │
│     • Page is not selectable (no click handler)              │
│     • SelectionFrame never shown                             │
│     • Toolbar never shown                                    │
│                                                              │
│  5. Layout constraints:                                      │
│     • If pages[].layout is set → LOCKED                      │
│     • User cannot change layout once applied                 │
│     • Color + texture always editable                        │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Render page backgrounds with lowest z-index. Optional toolbar for page properties.

**Key Characteristics:**

| Feature | Behavior |
|---------|----------|
| **Z-index** | Always -999 (lowest layer) |
| **Selectability** | Only if `renderPageToolbar` provided |
| **Layout locking** | Once set, cannot be changed |
| **Color/Texture** | Always editable (if selectable) |
| **DPS detection** | Single page in pages[] = DPS, two pages = non-DPS |

**Shared Types:**

```typescript
interface PageData {
  number: string | number;  // DPS: "0-1" | non-DPS: 0, 1
  type: 'normal_page' | 'front_matter' | 'back_matter' | 'dedication';
  layout: string | null;    // UUID FK → template_layouts (null if not set)
  background: {
    color: string;          // hex color
    texture: string | null; // texture key or null
  };
}

type LayoutOption = {
  id: string;              // UUID
  title: string;           // Display name
  thumbnail_url: string;   // Preview image
  type: 1 | 2;            // 1: double page, 2: single page
};

type TextureOption = 'paper' | 'canvas' | 'linen' | 'watercolor' | null;
```

### 2.2 Interface

**Props:**

```typescript
interface PageItemProps<TSpread extends BaseSpread> {
  // === Data ===
  page: PageData;
  pageIndex: number;        // Index in spread.pages[]
  spread: TSpread;
  spreadId: string;

  // === Page position ===
  position: 'left' | 'right' | 'single';  // DPS: left/right, non-DPS: single

  // === Selection ===
  isSelected: boolean;
  onSelect?: () => void;    // Only provided if renderPageToolbar exists

  // === Callbacks ===
  onUpdatePage: (updates: Partial<PageData>) => void;

  // === Toolbar render (optional) ===
  renderPageToolbar?: (context: PageToolbarContext<TSpread>) => ReactNode;

  // === Layout config ===
  availableLayouts: LayoutOption[];  // Filtered by spread type (DPS vs non-DPS)
}
```

**Local State:**

```typescript
// PageItem has NO local state
// All state managed by parent (SpreadEditorPanel)
```

**Store Integration:**

```typescript
// PageItem is a pure presentational component
// No direct store access - all data flows through props from SpreadEditorPanel
// Parent (SpreadEditorPanel) handles store interactions via:
//   - useSnapshotSelectors() for reading spread data
//   - useSnapshotActions() for updating page properties
```

#### 2.2.1 Toolbar Context

Consumer receives full context when rendering toolbar:

```typescript
interface PageToolbarContext<TSpread> {
  // Page data
  page: PageData;
  pageIndex: number;
  position: 'left' | 'right' | 'single';

  // Spread context
  spread: TSpread;
  spreadId: string;

  // Selection
  isSelected: boolean;

  // Callbacks
  onUpdateLayout: (layoutId: string) => void;     // ⚡ Locked if page.layout !== null
  onUpdateColor: (color: string) => void;
  onUpdateTexture: (texture: TextureOption) => void;

  // Config
  availableLayouts: LayoutOption[];
  availableTextures: TextureOption[];

  // State flags
  isLayoutLocked: boolean;  // = page.layout !== null
}
```

### 2.3 Render Logic (pseudo)

```
PageItem<TSpread>:
  // Extract page properties
  pageNumber = page.number
  backgroundColor = page.background.color
  textureKey = page.background.texture

  // Determine if selectable
  isSelectable = renderPageToolbar !== undefined

  // Compute z-index (always lowest)
  zIndex = -999

  // Click handler (only if selectable)
  handleClick = isSelectable ? () => onSelect?.() : undefined

  RENDER PageContainer với:
    - style:
        position: absolute
        z-index: -999
        inset: 0 (full page coverage)
        background-color: backgroundColor
        background-image: textureKey ? `url(/textures/${textureKey}.png)` : none
        pointer-events: isSelectable ? 'auto' : 'none'
        cursor: isSelectable ? 'pointer' : 'default'

    - onClick: handleClick
    - data-page-index: pageIndex
    - data-page-number: pageNumber

  // Overlay when selected
  IF isSelected AND renderPageToolbar:
    RENDER SelectionFrame với:
      - geometry: { x: 0, y: 0, width: 100, height: 100 }
      - canResize: false
      - canDrag: false
      - showHandles: false  // Page is not resizable/draggable

    RENDER Toolbar:
      context = buildToolbarContext()
      renderPageToolbar(context)

  // Build context
  buildToolbarContext():
    return {
      page,
      pageIndex,
      position,
      spread,
      spreadId,
      isSelected,
      onUpdateLayout: (id) => {
        IF page.layout !== null:
          showWarning("Layout is locked and cannot be changed")
          RETURN
        onUpdatePage({ layout: id })
      },
      onUpdateColor: (color) => onUpdatePage({
        background: { ...page.background, color }
      }),
      onUpdateTexture: (texture) => onUpdatePage({
        background: { ...page.background, texture }
      }),
      availableLayouts: availableLayouts.filter(l =>
        spread.pages.length === 1 ? l.type === 1 : l.type === 2
      ),
      availableTextures: ['paper', 'canvas', 'linen', 'watercolor', null],
      isLayoutLocked: page.layout !== null
    }
```

### 2.4 Visual

**Normal State (not selected):**

```
┌─────────────────────────────────────────────┐
│                                             │
│  ╔═══════════════════════════════════════╗  │
│  ║ Page Background (z-index: -999)       ║  │
│  ║   color: #FFFFFF                    ║  │
│  ║   texture: paper (optional)           ║  │
│  ║                                       ║  │
│  ║   [Content items render on top]       ║  │
│  ╚═══════════════════════════════════════╝  │
│               Page Number: 2                │
└─────────────────────────────────────────────┘
     ↑ No click interaction if renderPageToolbar missing
```

**Selected State (DPS):**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Manuscript                                  │
│                                                                     │
│  Layout   [Default        ▾]                                        │
│  Color    [■ #FFFFFF     ]                                        │
│  Texture  [None           ▾]                                        │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  ╔═══════════════════════════════════════════════════════════════╗  │
│  ║                     Selection Frame                           ║  │
│  ║  ┌──────────────────────┬──────────────────────┐              ║  │
│  ║  │  Left Page (BG)      │  Right Page (BG)     │              ║  │
│  ║  │  color: #FFFFFF    │  color: #FFFFFF    │              ║  │
│  ║  │  texture: None       │  texture: None       │              ║  │
│  ║  │                      │                      │              ║  │
│  ║  │  [Photo]             │  [Text placeholder]  │              ║  │
│  ║  │                      │                      │              ║  │
│  ║  │         2            │         3            │              ║  │
│  ║  └──────────────────────┴──────────────────────┘              ║  │
│  ╚═══════════════════════════════════════════════════════════════╝  │
│                      ↑ Blue border (selected page)                  │
└─────────────────────────────────────────────────────────────────────┘
     ⚠ Layout locked once set (dropdown disabled)
```

**Selected State (non-DPS):**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Illustration    >  Retouch                  │
│                                                                     │
│  Layout   [Default        ▾]  ⚠ Once set, cannot change             │
│  Color    [■ #FFFFFF     ]                                        │
│  Texture  [None           ▾]                                        │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  ╔═══════════════════════════════════╗                              │
│  ║      Selection Frame              ║                              │
│  ║  ┌──────────────────────┐         ║                              │
│  ║  │  Single Page (BG)    │         ║                              │
│  ║  │  color: #FFFFFF    │         ║    [No selection on          │
│  ║  │  texture: None       │         ║     right page - grayed]     │
│  ║  │                      │         ║                              │
│  ║  │  [Illustration]      │         ║                              │
│  ║  │                      │         ║                              │
│  ║  │         10           │    11   ║                              │
│  ║  └──────────────────────┘         ║                              │
│  ╚═══════════════════════════════════╝                              │
│       ↑ Blue border on left page only                               │
│       ↑ Central spine separator visible                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Note:** PageItem is a **leaf component** with no child components. All rendering is handled internally using CSS for background colors and textures.

✅ **No child components** - PageItem directly renders page background layer. Selection overlay and toolbar are provided by parent (SpreadEditorPanel) when page is selected.

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Z-Index Strategy**

Page backgrounds always render at z-index -999 (lowest layer), ensuring all content items stack on top.

| Layer | Z-Index | Elements |
|-------|---------|----------|
| Page background | -999 | PageItem components |
| Images | 0-999 | Based on spread.images[].z-index |
| Textboxes | 1000-1999 | Based on spread.textboxes[].z-index |
| Objects | 2000-2999 | Based on spread.objects[].z-index |
| Selection overlay | 10000 | SelectionFrame |
| Toolbar | 10001 | Floating toolbar |

**Rationale:** Page always renders first (lowest z-index), content stacks on top.

**Conditional Selectability**

Page is selectable ONLY if `renderPageToolbar` prop is provided. Without toolbar, page selection serves no purpose.

**Rationale:** Clean API - toolbar presence determines selectability. Avoids unnecessary interaction states.

**Layout Immutability**

Once page layout is set (`page.layout !== null`), it cannot be changed without clearing all content first.

**Rationale:** Layout defines text/image slot structure. Changing layout would invalidate existing content placement, causing data corruption.

### 4.2 Layout Locking

**Rationale:**

Once a page layout is applied:
- Structure is committed (text/image slots created)
- Changing layout would invalidate existing content
- User must keep layout or delete all content first

**Behavior:**

1. User attempts to change layout when `page.layout !== null`
2. System shows warning notification
3. User options:
   - **Clear Content** → Unlock layout, delete all page content
   - **Cancel** → Keep current layout

### 4.3 DPS Detection

**Detection Logic:**

| Spread Type | `pages.length` | Position Mapping |
|-------------|----------------|------------------|
| **DPS** | 1 | `'single'` for all pages |
| **Non-DPS** | 2 | pageIndex 0 → `'left'`, pageIndex 1 → `'right'` |

### 4.4 Layout Filtering

**Filter Rule:**

| Spread Type | Layout Type Filter |
|-------------|-------------------|
| DPS (`pages.length === 1`) | `layout.type === 1` |
| Non-DPS (`pages.length === 2`) | `layout.type === 2` |

**Validation:** Show error if layout type doesn't match spread type.

### 4.5 Texture Assets

**Asset Structure:**

```
public/textures/
├── paper.png       # 1024x1024, seamless
├── canvas.png      # 1024x1024, seamless
├── linen.png       # 1024x1024, seamless
└── watercolor.png  # 1024x1024, seamless
```

**Rendering:** Tiled background with 256px repeat, applied via `background-image` CSS property.

### 4.6 Integration with SpreadEditorPanel

**Updated Props:**

Add page-related props to `SpreadEditorPanelProps`:

```typescript
interface SpreadEditorPanelProps<TSpread extends BaseSpread> {
  // ... existing props ...

  // === Page rendering (optional) ===
  renderPageToolbar?: (context: PageToolbarContext<TSpread>) => ReactNode;

  // === Page callbacks ===
  onUpdatePage?: (pageIndex: number, updates: Partial<PageData>) => void;

  // === Layout config ===
  availableLayouts?: LayoutOption[];  // Template layouts from DB
}
```

**Updated State:**

Add page selection to `SpreadEditorPanelState`:

```typescript
interface SpreadEditorPanelState {
  // Selection
  selectedElement: SelectedElement | null;
  selectedPageIndex: number | null;  // NEW: Page selection (mutually exclusive with element)
  isTextboxEditing: boolean;

  // ... rest unchanged ...
}
```

**Updated SelectedElement Type:**

```typescript
type SelectedElementType = 'image' | 'textbox' | 'object' | 'animation' | 'page';

interface SelectedElement {
  type: SelectedElementType;
  index: number;  // pageIndex if type === 'page'
}
```

**Rendering Order:**

Updated render order in SpreadEditorPanel:

```
RENDER SpreadEditorPanel:
  1. Render PageItems first (z-index: -999)
     FOR each page in spread.pages[]:
       RENDER PageItem với:
         - page data
         - onSelect (if renderPageToolbar)
         - renderPageToolbar

  2. Render content items (z-index: 0+)
     FOR each item in renderItems:
       RENDER based on item type

  3. Render SelectionFrame overlay
     IF selectedElement.type === 'page':
       RENDER full page selection frame
     ELSE IF selectedElement !== null:
       RENDER element selection frame

  4. Render Toolbar
     IF selectedElement.type === 'page' AND renderPageToolbar:
       RENDER page toolbar
     ELSE IF selectedElement !== null AND render*Toolbar:
       RENDER item toolbar
```

**Selection Handling:**

Updated selection logic:

```typescript
handleElementSelect(element: SelectedElement | null):
  // Mutual exclusion: page vs element
  IF element?.type === 'page':
    setSelectedPageIndex(element.index)
    setSelectedElement(null)
  ELSE:
    setSelectedElement(element)
    setSelectedPageIndex(null)

handlePageClick(pageIndex: number):
  IF renderPageToolbar:
    handleElementSelect({ type: 'page', index: pageIndex })

handleCanvasClick(e):
  IF e.target === canvasRef.current:
    handleElementSelect(null)  // Deselect both page and element
    setSelectedPageIndex(null)
```

### 4.7 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| PageItem | `button` (if selectable) | `aria-label="Page {number}, layout {name}"` |
| PageItem | `presentation` (if not selectable) | `aria-hidden="true"` |
| PageToolbar | `toolbar` | `aria-label="Page properties"`, `aria-controls="page-{index}"` |
| Layout dropdown | `combobox` | `aria-disabled="true"` if locked |

### 4.8 Usage Patterns

**Pattern 1: With Page Toolbar (Selectable)**

Consumer passes `renderPageToolbar` prop to enable page selection and toolbar rendering. Toolbar receives `PageToolbarContext` with page data, callbacks, and layout options.

**Pattern 2: Without Page Toolbar (Non-selectable)**

Consumer omits `renderPageToolbar` prop. Pages render as pure backgrounds without click handlers or selection states.

---
