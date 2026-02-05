# IconRail: Component Design

**Screenshot:** `screenshots/icon-rail.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌────────────────────────────────────────────┐
│               IconRail                      │
│  ┌──────────────────────────────────────┐  │
│  │ IconRailItem - Manuscripts (active)  │  │
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
┌──────────────────────────────────────────────────────────────────┐
│                           EditorPage                              │
│  State: activeWorkspace, currentStep                              │
└───────────────────────────────────┬──────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                           IconRail                                │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Props: activeWorkspace, currentStep, onWorkspaceChange     │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                │                                  │
│    FOR EACH item IN ICON_RAIL_ITEMS:                             │
│         │                                                         │
│         ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                      IconRailItem                            │ │
│  │  Props:                                                      │ │
│  │  • item: IconRailItemConfig                                  │ │
│  │  • isActive: boolean                                         │ │
│  │  • isEnabled: boolean                                        │ │
│  │  Callback:                                                   │ │
│  │  • onClick: () => void                                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
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
                              ▲                       ▲
                        filled bg (primary)     opacity: 0.4
                        white icon              cursor: not-allowed
```

---

## 2. Component Designs

### 2.1 IconRail (Root Component)

**Mục đích:** Sidebar navigation dọc bên trái Editor. Render trực tiếp 11 IconRailItem với 2 separators.

**Shared Types:**

```typescript
type Step = 'idea' | 'sketch' | 'illustration' | 'retouch';

type WorkspaceType =
  | 'manuscripts' | 'characters' | 'props' | 'stages' | 'spreads'
  | 'objects' | 'animations' | 'flags' | 'shares' | 'collabs' | 'config';

interface IconRailItemConfig {
  id: WorkspaceType;
  icon: string;               // Lucide icon name
  label: string;              // Tooltip text
  enabledFromStep: Step;
}
```

**Interface:**

```typescript
interface IconRailProps {
  activeWorkspace: WorkspaceType;
  currentStep: Step;
  onWorkspaceChange: (workspace: WorkspaceType) => void;
}
```

**Configuration:**

```typescript
const STEP_ORDER: Record<Step, number> = {
  idea: 0,
  sketch: 1,
  illustration: 2,
  retouch: 3,
};

const ICON_RAIL_ITEMS: IconRailItemConfig[] = [
  { id: 'manuscripts', icon: 'FileText',  label: 'Manuscripts',   enabledFromStep: 'idea' },
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

function isWorkspaceEnabled(item: IconRailItemConfig, currentStep: Step): boolean {
  return STEP_ORDER[currentStep] >= STEP_ORDER[item.enabledFromStep];
}
```

**Render Logic (pseudo):**

```
IconRail:
  RENDER nav container với flex-col, bg-background, py-2

  FOR EACH item IN ICON_RAIL_ITEMS:
    isEnabled = isWorkspaceEnabled(item, currentStep)
    isActive = activeWorkspace === item.id

    RENDER IconRailItem với item, isActive, isEnabled, onClick
```

---

### 2.2 IconRailItem

**Mục đích:** Icon button đơn lẻ. Handle active/hover/disabled states, show tooltip on hover.

**Interface:**

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

## 3. Technical Notes

### 3.1 Key Design Decisions

**Flat Structure**
Render 11 items trực tiếp, không có intermediate group component. Đơn giản, dễ maintain.

**Active State = Filled Background**
Theo screenshot: active item có background primary (blue) với icon màu trắng, rounded-lg. Khác với left accent bar pattern.

**Progressive Unlock**
Items disabled dựa trên `enabledFromStep`. Disabled items hiển thị (grayed) để user biết sẽ unlock ở step nào.

**Icon Mapping (theo screenshot)**
| Workspace | Icon (Lucide) | Visual |
|-----------|---------------|--------|
| manuscripts | FileText | Document with lines |
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

### 3.2 Separator Positions

| After Item | Position | Visual Gap |
|------------|----------|------------|
| `stages` (index 3) | After item 4 | Horizontal line |
| `animations` (index 6) | After item 7 | Horizontal line + larger gap |

### 3.3 Accessibility

- `role="navigation"` on container
- `aria-current="page"` for active item
- `aria-label` = item label
- `aria-disabled="true"` for disabled items
- Keyboard: Arrow keys navigate, Enter/Space select

### 3.4 Step → Enabled Items

| currentStep | Enabled Workspaces |
|-------------|--------------------|
| `idea` | manuscripts, flags, shares, collabs, config |
| `sketch` | + characters, props, stages, spreads |
| `illustration` | (same as sketch) |
| `retouch` | + objects, animations |

### 3.5 Edge Case

Nếu `activeWorkspace` bị disable sau khi step regress → auto-switch về `manuscripts`.
