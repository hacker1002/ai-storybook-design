# SpreadViewHeader: Component Design

> **Note:** Header component với toggle button và dual-purpose slider (zoom/columns).

**Screenshots:**
- Edit mode: `component/editor-page/screenshots/manuscript-edit-view.png`
- Grid mode: `component/editor-page/screenshots/manuscript-grid-view.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadViewHeader                                    │
│  ┌───────────────┐                                       ┌────────────────────┐ │
│  │  ViewToggle   │                                       │  DualPurposeSlider │ │
│  │  [☐] ⚏       │                                       │  ─ ●────── + 100%  │ │
│  └───────────────┘                                       └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Layout

**Edit Mode:**
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                      ─ ●────────── + 100%  │
│   ↑ icon only                                             └→ Zoom (25%-200%)   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Grid Mode:**
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  [⚏]  ☑ Show full spread view                             ─ ●────────── + 4    │
│   ↑ icon  ↑ checkbox                                      └→ Columns (1-6)     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

> **Note:** Translation is handled at EditorPage level via `TranslationNotAvailableDialog`.
> See [01-04-translation-not-available-dialog.md](component/editor-page/01-04-translation-not-available-dialog.md).

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Header toolbar cho SpreadView với toggle button để switch giữa Edit và Grid modes. Slider có dual-purpose: zoom (edit mode) hoặc columns (grid mode).

### 2.2 Interface

```typescript
type ViewMode = 'edit' | 'grid';

interface SpreadViewHeaderProps {
  viewMode: ViewMode;
  zoomLevel: number;                     // 25-200, default 100 (edit mode only)
  columnsPerRow: number;                 // 1-6, default 4 (grid mode only)
  spreadViewMode: SpreadViewMode;        // 'dummy' | 'finalize'

  onViewModeToggle: () => void;
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}
```

### 2.3 Render Logic (pseudo)

```
SpreadViewHeader:
  RENDER Container (flex row, justify-between, align-center):

    // Left section
    RENDER ViewToggle icon button
    IF viewMode === 'grid':
      RENDER Checkbox "Show full spread view" (checked):
        - onChange: onViewModeToggle

    // Center section (spacer)

    // Right section - Dual Purpose Slider
    IF viewMode === 'edit':
      RENDER ZoomSlider với:
        - value: zoomLevel
        - min: 25
        - max: 200
        - step: 25
        - label: "{zoomLevel}%"
        - onChange: onZoomChange
    ELSE (grid mode):
      RENDER ColumnsSlider với:
        - value: columnsPerRow
        - min: 1
        - max: 6
        - step: 1
        - label: "{columnsPerRow}"
        - onChange: onColumnsChange
```

### 2.4 Visual

**Edit Mode (default):**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ╔═══╗                                                   ┌─────────────────────┐ │
│  ║ ⚏ ║                                                   │ ─  ●──────  + 100%  │ │
│  ╚═══╝                                                   └─────────────────────┘ │
│    ↑ View toggle icon                                       ↑ Zoom slider        │
│    (click to switch to grid)                                (25%-200%)           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Grid Mode:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ╔═══╗  ☑ Show full spread view                          ┌─────────────────────┐ │
│  ║ ⚏ ║                                                   │ ─  ●──────  +   4   │ │
│  ╚═══╝                                                   └─────────────────────┘ │
│    ↑ View toggle   ↑ Checkbox (uncheck to exit grid)        ↑ Columns slider     │
│                                                             (1-6)                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Slider Behavior by Mode:**

| Mode | Slider Purpose | Range | Step | Label Format |
|------|----------------|-------|------|--------------|
| Edit | Zoom SpreadCanvas | 25%-200% | 25 | `{value}%` |
| Grid | Columns per row | 1-6 | 1 | `{value}` |

**Slider Controls:**

```
Edit Mode (Zoom):
┌───────────────────────────────────────────────────────┐
│  ─                    ●                    +    100%  │
│  ↑                    ↑                    ↑     ↑    │
│  decrease         slider              increase  value │
│  (-25)            (25-200)             (+25)          │
└───────────────────────────────────────────────────────┘

Grid Mode (Columns):
┌───────────────────────────────────────────────────────┐
│  ─                    ●                    +      4   │
│  ↑                    ↑                    ↑     ↑    │
│  decrease         slider              increase  value │
│  (-1)              (1-6)                (+1)          │
└───────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

### 3.1 ViewToggle

**Mục đích:** Icon button + optional checkbox để switch giữa Edit và Grid modes.

**Props:**

```typescript
interface ViewToggleProps {
  viewMode: ViewMode;                    // 'edit' | 'grid'
  onToggle: () => void;
}
```

**Visual:**

```
Edit Mode:                              Grid Mode:
┌─────────────────┐                     ┌─────────────────────────────────┐
│  ╔═══╗          │                     │  ╔═══╗  ☑ Show full spread view │
│  ║ ⚏ ║  ← icon  │                     │  ║ ⚏ ║                          │
│  ╚═══╝          │                     │  ╚═══╝                          │
│                 │                     │    ↑     ↑ Checkbox visible     │
│  Click → Grid   │                     │  Click/Uncheck → Edit mode      │
└─────────────────┘                     └─────────────────────────────────┘
```

### 3.2 DualPurposeSlider

**Mục đích:** Slider với dual behavior dựa trên viewMode.

**Props:**

```typescript
interface DualPurposeSliderProps {
  viewMode: ViewMode;
  zoomLevel: number;                     // for edit mode
  columnsPerRow: number;                 // for grid mode
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}
```

**Config by Mode:**

```typescript
const SLIDER_CONFIG = {
  edit: {
    min: 25,
    max: 200,
    step: 25,
    getValue: (props) => props.zoomLevel,
    formatLabel: (value) => `${value}%`,
    onChange: (props) => props.onZoomChange,
  },
  grid: {
    min: 1,
    max: 6,
    step: 1,
    getValue: (props) => props.columnsPerRow,
    formatLabel: (value) => `${value}`,
    onChange: (props) => props.onColumnsChange,
  },
};
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Dual-Purpose Slider**
Slider changes behavior based on viewMode:
- Edit mode: Controls zoom (25%-200%) for SpreadCanvas
- Grid mode: Controls columns (1-6) for grid layout

Lý do: Space-efficient UI, cùng 1 slider phục vụ 2 mục đích khác nhau theo context.

**View Toggle with Checkbox**
- Edit mode: Only icon button visible
- Grid mode: Icon + "Show full spread view" checkbox

Lý do: Checkbox làm rõ đang ở grid mode, uncheck để quay về edit mode.

**Zoom Applies to SpreadCanvas Only**
Zoom level chỉ ảnh hưởng SpreadCanvas trong edit mode, không ảnh hưởng thumbnails. Lý do: Thumbnails cần consistent size để navigate.

> **Note:** Translation is handled at EditorPage level via `TranslationNotAvailableDialog`.
> See [01-04-translation-not-available-dialog.md](component/editor-page/01-04-translation-not-available-dialog.md).

### 4.2 Styling

```css
.spread-view-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 16px;
  background: var(--surface-secondary);
  border-bottom: 1px solid var(--border-color);
  height: 48px;
}

.view-toggle {
  display: flex;
  align-items: center;
  gap: 8px;
}

.view-toggle-checkbox {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 14px;
}

.dual-slider {
  display: flex;
  align-items: center;
  gap: 8px;
}

.dual-slider-track {
  width: 120px;
}

.dual-slider-value {
  min-width: 40px;
  text-align: right;
  font-variant-numeric: tabular-nums;
}
```

### 4.3 Accessibility

```typescript
const toggleA11y = (viewMode: ViewMode) => ({
  role: 'button',
  'aria-pressed': viewMode === 'grid',
  'aria-label': viewMode === 'edit' ? 'Switch to grid view' : 'Switch to edit view',
});

const sliderA11y = (viewMode: ViewMode, value: number) => ({
  role: 'slider',
  'aria-label': viewMode === 'edit' ? 'Zoom level' : 'Columns per row',
  'aria-valuemin': viewMode === 'edit' ? 25 : 1,
  'aria-valuemax': viewMode === 'edit' ? 200 : 6,
  'aria-valuenow': value,
  'aria-valuetext': viewMode === 'edit' ? `${value}%` : `${value} columns`,
});
```
