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
│                              SpreadViewHeader                                   │
│  ┌───────────────┐                                       ┌────────────────────┐ │
│  │  ViewToggle   │                                       │  DualPurposeSlider │ │
│  │  [⚏]         │                                       │  ─ ●────── + 100%  │ │
│  └───────────────┘                                       └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                            ManuscriptSpreadView (Parent)                           │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ Local State: viewMode, zoomLevel, columnsPerRow                              │  │
│  └────────────────────────────────┬─────────────────────────────────────────────┘  │
└───────────────────────────────────┼────────────────────────────────────────────────┘
                                    │ (props + callbacks)
                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadViewHeader                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: viewMode, zoomLevel, columnsPerRow, spreadViewMode                   │  │
│  │  Callbacks: onViewModeToggle, onZoomChange, onColumnsChange                  │  │
│  │  Local State: (none)                                                         │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                              │           │
│         ▼                                                              ▼           │
│  ┌──────────────────┐                                   ┌──────────────────────┐   │
│  │   ViewToggle     │                                   │  DualPurposeSlider   │   │
│  │                  │                                   │                      │   │
│  │ Props:           │                                   │ Props:               │   │
│  │ • viewMode       │                                   │ • viewMode           │   │
│  │                  │                                   │ • zoomLevel          │   │
│  │ Callbacks:       │                                   │ • columnsPerRow      │   │
│  │ • onToggle       │                                   │                      │   │
│  │                  │                                   │ Callbacks:           │   │
│  │                  │                                   │ • onZoomChange       │   │
│  │                  │                                   │ • onColumnsChange    │   │
│  └──────────────────┘                                   └──────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Lưu ý:** SpreadViewHeader là **stateless presentational component**. Tất cả state được quản lý bởi parent (ManuscriptSpreadView) và truyền xuống qua props.

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Header toolbar cho SpreadView với toggle button để switch giữa Edit và Grid modes. Slider có dual-purpose: zoom (edit mode) hoặc columns (grid mode).

**Shared Types:**

```typescript
type ViewMode = 'edit' | 'grid';
type SpreadViewMode = 'dummy' | 'finalize';
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SpreadViewHeaderProps {
  viewMode: ViewMode;                      // 'edit' | 'grid'
  zoomLevel: number;                       // 25-200, default 100 (edit mode only)
  columnsPerRow: number;                   // 1-6, default 4 (grid mode only)
  spreadViewMode: SpreadViewMode;          // 'dummy' | 'finalize'

  onViewModeToggle: () => void;
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}

// No local state - fully controlled component
```

**Store Integration:**

```typescript
// None - stateless presentational component
// All state managed by parent ManuscriptSpreadView
// Parent handles: viewMode, zoomLevel, columnsPerRow as local state
```

### 2.3 Render Logic (pseudo)

```
SpreadViewHeader:
  RENDER Container (flex row, justify-between, align-center, h-48px):

    // Left section
    RENDER ViewToggle với:
      - viewMode
      - onToggle: onViewModeToggle

    // Center section (flex-grow spacer)

    // Right section
    RENDER DualPurposeSlider với:
      - viewMode
      - zoomLevel (for edit mode)
      - columnsPerRow (for grid mode)
      - onZoomChange
      - onColumnsChange
```

### 2.4 Visual

**Edit Mode (default):**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ╔═══╗                                                   ┌─────────────────────┐ │
│  ║ ⚏ ║                                                   │ ─  ●──────  + 100%  │ │
│  ╚═══╝                                                   └─────────────────────┘ │
│    ↑ View toggle (tooltip: "Show spread grid view")         ↑ Zoom slider        │
│    (click to switch to grid)                                (25%-200%)           │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Grid Mode:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ╔═══╗                                                   ┌─────────────────────┐ │
│  ║ ⚏ ║                                                   │ ─  ●──────  +   4   │ │
│  ╚═══╝                                                   └─────────────────────┘ │
│    ↑ View toggle (tooltip: "Show full spread view")         ↑ Columns slider     │
│    (click to switch to edit)                                (1-6)                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Slider Behavior by Mode:**

| Mode | Slider Purpose | Range | Step | Label Format |
|------|----------------|-------|------|--------------|
| Edit | Zoom SpreadCanvas | 25%-200% | 25 | `{value}%` |
| Grid | Columns per row | 1-6 | 1 | `{value}` |

---

## 3. Child Components Interface

> **Lưu ý quan trọng:**
> - ViewToggle và DualPurposeSlider là **inline components** (không có file riêng)
> - Components đơn giản, không cần store integration
> - Thiết kế chi tiết ở đây vì sẽ implement inline

### 3.1 ViewToggle (inline)

**Mục đích:** Icon button để toggle giữa Edit và Grid view modes.

**Props & Callbacks:**

```typescript
interface ViewToggleProps {
  viewMode: ViewMode;          // 'edit' | 'grid'
  onToggle: () => void;
}
```

**Render Logic:**

```
ViewToggle:
  RENDER IconButton với:
    - icon: GridViewIcon (same icon for both modes)
    - tooltip: "Show full spread view"
    - aria-pressed: viewMode === 'grid'
    - aria-label: "Switch to {opposite} view"
    - onClick: onToggle
```

**Visual:**

```
Edit Mode:                              Grid Mode:
┌─────────────────────────────┐         ┌─────────────────────────────────┐
│  ╔═══╗  ← icon button       │         │  ╔═══╗  ← icon button           │
│  ║ ⚏ ║                      │        │  ║ ⚏ ║  (same icon)            │
│  ╚═══╝                      │         │  ╚═══╝                          │
│  Click → Grid               │         │  Click → Edit                   │
└─────────────────────────────┘         └─────────────────────────────────┘
```

### 3.2 DualPurposeSlider (inline)

**Mục đích:** Slider với behavior thay đổi dựa trên viewMode.

**Props & Callbacks:**

```typescript
interface DualPurposeSliderProps {
  viewMode: ViewMode;
  zoomLevel: number;                       // for edit mode
  columnsPerRow: number;                   // for grid mode
  onZoomChange: (level: number) => void;
  onColumnsChange: (columns: number) => void;
}
```

**Render Logic:**

```
DualPurposeSlider:
  IF viewMode === 'edit':
    config = { value: zoomLevel, min: 25, max: 200, step: 25, label: "{value}%", onChange: onZoomChange }
  ELSE:
    config = { value: columnsPerRow, min: 1, max: 6, step: 1, label: "{value}", onChange: onColumnsChange }

  RENDER Container (flex row, align-center, gap-8px):
    RENDER IconButton (minus) onClick: config.value - config.step
    RENDER Slider với config
    RENDER IconButton (plus) onClick: config.value + config.step
    RENDER Text label: config.label
```

**Config by Mode:**

| Mode | Value | Min | Max | Step | Label |
|------|-------|-----|-----|------|-------|
| Edit | zoomLevel | 25 | 200 | 25 | `{value}%` |
| Grid | columnsPerRow | 1 | 6 | 1 | `{value}` |

**Visual:**

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

## 4. Technical Notes

### 4.1 Key Design Decisions

**Stateless Presentational Component**
SpreadViewHeader không có local state. Tất cả UI state (viewMode, zoomLevel, columnsPerRow) được quản lý bởi parent ManuscriptSpreadView và truyền xuống qua props. Lý do: Single source of truth, dễ test, và parent cần state này cho các child components khác (SpreadEditorPanel, SpreadThumbnailList).

**Dual-Purpose Slider**
Slider changes behavior based on viewMode:
- Edit mode: Controls zoom (25%-200%) for SpreadCanvas
- Grid mode: Controls columns (1-6) for grid layout

Lý do: Space-efficient UI, cùng 1 slider phục vụ 2 mục đích khác nhau theo context.

**Inline Child Components**
ViewToggle và DualPurposeSlider được implement inline (không tách file riêng). Lý do: Components đơn giản, không có state phức tạp, không reuse ở nơi khác.

**Zoom Applies to SpreadCanvas Only**
Zoom level chỉ ảnh hưởng SpreadCanvas trong edit mode, không ảnh hưởng thumbnails. Lý do: Thumbnails cần consistent size để navigate.

### 4.2 Accessibility

| Element | Role | ARIA attributes |
|---------|------|-----------------|
| ViewToggle | `button` | `aria-pressed`, `aria-label="Switch to {mode} view"` |
| Slider | `slider` | `aria-label`, `aria-valuemin`, `aria-valuemax`, `aria-valuenow`, `aria-valuetext` |
| Decrease btn | `button` | `aria-label="Decrease {zoom/columns}"` |
| Increase btn | `button` | `aria-label="Increase {zoom/columns}"` |

### 4.3 Layout Constants

| Element | Value | Note |
|---------|-------|------|
| Header height | 48px | Fixed, matches parent layout |
| Toggle button | 36×36px | IconButton size |
| Slider width | 120px | Slider track only |
| Gap between controls | 8px | Consistent spacing |

---

