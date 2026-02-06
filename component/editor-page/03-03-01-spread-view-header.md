# SpreadViewHeader: Component Design

> **Note:** Renamed from `GridHeader`. Now includes toggle button for editor visibility and mode-specific actions.

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SpreadViewHeader                                    │
│  ┌───────────────┐                                       ┌────────────────────┐ │
│  │ EditorToggle  │                                       │   ZoomControls     │ │
│  │  ☐            │                                       │  ─ ●────── + 100%  │ │
│  └───────────────┘                                       └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Layout

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ☐                                                        ─ ●────────── + 100%  │
│  └→ Toggle                                                └→ Zoom controls     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

> **Note:** Translation is handled at EditorPage level via `TranslationNotAvailableDialog`.
> See [01-04-translation-not-available-dialog.md](component/editor-page/01-04-translation-not-available-dialog.md).

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Header toolbar cho SpreadView với toggle button để switch giữa Editor+Filmstrip và Grid layouts, và zoom controls.

### 2.2 Interface

```typescript
interface SpreadViewHeaderProps {
  isEditorVisible: boolean;
  zoomLevel: number;                     // 50-200, default 100
  mode: SpreadViewMode;

  onToggleEditor: () => void;
  onZoomChange: (level: number) => void;
}
```

### 2.3 Render Logic (pseudo)

```
SpreadViewHeader:
  RENDER Container (flex row, justify-between, align-center):

    // Left section
    RENDER EditorToggle với:
      - isChecked: !isEditorVisible  // Unchecked = editor visible
      - tooltip: isEditorVisible ? "Show slider only" : "Show editor"
      - onChange: onToggleEditor

    // Center section (spacer)

    // Right section
    RENDER ZoomControls với:
      - value: zoomLevel
      - min: 50
      - max: 200
      - step: 10
      - onChange: onZoomChange
```

### 2.4 Visual

**Full Layout:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ╔═══╗                                                                          │
│  ║ ☐ ║  "Show slider only"                                                      │
│  ╚═══╝                                                                          │
│    ↑ Toggle button with tooltip                                                 │
│                                                                                 │
│                                                         ┌─────────────────────┐ │
│                                                         │ ─  ●──────  +  100% │ │
│                                                         └─────────────────────┘ │
│                                                           ↑ Zoom slider          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Toggle Button States:**

```
Editor Visible (default):           Grid Only (toggled):
┌───────────────────────┐           ┌───────────────────────┐
│  ☐                    │           │  ☑                    │
│  └→ unchecked         │           │  └→ checked           │
│                       │           │                       │
│  Tooltip:             │           │  Tooltip:             │
│  "Show slider only"   │           │  "Show editor"        │
└───────────────────────┘           └───────────────────────┘

Visual:                              Visual:
  ┌─────┐                              ┌─────┐
  │ ☐☐☐ │  ← grid icon                │ ☑☐☐ │  ← filmstrip icon
  │ ☐☐☐ │     (current view)          │     │     (current view)
  └─────┘                              └─────┘
```

**Zoom Controls:**

```
┌───────────────────────────────────────────────────────┐
│  ─                    ●                    +    100%  │
│  ↑                    ↑                    ↑     ↑    │
│  decrease         slider              increase  value │
│  (-10)            (50-200)             (+10)          │
└───────────────────────────────────────────────────────┘

States:
- Min (50%):  [disabled ─] ●------ [+] 50%
- Default:    [─] -------●------- [+] 100%
- Max (200%): [─] ------● [disabled +] 200%
```

---

## 3. Child Components Interface

### 3.1 EditorToggle

**Mục đích:** Toggle button để switch giữa Editor+Filmstrip và Grid layouts.

**Props:**

```typescript
interface EditorToggleProps {
  isChecked: boolean;                    // true = grid only, false = editor visible
  tooltip: string;
  onChange: () => void;
}
```

**Visual:**

```
┌─────────────────┐
│  ╔═══╗          │
│  ║ ☐ ║  ← icon  │
│  ╚═══╝          │
│                 │
│  Hover:         │
│  ┌───────────┐  │
│  │ Show      │  │
│  │ slider    │  │
│  │ only      │  │
│  └───────────┘  │
│    ↑ tooltip    │
└─────────────────┘
```

### 3.2 ZoomControls

**Mục đích:** Slider + buttons để control zoom level.

**Props:**

```typescript
interface ZoomControlsProps {
  value: number;
  min: number;
  max: number;
  step: number;
  onChange: (value: number) => void;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Toggle Icon Design**
Icon thay đổi dựa trên current state:
- Editor visible: Grid icon (gợi ý "click để switch sang grid")
- Grid only: Filmstrip icon (gợi ý "click để switch sang editor")

**Zoom Applies to Editor Only**
Zoom level chỉ ảnh hưởng SpreadEditorPanel, không ảnh hưởng Filmstrip/Grid. Lý do: Filmstrip cần consistent size để navigate, Editor cần zoom để edit detail.

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

.editor-toggle {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}

.zoom-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.zoom-slider {
  width: 120px;
}

.zoom-value {
  min-width: 40px;
  text-align: right;
  font-variant-numeric: tabular-nums;
}
```

### 4.3 Accessibility

```typescript
const toggleA11y = {
  role: 'switch',
  'aria-checked': !isEditorVisible,
  'aria-label': isEditorVisible ? 'Show slider only' : 'Show editor',
};

const zoomSliderA11y = {
  role: 'slider',
  'aria-label': 'Zoom level',
  'aria-valuemin': 50,
  'aria-valuemax': 200,
  'aria-valuenow': zoomLevel,
  'aria-valuetext': `${zoomLevel}%`,
};
```
