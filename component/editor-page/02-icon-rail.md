# IconRail: Component Design

**Screenshot:** `screenshots/icon-rail.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌────────────────────────────────────────────┐
│               IconRail                     │
│  ┌──────────────────────────────────────┐  │
│  │ IconRailItem - Manuscript (active)   │  │
│  │ IconRailItem - Characters            │  │
│  │ IconRailItem - Props                 │  │
│  │ IconRailItem - Stages                │  │
│  ├──────────────────────────────────────┤  │  ← Separator
│  │ IconRailItem - Spreads               │  │
│  │ IconRailItem - Objects               │  │
│  │ IconRailItem - Animations            │  │
│  ├──────────────────────────────────────┤  │  ← Separator
│  │ IconRailItem - Flags                 │  │
│  │ IconRailItem - Shares                │  │
│  │ IconRailItem - Collaborators         │  │
│  │ IconRailItem - Settings              │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
        ┌───────────────────────────┐          ┌───────────────────────┐
        │        EditorPage         │          │  EditorSettingsStore  │
        │   (parent component)      │          │   (Zustand global)    │
        │                           │          │                       │
        │ • activeCreativeSpace     │          │ • currentStep         │
        │ • onCreativeSpaceChange   │          └───────────┬───────────┘
        └─────────────┬─────────────┘                      │
                      │ (props)                            │ (selector)
                      ▼                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                             IconRail                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Props: activeCreativeSpace, onCreativeSpaceChange             │  │
│  │  Store: currentStep via useCurrentStep()                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│         │                                                            │
│    FOR EACH item IN ICON_RAIL_ITEMS:                                 │
│         │                                                            │
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                        IconRailItem                             │  │
│  │  Props:                                                         │  │
│  │  • item: IconRailItemConfig                                     │  │
│  │  • isActive: boolean                                            │  │
│  │  • isEnabled: boolean                                           │  │
│  │  Callback:                                                      │  │
│  │  • onClick: () => void                                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.3 Visual States (theo screenshot)

```
Default:                Active:                  Disabled:
┌────────────────┐      ┌────────────────┐       ┌────────────────┐
│                │      │ ┌────────────┐ │       │                │
│   ┌────────┐   │      │ │   Icon     │ │       │   ┌────────┐   │
│   │  Icon  │   │      │ │  (white)   │ │       │   │  Icon  │   │
│   └────────┘   │      │ └────────────┘ │       │   └────────┘   │
│                │      │  blue bg,      │       │  (grayed out)  │
└────────────────┘      │  rounded-lg    │       └────────────────┘
                        └────────────────┘
                              ↑                       ↑
                        filled bg (primary)     opacity: 0.4
                        white icon              cursor: not-allowed
```

### 1.4 Step → Enabled Items

| currentStep | Enabled CreativeSpaces |
|-------------|--------------------|
| `idea` | manuscript, flags, shares, collabs, config |
| `sketch` | + characters, props, stages, spreads |
| `illustration` | (same as sketch) |
| `retouch` | + objects, animations |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Sidebar navigation dọc bên trái Editor. Render 11 IconRailItem với 2 separators. Items enabled/disabled dựa trên `currentStep`.

**Shared Types:**

```typescript
type PipelineStep = 'idea' | 'sketch' | 'illustration' | 'retouch';

type CreativeSpaceType =
  | 'manuscript' | 'characters' | 'props' | 'stages' | 'spreads'
  | 'objects' | 'animations' | 'flags' | 'shares' | 'collabs' | 'config';

interface IconRailItemConfig {
  id: CreativeSpaceType;
  icon: string;               // Lucide icon name
  label: string;              // Tooltip text
  enabledFromStep: PipelineStep;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface IconRailProps {
  activeCreativeSpace: CreativeSpaceType;
  // currentStep via useCurrentStep() - no prop drilling
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
}

// No local state - stateless component
```

**Store Integration:**

```typescript
// EditorSettingsStore (global UI state)
currentStep = useCurrentStep();  // ⚡ no prop drilling

// Actions: none (read-only consumer)
```

### 2.3 Configuration

```typescript
const STEP_ORDER: Record<PipelineStep, number> = {
  idea: 0,
  sketch: 1,
  illustration: 2,
  retouch: 3,
};

const ICON_RAIL_ITEMS: IconRailItemConfig[] = [
  { id: 'manuscript',  icon: 'FileText',  label: 'Manuscript',    enabledFromStep: 'idea' },
  { id: 'characters',  icon: 'Smile',     label: 'Characters',    enabledFromStep: 'sketch' },
  { id: 'props',       icon: 'Box',       label: 'Props',         enabledFromStep: 'sketch' },
  { id: 'stages',      icon: 'Mountain',  label: 'Stages',        enabledFromStep: 'sketch' },
  { id: 'spreads',     icon: 'BookOpen',  label: 'Spreads',       enabledFromStep: 'sketch' },
  { id: 'objects',     icon: 'Layers',    label: 'Objects',       enabledFromStep: 'retouch' },
  { id: 'animations',  icon: 'Zap',       label: 'Animations',    enabledFromStep: 'retouch' },
  { id: 'flags',       icon: 'Flag',      label: 'Flags',         enabledFromStep: 'idea' },
  { id: 'shares',      icon: 'Share2',    label: 'Share Links',   enabledFromStep: 'idea' },
  { id: 'collabs',     icon: 'Users',     label: 'Collaborators', enabledFromStep: 'idea' },
  { id: 'config',      icon: 'Settings',  label: 'Settings',      enabledFromStep: 'idea' },
];

function isCreativeSpaceEnabled(item: IconRailItemConfig, currentStep: PipelineStep): boolean {
  return STEP_ORDER[currentStep] >= STEP_ORDER[item.enabledFromStep];
}
```

### 2.4 Render Logic (pseudo)

```
IconRail:
  // Get currentStep from EditorSettingsStore (no prop drilling)
  currentStep = useCurrentStep()

  RENDER nav container với flex-col, bg-background, py-2

  FOR EACH item IN ICON_RAIL_ITEMS:
    isEnabled = isCreativeSpaceEnabled(item, currentStep)
    isActive = activeCreativeSpace === item.id

    RENDER IconRailItem với item, isActive, isEnabled, onClick

    // Separators
    IF item.id === 'stages':
      RENDER Separator
    IF item.id === 'animations':
      RENDER Separator
```

### 2.5 Visual

```
┌─────────────────────┐
│  ┌───────────────┐  │
│  │ 📄 Manuscript │  │  ← active (blue bg, white icon)
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ 😊 Characters │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ 📦 Props      │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ ⛰️ Stages     │  │
│  └───────────────┘  │
│  ─────────────────  │  ← Separator
│  ┌───────────────┐  │
│  │ 📖 Spreads    │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ 📚 Objects    │  │  ← disabled (grayed)
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ ⚡ Animations  │  │  ← disabled (grayed)
│  └───────────────┘  │
│  ─────────────────  │  ← Separator
│  ┌───────────────┐  │
│  │ 🚩 Flags      │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ 🔗 Shares     │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ 👥 Collabs    │  │
│  └───────────────┘  │
│  ┌───────────────┐  │
│  │ ⚙️ Settings    │  │
│  └───────────────┘  │
└─────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 IconRailItem

📄 **Doc:** *(inline, không cần file riêng)*

**Mục đích:** Icon button đơn lẻ. Handle active/hover/disabled states, show tooltip on hover.

**Props & Callbacks:**

```typescript
interface IconRailItemProps {
  item: IconRailItemConfig;
  isActive: boolean;
  isEnabled: boolean;
  onClick: () => void;
}
```

**Visual (theo screenshot):**

```
Normal (enabled):        Active:                  Hover:
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│                  │     │                  │     │                  │
│    ┌────────┐    │     │  ┌────────────┐  │     │    ┌────────┐    │
│    │ [icon] │    │     │  │   [icon]   │  │     │    │ [icon] │    │
│    │  gray  │    │     │  │   white    │  │     │    │  gray  │    │
│    └────────┘    │     │  └────────────┘  │     │    └────────┘    │
│                  │     │   blue bg        │     │   subtle bg      │
└──────────────────┘     │   rounded-lg     │     └──────────────────┘
                         └──────────────────┘
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Flat Structure**
Render 11 items trực tiếp, không có intermediate group component. Đơn giản, dễ maintain.

**Active State = Filled Background**
Theo screenshot: active item có background primary (blue) với icon màu trắng, rounded-lg. Khác với left accent bar pattern.

**Progressive Unlock**
Items disabled dựa trên `enabledFromStep`. Disabled items hiển thị (grayed) để user biết sẽ unlock ở step nào.

**Store-based currentStep**
`currentStep` lấy từ `EditorSettingsStore` thay vì props để tránh prop drilling từ EditorPage.

### 4.2 Icon Mapping (theo screenshot)

| CreativeSpace | Icon (Lucide) | Visual |
|-----------|---------------|--------|
| manuscript | FileText | Document with lines |
| characters | Smile | Smiley face |
| props | Box | 3D cube |
| stages | Mountain | Mountains |
| spreads | BookOpen | Open book |
| objects | Layers | Stacked layers |
| animations | Zap | Lightning bolt |
| flags | Flag | Flag |
| shares | Share2 | Share nodes |
| collabs | Users | Multiple people |
| config | Settings | Gear |

### 4.3 Separator Positions

| After Item | Position | Visual Gap |
|------------|----------|------------|
| `stages` (index 3) | After item 4 | Horizontal line |
| `animations` (index 6) | After item 7 | Horizontal line + larger gap |

### 4.4 Accessibility

| Attribute | Value |
|-----------|-------|
| role | `navigation` on container |
| aria-current | `page` for active item |
| aria-label | item.label |
| aria-disabled | `true` for disabled items |
| keyboard | Arrow keys navigate, Enter/Space select |

---
