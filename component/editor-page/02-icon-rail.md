# IconRail: Component Design

**Screenshots:**
- Manuscript Dummy: `screenshots/manuscript-dummy-space.png`
- Manuscript Sketch: `screenshots/manuscript-sketch-space.png`
- Retouch Remix: `screenshots/Retouch-remix-space.png`
- History: `screenshots/history-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌────────────────────────────────────────────┐
│               IconRail                     │
│  ┌──────────────────────────────────────┐  │
│  │ TOP: STEP-SPECIFIC ICONS             │  │
│  │ ─────────────────────────────────────│  │
│  │ Manuscript: doc, dummy, sketch       │  │
│  │ Illustration: character, prop,       │  │
│  │               stage, spread          │  │
│  │ Retouch: object, animation, remix    │  │
│  │                                      │  │
│  │           (flex spacer)              │  │
│  │                                      │  │
│  │ BOTTOM: DEFAULT ICONS                │  │
│  │ ─────────────────────────────────────│  │
│  │ history, flag, share, collaborator   │  │
│  ├──────────────────────────────────────┤  │  ← Separator (only above setting)
│  │ setting                              │  │
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
│  │  Derived: getIconsForStep(currentStep)                         │  │
│  └────────────────────────────────────────────────────────────────┘  │
│         │                                                            │
│    STRUCTURE: [stepIcons] + spacer + [DEFAULT_ICONS] + sep + setting │
│         │                                                            │
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                        IconRailItem                            │  │
│  │  Props:                                                        │  │
│  │  • item: IconRailItemConfig                                    │  │
│  │  • isActive: boolean                                           │  │
│  │  Callback:                                                     │  │
│  │  • onClick: () => void                                         │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.3 Visual States

```
Default:                Active:
┌────────────────┐      ┌────────────────┐
│                │      │ ┌────────────┐ │
│   ┌────────┐   │      │ │   Icon     │ │
│   │  Icon  │   │      │ │  (white)   │ │
│   └────────┘   │      │ └────────────┘ │
│                │      │  blue bg,      │
└────────────────┘      │  rounded-lg    │
                        └────────────────┘
                              ↑
                        filled bg (primary)
                        white icon
```

### 1.4 Step → Icon Mapping

| currentStep | Step-specific Icons | Default Icons |
|-------------|---------------------|---------------|
| `manuscript` | doc, dummy, sketch | history, flag, share, collaborator, setting |
| `illustration` | character, prop, stage, spread | history, flag, share, collaborator, setting |
| `retouch` | object, animation, remix | history, flag, share, collaborator, setting |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Sidebar navigation dọc bên trái Editor. Render step-specific icons + default icons với 1 separator giữa. Icons thay đổi hoàn toàn dựa trên `currentStep`.

**Shared Types:**

```typescript
type PipelineStep = 'manuscript' | 'illustration' | 'retouch';

// Step-specific creative spaces
type ManuscriptSpace = 'doc' | 'dummy' | 'sketch';
type IllustrationSpace = 'character' | 'prop' | 'stage' | 'spread';
type RetouchSpace = 'object' | 'animation' | 'remix';

// Default creative spaces (always available)
type DefaultSpace = 'history' | 'flag' | 'share' | 'collaborator' | 'setting';

type CreativeSpaceType = ManuscriptSpace | IllustrationSpace | RetouchSpace | DefaultSpace;

interface IconRailItemConfig {
  id: CreativeSpaceType;
  icon: string;               // Lucide icon name
  label: string;              // Tooltip text
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface IconRailProps {
  activeCreativeSpace: CreativeSpaceType;
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
// Step-specific icons
const MANUSCRIPT_ICONS: IconRailItemConfig[] = [
  { id: 'doc',    icon: 'FileText',   label: 'Document' },
  { id: 'dummy',  icon: 'LayoutGrid', label: 'Dummy Layout' },
  { id: 'sketch', icon: 'Pencil',     label: 'Sketch' },
];

const ILLUSTRATION_ICONS: IconRailItemConfig[] = [
  { id: 'character', icon: 'Smile',    label: 'Characters' },
  { id: 'prop',      icon: 'Box',      label: 'Props' },
  { id: 'stage',     icon: 'Mountain', label: 'Stages' },
  { id: 'spread',    icon: 'BookOpen', label: 'Spreads' },
];

const RETOUCH_ICONS: IconRailItemConfig[] = [
  { id: 'object',    icon: 'Layers',    label: 'Objects' },
  { id: 'animation', icon: 'Zap',       label: 'Animations' },
  { id: 'remix',     icon: 'RefreshCw', label: 'Remix' },
];

// Default icons (bottom, always visible)
const DEFAULT_ICONS: IconRailItemConfig[] = [
  { id: 'history',      icon: 'History',  label: 'History' },
  { id: 'flag',         icon: 'Flag',     label: 'Flags' },
  { id: 'share',        icon: 'Share2',   label: 'Share Links' },
  { id: 'collaborator', icon: 'Users',    label: 'Collaborators' },
];

// Setting icon (separated at very bottom)
const SETTING_ICON: IconRailItemConfig =
  { id: 'setting', icon: 'Settings', label: 'Settings' };

const STEP_ICONS: Record<PipelineStep, IconRailItemConfig[]> = {
  manuscript: MANUSCRIPT_ICONS,
  illustration: ILLUSTRATION_ICONS,
  retouch: RETOUCH_ICONS,
};

function getIconsForStep(step: PipelineStep): IconRailItemConfig[] {
  return STEP_ICONS[step] ?? MANUSCRIPT_ICONS;
}
```

### 2.4 Render Logic (pseudo)

```
IconRail:
  currentStep = useCurrentStep()
  stepIcons = getIconsForStep(currentStep)

  RENDER nav container với flex-col, h-full, bg-background, py-2

  // TOP: Step-specific icons
  FOR EACH item IN stepIcons:
    isActive = activeCreativeSpace === item.id
    RENDER IconRailItem với item, isActive, onClick

  // Flex spacer (push default icons to bottom)
  RENDER div với flex-1

  // BOTTOM: Default icons (no separator)
  FOR EACH item IN DEFAULT_ICONS:
    isActive = activeCreativeSpace === item.id
    RENDER IconRailItem với item, isActive, onClick

  // Separator line (only above setting)
  RENDER Separator

  // Setting icon (very bottom)
  RENDER IconRailItem với SETTING_ICON, isActive, onClick
```

### 2.5 Visual by Step

**Manuscript Step:**
```
┌─────────────────────┐
│  ┌───────────────┐  │
│  │ 📄 Doc        │  │  ← FileText      ┐
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │ TOP
│  │ ⊞ Dummy       │  │  ← LayoutGrid    │ (step icons)
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ ✏️ Sketch      │  │  ← Pencil        ┘
│  └───────────────┘  │
│                     │
│    (flex spacer)    │
│                     │
│  ┌───────────────┐  │                  ┐
│  │ 🕐 History    │  │  ← History       │
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │ BOTTOM
│  │ 🚩 Flags      │  │                  │ (default icons)
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ 🔗 Share      │  │                  │
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ 👥 Collabs    │  │                  ┘
│  └───────────────┘  │
│  ─────────────────  │  ← Separator (only here)
│  ┌───────────────┐  │
│  │ ⚙️ Settings    │  │  ← isolated at bottom
│  └───────────────┘  │
└─────────────────────┘
```

**Illustration Step:**
```
┌─────────────────────┐
│  ┌───────────────┐  │
│  │ 😊 Character  │  │  ← Smile         ┐
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │ TOP
│  │ 📦 Prop       │  │  ← Box           │
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ ⛰️ Stage      │  │  ← Mountain      │
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ 📖 Spread     │  │  ← BookOpen      ┘
│  └───────────────┘  │
│                     │
│    (flex spacer)    │
│                     │
│  ... (default icons)│  ← BOTTOM
│  ─────────────────  │  ← Separator
│  ⚙️ Settings         │
└─────────────────────┘
```

**Retouch Step:**
```
┌─────────────────────┐
│  ┌───────────────┐  │
│  │ 📚 Object     │  │  ← Layers        ┐
│  └───────────────┘  │                  │ TOP
│  ┌───────────────┐  │                  │
│  │ ⚡ Animation   │  │  ← Zap           │
│  └───────────────┘  │                  │
│  ┌───────────────┐  │                  │
│  │ 🔄 Remix      │  │  ← RefreshCw     ┘
│  └───────────────┘  │
│                     │
│    (flex spacer)    │
│                     │
│  ... (default icons)│  ← BOTTOM
│  ─────────────────  │  ← Separator
│  ⚙️ Settings         │
└─────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 IconRailItem

📄 **Doc:** *(inline, không cần file riêng)*

**Mục đích:** Icon button đơn lẻ. Handle active/hover states, show tooltip on hover.

**Props & Callbacks:**

```typescript
interface IconRailItemProps {
  item: IconRailItemConfig;
  isActive: boolean;
  onClick: () => void;
}
```

**Visual:**

```
Normal:                  Active:                  Hover:
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

**Dynamic Icon Set**
Icons thay đổi hoàn toàn dựa trên `currentStep`, không phải enable/disable. Khi user chuyển step, icon rail render bộ icons hoàn toàn khác.

**Active State = Filled Background**
Theo screenshot: active item có background primary (blue) với icon màu trắng, rounded-lg.

**Top-Bottom Layout với Flex Spacer**
Step icons ở top, default icons ở bottom, flex spacer đẩy default icons xuống. Separator chỉ xuất hiện trên icon Settings để tách biệt với các default icons khác.

**Store-based currentStep**
`currentStep` lấy từ `EditorSettingsStore` thay vì props để tránh prop drilling từ EditorPage.

### 4.2 Icon Mapping

| CreativeSpace | Icon (Lucide) | Step |
|---------------|---------------|------|
| doc | FileText | manuscript |
| dummy | LayoutGrid | manuscript |
| sketch | Pencil | manuscript |
| character | Smile | illustration |
| prop | Box | illustration |
| stage | Mountain | illustration |
| spread | BookOpen | illustration |
| object | Layers | retouch |
| animation | Zap | retouch |
| remix | RefreshCw | retouch |
| history | History | default |
| flag | Flag | default |
| share | Share2 | default |
| collaborator | Users | default |
| setting | Settings | default |

### 4.3 Step Transition Behavior

Khi `currentStep` thay đổi:
1. Icon rail re-renders với bộ icons mới
2. Nếu `activeCreativeSpace` không còn valid cho step mới → auto-select first icon của step mới
3. Ví dụ: đang ở `doc` (manuscript) → chuyển sang illustration → auto-select `character`

```typescript
// Handle step transition
useEffect(() => {
  const validSpaces = [
    ...getIconsForStep(currentStep).map(i => i.id),
    ...DEFAULT_ICONS.map(i => i.id),
    SETTING_ICON.id
  ];
  if (!validSpaces.includes(activeCreativeSpace)) {
    onCreativeSpaceChange(getIconsForStep(currentStep)[0].id);
  }
}, [currentStep]);
```

### 4.4 Accessibility

| Attribute | Value |
|-----------|-------|
| role | `navigation` on container |
| aria-current | `page` for active item |
| aria-label | item.label |
| keyboard | Arrow keys navigate, Enter/Space select |

---
