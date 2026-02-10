# DummySidebar: Component Design

> **Parent:** [DummyCreativeSpace](./README.md)

**Screenshot:** `screenshots/manuscript-dummy-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                DummySidebar                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SidebarHeader                                                           │   │
│  │  "Dummies"                                                        [+]    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  DummyList (accordion, dynamic)                                          │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ DummyItem (Prose Dummy 1) - EXPANDED                               │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ Header: 📐 Prose Dummy 1  [prose badge]                    [∨] │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ PromptPanel                                                    │ │  │   │
│  │  │ │   PROMPT: [Enter your prompt for this dummy layout...]         │ │  │   │
│  │  │ │   [✨ Generate]                                    [🗑 Delete] │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ DummyItem (Poetry Dummy 1) - COLLAPSED                         [>] │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  AddDummyDropdown (on + click)                                           │   │
│  │  ┌────────────────┐                                                      │   │
│  │  │ 📐 Prose Dummy │                                                      │   │
│  │  │ 📐 Poetry Dummy│                                                      │   │
│  │  └────────────────┘                                                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
        ┌─────────────────────────────┐
        │     DummyCreativeSpace      │
        │     (parent component)      │
        └─────────────┬───────────────┘
                      │ (props)
                      ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                               DummySidebar                                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: activeDummyId, onDummySelect, onAddDummy                            │  │
│  │  Local State: expandedDummyId, promptInputs, isGenerating, showAddDropdown  │  │
│  │  Store: dummies = useDummies()                                              │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                         │
│         ▼                                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────┐    │
│  │                          DummyItem × N                                    │    │
│  │  Props:                                                                   │    │
│  │  • dummy: ManuscriptDummy                                                 │    │
│  │  • isActive: boolean                                                      │    │
│  │  • isExpanded: boolean                                                    │    │
│  │  • promptInput: string                                                    │    │
│  │  • isGenerating: boolean                                                  │    │
│  │  Callbacks:                                                               │    │
│  │  • onToggle: () => void                                                   │    │
│  │  • onPromptChange: (value: string) => void                                │    │
│  │  • onGenerate: () => void                                                 │    │
│  │  • onDelete: () => void                                                   │    │
│  └───────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Left sidebar với dynamic dummy list (accordion). User can create multiple dummies, each with PromptPanel for AI generation.

**Shared Types:**

```typescript
type DummyType = 'prose' | 'poetry';

interface ManuscriptDummy {
  id: string;
  title: string;
  type: DummyType;
  spreads: DummySpread[];
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface DummySidebarProps {
  activeDummyId: string | null;
  onDummySelect: (dummyId: string) => void;
  onAddDummy: (type: DummyType) => void;
}

interface DummySidebarState {
  expandedDummyId: string | null;              // Which accordion is expanded
  promptInputs: Record<string, string>;        // dummyId → prompt
  isGenerating: Record<string, boolean>;       // dummyId → loading state
  showAddDropdown: boolean;                    // Add button dropdown visibility
  editingTitleId: string | null;               // Which dummy's title is being edited
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors
dummies = useDummies();  // ManuscriptDummy[]

// SnapshotStore Actions
{ deleteDummy, updateDummy } = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
DummySidebar:
  // Props from parent
  { activeDummyId, onDummySelect, onAddDummy } = props

  // Store
  dummies = useDummies()
  { deleteDummy } = useSnapshotActions()

  // Local state
  [expandedDummyId, setExpandedDummyId] = useState(activeDummyId)
  [promptInputs, setPromptInputs] = useState({})
  [isGenerating, setIsGenerating] = useState({})
  [showAddDropdown, setShowAddDropdown] = useState(false)

  handleToggle(dummyId):
    IF expandedDummyId === dummyId:
      setExpandedDummyId(null)
    ELSE:
      setExpandedDummyId(dummyId)
      onDummySelect(dummyId)  // Also select when expanding

  handlePromptChange(dummyId, value):
    setPromptInputs({ ...promptInputs, [dummyId]: value })

  handleGenerate(dummyId):
    setIsGenerating({ ...isGenerating, [dummyId]: true })
    // Call generate API (via hook or callback)
    // On complete: setIsGenerating({ ...isGenerating, [dummyId]: false })

  handleDelete(dummyId):
    // Show confirmation dialog
    IF confirmed:
      deleteDummy(dummyId)

  handleTitleEditStart(dummyId):
    setEditingTitleId(dummyId)

  handleTitleChange(dummyId, newTitle):
    updateDummy(dummyId, { title: newTitle })
    setEditingTitleId(null)

  handleTitleEditCancel():
    setEditingTitleId(null)

  handleAddClick():
    setShowAddDropdown(!showAddDropdown)

  handleAddDummy(type: DummyType):
    onAddDummy(type)
    setShowAddDropdown(false)

  RENDER Container (flex column):

    // Header
    RENDER SidebarHeader:
      - title: "Dummies"
      - addButton: onClick → handleAddClick

    // Add dropdown (when visible)
    IF showAddDropdown:
      RENDER AddDummyDropdown:
        - onSelect: handleAddDummy
        - onClose: () => setShowAddDropdown(false)

    // Empty state
    IF dummies.length === 0:
      RENDER EmptyState:
        - "No dummies yet"
        - "Click + to create your first dummy"
      RETURN

    // Dummy list
    FOR EACH dummy IN dummies:
      isActive = activeDummyId === dummy.id
      isExpanded = expandedDummyId === dummy.id

      RENDER DummyItem với:
        - dummy
        - isActive
        - isExpanded
        - isEditingTitle: editingTitleId === dummy.id
        - promptInput: promptInputs[dummy.id] ?? ''
        - isGenerating: isGenerating[dummy.id] ?? false
        - onToggle: () => handleToggle(dummy.id)
        - onTitleEditStart: () => handleTitleEditStart(dummy.id)
        - onTitleChange: (title) => handleTitleChange(dummy.id, title)
        - onTitleEditCancel: handleTitleEditCancel
        - onPromptChange: (v) => handlePromptChange(dummy.id, v)
        - onGenerate: () => handleGenerate(dummy.id)
        - onDelete: () => handleDelete(dummy.id)
```

### 2.4 Visual

**With Dummies (one expanded):**

```
┌────────────────────────────────┐
│ Dummies                   [+]  │
│ ┌────────────────────────────┐ │
│ │ 📐 Prose Dummy 1   prose ∨ │ │  ← Expanded, Active
│ │ ────────────────────────── │ │
│ │ PROMPT                     │ │
│ │ ┌────────────────────────┐ │ │
│ │ │ Enter your prompt for  │ │ │
│ │ │ this dummy layout...   │ │ │
│ │ └────────────────────────┘ │ │
│ │ ┌──────────────┐ ┌───────┐ │ │
│ │ │ ✨ Generate  │ │ 🗑    │ │ │
│ │ └──────────────┘ └───────┘ │ │
│ └────────────────────────────┘ │
│ 📐 Poetry Dummy 1  poetry  >   │  ← Collapsed
│ 📐 Prose Dummy 2   prose   >   │  ← Collapsed
└────────────────────────────────┘
```

**Title Hover State:**

```
┌────────────────────────────────┐
│ 📐 Prose Dummy 1   prose  ✏️ ∨  │  ← Pencil icon appears on hover
│ ────────────────────────────── │
│ PROMPT                         │
└────────────────────────────────┘
```

**Title Edit State:**

```
┌────────────────────────────────┐
│ 📐 [Prose Dummy 1      ]  ✓ ✕  │  ← Input field, confirm/cancel
│ ────────────────────────────── │
│ PROMPT                         │
└────────────────────────────────┘
```

**Add Dropdown:**

```
┌────────────────────────────────┐
│ Dummies                   [+]  │
│                          ┌─────────────────┐
│                          │ 📐 Prose Dummy  │
│                          │ 📐 Poetry Dummy │
│                          └─────────────────┘
│ 📐 Prose Dummy 1   prose    >  │
│ ...                            │
└────────────────────────────────┘
```

**Empty State:**

```
┌────────────────────────────────┐
│ Dummies                   [+]  │
│                                │
│                                │
│      📐 No dummies yet         │
│                                │
│      Click + to create your    │
│      first dummy layout.       │
│                                │
│                                │
└────────────────────────────────┘
```

**Generating State:**

```
┌────────────────────────────────┐
│ 📐 Prose Dummy 1   prose    ∨  │
│ ────────────────────────────── │
│ PROMPT                         │
│ ┌────────────────────────────┐ │
│ │ Create a layout for a      │ │
│ │ children's adventure...    │ │
│ └────────────────────────────┘ │
│ ┌──────────────────┐ ┌───────┐ │
│ │  ⏳ Generating...│ │ 🗑    │ │  ← Disabled
│ └──────────────────┘ └───────┘ │
└────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks**.
> Sub-components là elements, không cần file riêng.

### 3.1 SidebarHeader

**Mục đích:** Header với title và add button.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Title | `<span>` | "Dummies" |
| Add button | `<button>` | Opens dropdown for type selection |

### 3.2 DummyItem

**Mục đích:** Accordion item for a single dummy. Expandable với PromptPanel.

**Props & Callbacks:**

```typescript
interface DummyItemProps {
  dummy: ManuscriptDummy;
  isActive: boolean;
  isExpanded: boolean;
  isEditingTitle: boolean;
  promptInput: string;
  isGenerating: boolean;
  onToggle: () => void;
  onTitleEditStart: () => void;
  onTitleChange: (title: string) => void;
  onTitleEditCancel: () => void;
  onPromptChange: (value: string) => void;
  onGenerate: () => void;
  onDelete: () => void;
}
```

**Visual:**

```
Normal (not hovering):
┌────────────────────────────────────────────────────┐
│ 📐 Prose Dummy 1   [prose]                       > │
└────────────────────────────────────────────────────┘

Hover (shows pencil):
┌────────────────────────────────────────────────────┐
│ 📐 Prose Dummy 1   [prose]  ✏️                    > │
│                              ↑ pencil on hover     │
└────────────────────────────────────────────────────┘

Editing Title:
┌────────────────────────────────────────────────────┐
│ 📐 [Prose Dummy 1                ]  ✓  ✕         > │
│     └─────── input field ────────┘  ↑  ↑           │
│                                  save cancel       │
└────────────────────────────────────────────────────┘

Expanded (not editing):
┌────────────────────────────────────────────────────┐
│ 📐 Prose Dummy 1   [prose]  ✏️                    ∨ │
├────────────────────────────────────────────────────┤
│ PROMPT                                             │
│ ┌────────────────────────────────────────────────┐ │
│ │ Enter your prompt...                           │ │
│ └────────────────────────────────────────────────┘ │
│ ┌────────────────────┐ ┌────────────────────────┐  │
│ │   ✨ Generate      │ │        🗑 Delete       │  │
│ └────────────────────┘ └────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

### 3.3 AddDummyDropdown

**Mục đích:** Dropdown for selecting dummy type when adding new dummy.

**Props & Callbacks:**

```typescript
interface AddDummyDropdownProps {
  onSelect: (type: DummyType) => void;
  onClose: () => void;
}
```

**Visual:**

```
┌─────────────────────┐
│ 📐 Prose Dummy      │  ← Creates prose-type dummy
│ 📐 Poetry Dummy     │  ← Creates poetry-type dummy
└─────────────────────┘
```

### 3.4 PromptPanel

**Mục đích:** Prompt textarea with generate button.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Prompt label | `<label>` | "PROMPT" |
| Prompt textarea | `<textarea>` | Multi-line input |
| Generate button | `<button>` | Primary action |
| Delete button | `<button>` | Danger action, with confirmation |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Dynamic List vs Fixed Tabs**
DummySidebar has a dynamic list. User can create multiple dummies of any type.

**Accordion Exclusivity**
Only one DummyItem expanded at a time. Expanding also selects the dummy for viewing in SpreadView.

**Delete Confirmation**
Delete button shows confirmation dialog to prevent accidental deletion.

**Type Badge**
Each DummyItem shows a type badge (`prose` or `poetry`) to distinguish dummy types.

### 4.2 Layout Constants

| Element | Value |
|---------|-------|
| Sidebar width | 280px |
| Item header height | 44px |
| PromptPanel padding | 12px |
| Textarea min-height | 80px |
| Type badge | 12px font, rounded, muted bg |

### 4.3 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Container | `navigation` | `aria-label="Dummy sidebar"` |
| Add button | `button` | `aria-haspopup="menu"`, `aria-expanded` |
| DummyList | `listbox` | `aria-label="Dummy layouts"` |
| DummyItem header | `option` | `aria-selected`, `aria-expanded` |
| Title input | `textbox` | `aria-label="Edit dummy title"` |
| Save title btn | `button` | `aria-label="Save title"` |
| Cancel edit btn | `button` | `aria-label="Cancel editing"` |
| Delete button | `button` | `aria-label="Delete dummy"` |

### 4.4 Keyboard Navigation

| Key | Action |
|-----|--------|
| `Tab` | Move between items |
| `Enter` / `Space` | Toggle expanded |
| `Arrow Up/Down` | Navigate items |
| `F2` | Edit selected dummy title |
| `Enter` (in title input) | Save title |
| `Escape` (in title input) | Cancel editing |
| `Delete` | Delete selected (with confirmation) |

---
