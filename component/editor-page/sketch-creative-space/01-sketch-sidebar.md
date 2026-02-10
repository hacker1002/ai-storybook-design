# SketchSidebar: Component Design

> **Parent:** [SketchCreativeSpace](./README.md)

**Screenshot:** `screenshots/manuscript-sketch-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               SketchSidebar                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SidebarHeader                                                           │   │
│  │  "Sketch"                                                         [⚏]    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SketchTypeList (accordion)                                              │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ SketchTypeItem (Characters) - EXPANDED                             │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ Header: ◎ Characters                                       [∨] │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ PromptPanel                                                    │ │  │   │
│  │  │ │   PROMPT: [Enter your prompt for character sheets...]          │ │  │   │
│  │  │ │   [✨ Generate]                                                │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ SketchTypeItem (Props) - COLLAPSED                             [>] │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ SketchTypeItem (Spreads) - COLLAPSED                           [>] │  │   │
│  │  │ (no PromptPanel - info text only)                                  │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
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
│                              SketchSidebar                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: activeSketchType, onSketchTypeChange                                │  │
│  │  Local State: expandedSketchType, promptInputs, isGenerating                │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                         │
│         ▼                                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────┐    │
│  │                        SketchTypeItem × 3                                 │    │
│  │  Props:                                                                   │    │
│  │  • sketchType: SketchType                                                 │    │
│  │  • isActive: boolean                                                      │    │
│  │  • isExpanded: boolean                                                    │    │
│  │  • hasPromptPanel: boolean                                                │    │
│  │  • promptInput: string                                                    │    │
│  │  • isGenerating: boolean                                                  │    │
│  │  Callbacks:                                                               │    │
│  │  • onToggle: () => void                                                   │    │
│  │  • onPromptChange: (value: string) => void                                │    │
│  │  • onGenerate: () => void                                                 │    │
│  └───────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Left sidebar với accordion tabs for Characters, Props, Spreads. PromptPanel for Characters/Props only (Spreads uses existing data).

**Shared Types:**

```typescript
type SketchType = 'characters' | 'props' | 'spreads';

interface SketchTypeConfig {
  type: SketchType;
  icon: string;           // Lucide icon name
  label: string;
  hasPromptPanel: boolean;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SketchSidebarProps {
  activeSketchType: SketchType;
  onSketchTypeChange: (type: SketchType) => void;
}

interface SketchSidebarState {
  expandedSketchType: SketchType | null;           // Which accordion is expanded
  promptInputs: Record<SketchType, string>;        // Prompt per type (only for characters/props)
  isGenerating: Record<SketchType, boolean>;       // Loading state per type
}
```

**Store Integration:**

```typescript
// No direct store access - pure presentational with callbacks
// Generate action will be handled by parent via callback or separate hook
```

### 2.3 Configuration

```typescript
const SKETCH_TYPES: SketchTypeConfig[] = [
  { type: 'characters', icon: 'CircleDot',  label: 'Characters', hasPromptPanel: true },
  { type: 'props',      icon: 'CircleDot',  label: 'Props',      hasPromptPanel: true },
  { type: 'spreads',    icon: 'LayoutGrid', label: 'Spreads',    hasPromptPanel: false },
];
```

### 2.4 Render Logic (pseudo)

```
SketchSidebar:
  // Props from parent
  { activeSketchType, onSketchTypeChange } = props

  // Local state
  [expandedSketchType, setExpandedSketchType] = useState(activeSketchType)
  [promptInputs, setPromptInputs] = useState({ characters: '', props: '', spreads: '' })
  [isGenerating, setIsGenerating] = useState({ characters: false, props: false, spreads: false })

  handleToggle(sketchType):
    IF expandedSketchType === sketchType:
      setExpandedSketchType(null)
    ELSE:
      setExpandedSketchType(sketchType)
      onSketchTypeChange(sketchType)  // Also set active when expanding

  handlePromptChange(sketchType, value):
    setPromptInputs({ ...promptInputs, [sketchType]: value })

  handleGenerate(sketchType):
    setIsGenerating({ ...isGenerating, [sketchType]: true })
    // Call generate API (via hook or callback)
    // On complete: setIsGenerating({ ...isGenerating, [sketchType]: false })

  RENDER Container (flex column):

    // Header
    RENDER SidebarHeader:
      - title: "Sketch"
      - infoButton: [⚏] (optional info tooltip)

    // Accordion tabs
    FOR EACH config IN SKETCH_TYPES:
      isActive = activeSketchType === config.type
      isExpanded = expandedSketchType === config.type

      RENDER SketchTypeItem với:
        - sketchType: config.type
        - icon: config.icon
        - label: config.label
        - hasPromptPanel: config.hasPromptPanel
        - isActive
        - isExpanded
        - promptInput: promptInputs[config.type]
        - isGenerating: isGenerating[config.type]
        - onToggle: () => handleToggle(config.type)
        - onPromptChange: (v) => handlePromptChange(config.type, v)
        - onGenerate: () => handleGenerate(config.type)
```

### 2.5 Visual

**Characters Expanded (with PromptPanel):**

```
┌────────────────────────────────┐
│ Sketch                    [⚏]  │
│ ┌────────────────────────────┐ │
│ │ ◎ Characters             ∨ │ │  ← Expanded, Active
│ │ ────────────────────────── │ │
│ │ PROMPT                     │ │
│ │ ┌────────────────────────┐ │ │
│ │ │ Describe the character │ │ │
│ │ │ sheets you want...     │ │ │
│ │ └────────────────────────┘ │ │
│ │ ┌────────────────────────┐ │ │
│ │ │     ✨ Generate        │ │ │
│ │ └────────────────────────┘ │ │
│ └────────────────────────────┘ │
│ ◎ Props                    >   │  ← Collapsed
│ ▣ Spreads                  >   │  ← Collapsed
└────────────────────────────────┘
```

**Props Expanded (with PromptPanel):**

```
┌────────────────────────────────┐
│ Sketch                    [⚏]  │
│ ◎ Characters               >   │  ← Collapsed
│ ┌────────────────────────────┐ │
│ │ ◎ Props                 ∨  │ │  ← Expanded, Active
│ │ ────────────────────────── │ │
│ │ PROMPT                     │ │
│ │ ┌────────────────────────┐ │ │
│ │ │ Describe the prop      │ │ │
│ │ │ sheets you want...     │ │ │
│ │ └────────────────────────┘ │ │
│ │ ┌────────────────────────┐ │ │
│ │ │     ✨ Generate        │ │ │
│ │ └────────────────────────┘ │ │
│ └────────────────────────────┘ │
│ ▣ Spreads                  >   │  ← Collapsed
└────────────────────────────────┘
```

**Spreads Expanded (NO PromptPanel):**

```
┌────────────────────────────────┐
│ Sketch                    [⚏]  │
│ ◎ Characters               >   │  ← Collapsed
│ ◎ Props                    >   │  ← Collapsed
│ ┌────────────────────────────┐ │
│ │ ▣ Spreads               ∨  │ │  ← Expanded, Active
│ │ ────────────────────────── │ │
│ │                            │ │
│ │  View and edit your final  │ │
│ │  spreads from the snapshot │ │
│ │                            │ │
│ └────────────────────────────┘ │
└────────────────────────────────┘
```

**All Collapsed:**

```
┌────────────────────────────────┐
│ Sketch                    [⚏]  │
│ ◎ Characters               >   │
│ ◎ Props                    >   │
│ ▣ Spreads                  >   │
└────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks**.
> Sub-components là elements, không cần file riêng.

### 3.1 SidebarHeader

**Mục đích:** Header với title và optional info button.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Title | `<span>` | "Sketch" |
| Info button | `<button>` | Optional tooltip with info about sketch workflow |

### 3.2 SketchTypeItem

**Mục đích:** Accordion item for a sketch type. Expandable with optional PromptPanel.

**Props & Callbacks:**

```typescript
interface SketchTypeItemProps {
  sketchType: SketchType;
  icon: string;
  label: string;
  hasPromptPanel: boolean;
  isActive: boolean;
  isExpanded: boolean;
  promptInput: string;
  isGenerating: boolean;
  onToggle: () => void;
  onPromptChange: (value: string) => void;
  onGenerate: () => void;
}
```

**Visual:**

```
Collapsed:
┌────────────────────────────────────────────────────┐
│ ◎ Characters                                     > │
└────────────────────────────────────────────────────┘

Expanded (with PromptPanel - Characters/Props):
┌────────────────────────────────────────────────────┐
│ ◎ Characters                                     ∨ │
├────────────────────────────────────────────────────┤
│ PROMPT                                             │
│ ┌────────────────────────────────────────────────┐ │
│ │ Enter your prompt...                           │ │
│ └────────────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────────────┐ │
│ │            ✨ Generate                         │ │
│ └────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘

Expanded (without PromptPanel - Spreads):
┌────────────────────────────────────────────────────┐
│ ▣ Spreads                                        ∨ │
├────────────────────────────────────────────────────┤
│                                                    │
│   View and edit your final spreads                 │
│   from the snapshot.                               │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 3.3 PromptPanel

**Mục đích:** Prompt textarea with generate button. Used for Characters and Props only.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Prompt label | `<label>` | "PROMPT" |
| Prompt textarea | `<textarea>` | Multi-line input |
| Generate button | `<button>` | Primary action |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Spreads Has No PromptPanel**
Spreads tab shows existing `snapshot.spreads` data - no generation action needed here. User edits spreads directly in ManuscriptSpreadView.

**Accordion Exclusivity**
Same as DocSidebar: only one tab expanded at a time. Expanding also sets active.

**Icon Differentiation**
- Characters/Props: ◎ (CircleDot) - represents reference sheets
- Spreads: ▣ (LayoutGrid) - represents page layouts

### 4.2 Generate Workflow

| Type | Input | Output |
|------|-------|--------|
| Characters | Prompt + Dummy spreads | Character reference sheets → `sketch.character_sheets[]` |
| Props | Prompt + Dummy spreads | Prop reference sheets → `sketch.prop_sheets[]` |
| Spreads | N/A | Uses existing `snapshot.spreads` |

### 4.3 Layout Constants

| Element | Value |
|---------|-------|
| Sidebar width | 280px |
| Tab header height | 44px |
| PromptPanel padding | 12px |
| Textarea min-height | 80px |
| Generate button height | 36px |

### 4.4 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Container | `navigation` | `aria-label="Sketch sidebar"` |
| SketchTypeList | `tablist` | `aria-orientation="vertical"` |
| SketchTypeItem header | `tab` | `aria-selected`, `aria-expanded` |
| SketchTypeItem panel | `tabpanel` | `aria-labelledby` |
| Generate button | `button` | `aria-busy` when loading |

### 4.5 Keyboard Navigation

| Key | Action |
|-----|--------|
| `Tab` | Move between tabs |
| `Enter` / `Space` | Toggle expanded |
| `Arrow Up/Down` | Navigate tabs |

---
