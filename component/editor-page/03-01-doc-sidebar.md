# DocSidebar: Component Design

> **Parent:** [DocCreativeSpace](component/editor-page/03-doc-creative-space.md)

**Screenshot:** `screenshots/manuscript-docs-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 DocSidebar                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  SidebarHeader                                                           │   │
│  │  "Docs"                                                           [+]    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  DocTabList (accordion)                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ DocTabItem (Brief) - EXPANDED                                      │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ Header: 📄 Brief                                           [∨] │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  │ ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │ │ PromptPanel                                                    │ │  │   │
│  │  │ │   TARGET AUDIENCE: [Select target audience...]                 │ │  │   │
│  │  │ │   CORE VALUE: [Select core value...]                           │ │  │   │
│  │  │ │   FORMAT GENRE: [Select format genre...]                       │ │  │   │
│  │  │ │   CONTENT GENRE: [Select content genre...]                     │ │  │   │
│  │  │ │   ERA: [Select era (optional)...]                              │ │  │   │
│  │  │ │   LOCATION: [Select location (optional)...]                    │ │  │   │
│  │  │ │   PROMPT: [Enter your prompt...]                               │ │  │   │
│  │  │ │   [✨ Generate]                                                │ │  │   │
│  │  │ └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ DocTabItem (Draft) - COLLAPSED                                 [>] │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │   │
│  │  │ DocTabItem (Script) - COLLAPSED                                [>] │  │   │
│  │  └────────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
        ┌─────────────────────────────┐
        │       DocCreativeSpace      │
        │     (parent component)      │
        └─────────────┬───────────────┘
                      │ (props)
                      ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                DocSidebar                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: activeDocType, onDocTypeChange                                      │  │
│  │  Local State: expandedDocType, promptInputs, isGenerating                   │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                         │
│         ▼                                                                         │
│  ┌───────────────────────────────────────────────────────────────────────────┐    │
│  │                           DocTabItem × 3                                  │    │
│  │  Props:                                                                   │    │
│  │  • docType: DocType                                                       │    │
│  │  • isActive: boolean                                                      │    │
│  │  • isExpanded: boolean                                                    │    │
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

**Mục đích:** Left sidebar với accordion-style tabs cho Brief/Draft/Script. Each tab expands to show PromptPanel for AI generation.

**Shared Types:**

```typescript
type DocType = 'brief' | 'draft' | 'script';

interface DocTabConfig {
  type: DocType;
  icon: string;       // Lucide icon name
  label: string;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface DocSidebarProps {
  activeDocType: DocType;
  onDocTypeChange: (type: DocType) => void;
}

interface DocSidebarState {
  expandedDocType: DocType | null;           // Which accordion is expanded
  promptInputs: Record<DocType, string>;     // Prompt per doc type
  isGenerating: Record<DocType, boolean>;    // Loading state per doc type
}
```

**Store Integration:**

```typescript
// No direct store access - pure presentational with callbacks
// Generate action will be handled by parent via callback or separate hook
```

### 2.3 Configuration

```typescript
const DOC_TABS: DocTabConfig[] = [
  { type: 'brief',  icon: 'FileText', label: 'Brief' },
  { type: 'draft',  icon: 'FileText', label: 'Draft' },
  { type: 'script', icon: 'FileText', label: 'Script' },
];
```

### 2.4 Render Logic (pseudo)

```
DocSidebar:
  // Props from parent
  { activeDocType, onDocTypeChange } = props

  // Local state
  [expandedDocType, setExpandedDocType] = useState(activeDocType)
  [promptInputs, setPromptInputs] = useState({ brief: '', draft: '', script: '' })
  [isGenerating, setIsGenerating] = useState({ brief: false, draft: false, script: false })

  handleToggle(docType):
    IF expandedDocType === docType:
      setExpandedDocType(null)
    ELSE:
      setExpandedDocType(docType)
      onDocTypeChange(docType)  // Also set active when expanding

  handlePromptChange(docType, value):
    setPromptInputs({ ...promptInputs, [docType]: value })

  handleGenerate(docType):
    setIsGenerating({ ...isGenerating, [docType]: true })
    // Call generate API (via hook or callback)
    // On complete: setIsGenerating({ ...isGenerating, [docType]: false })

  RENDER Container (flex column):

    // Header
    RENDER SidebarHeader:
      - title: "Docs"
      - addButton: null (no add functionality for fixed doc types)

    // Accordion tabs
    FOR EACH tab IN DOC_TABS:
      isActive = activeDocType === tab.type
      isExpanded = expandedDocType === tab.type

      RENDER DocTabItem với:
        - docType: tab.type
        - icon: tab.icon
        - label: tab.label
        - isActive
        - isExpanded
        - promptInput: promptInputs[tab.type]
        - isGenerating: isGenerating[tab.type]
        - onToggle: () => handleToggle(tab.type)
        - onPromptChange: (v) => handlePromptChange(tab.type, v)
        - onGenerate: () => handleGenerate(tab.type)
```

### 2.5 Visual

**Expanded State (Brief):**

```
┌────────────────────────────────┐
│ Docs                      [+]  │
│ ┌────────────────────────────┐ │
│ │ 📄 Brief                 ∨ │ │  ← Expanded, Active
│ │ ────────────────────────── │ │
│ │ TARGET AUDIENCE *          │ │
│ │ [Select target audience...]│ │
│ │ CORE VALUE *               │ │
│ │ [Select core value...]     │ │
│ │ FORMAT GENRE *             │ │
│ │ [Select format genre...]   │ │
│ │ CONTENT GENRE *            │ │
│ │ [Select content genre...]  │ │
│ │ ERA                        │ │
│ │ [Select era (optional)...] │ │
│ │ LOCATION                   │ │
│ │ [Select location...]       │ │
│ │ PROMPT                     │ │
│ │ ┌────────────────────────┐ │ │
│ │ │ Enter your prompt for  │ │ │
│ │ │ this manuscript...     │ │ │
│ │ └────────────────────────┘ │ │
│ │ ┌────────────────────────┐ │ │
│ │ │     ✨ Generate        │ │ │
│ │ └────────────────────────┘ │ │
│ └────────────────────────────┘ │
│ 📄 Draft                   > │ │  ← Collapsed
│ 📄 Script                  > │ │  ← Collapsed
└────────────────────────────────┘
```

**Collapsed State (all):**

```
┌────────────────────────────────┐
│ Docs                      [+]  │
│ 📄 Brief                    >  │
│ 📄 Draft                    >  │
│ 📄 Script                   >  │
└────────────────────────────────┘
```

**Generating State:**

```
┌────────────────────────────────┐
│ 📄 Brief                     ∨ │
│ ────────────────────────────── │
│ ...                            │
│ PROMPT                         │
│ ┌────────────────────────────┐ │
│ │ Generate a story about...  │ │
│ └────────────────────────────┘ │
│ ┌────────────────────────────┐ │
│ │  ⏳ Generating...          │ │  ← Disabled, loading
│ └────────────────────────────┘ │
└────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks**.
> Sub-components là elements, không cần file riêng.

### 3.1 SidebarHeader

**Mục đích:** Header với title và optional add button.

**Elements (không phải component riêng):**

| Element | Type | Notes |
|---------|------|-------|
| Title | `<span>` | "Docs" |
| Add button | `<button>` | Hidden/disabled (no add for fixed doc types) |

### 3.2 DocTabItem

**Mục đích:** Accordion tab item. Expandable với PromptPanel.

**Props & Callbacks:**

```typescript
interface DocTabItemProps {
  docType: DocType;
  icon: string;
  label: string;
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
│ 📄 Brief                                         > │
└────────────────────────────────────────────────────┘

Expanded:
┌────────────────────────────────────────────────────┐
│ 📄 Brief                                         ∨ │
├────────────────────────────────────────────────────┤
│ PromptPanel content...                             │
│ [✨ Generate]                                      │
└────────────────────────────────────────────────────┘

Active + Collapsed:
┌────────────────────────────────────────────────────┐
│ 📄 Brief   ●                                     > │  ← Active indicator dot
└────────────────────────────────────────────────────┘
```

### 3.3 PromptPanel

**Mục đích:** Form với attribute selectors và prompt textarea.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Attribute selects | `<select>` | Target Audience, Core Value, etc. |
| Prompt textarea | `<textarea>` | Multi-line input |
| Generate button | `<button>` | Triggers AI generation |

**Attribute Fields (Brief only):**

| Field | Required | Options Source |
|-------|----------|----------------|
| TARGET AUDIENCE | ✅ | Constants / Book settings |
| CORE VALUE | ✅ | Constants / Book settings |
| FORMAT GENRE | ✅ | Constants |
| CONTENT GENRE | ✅ | Constants |
| ERA | ❌ | Constants |
| LOCATION | ❌ | Constants |

> **Note:** Draft and Script tabs show only PROMPT field (no attributes).

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Accordion Exclusivity**
Only one tab expanded at a time. Expanding a new tab collapses the previous one and sets it as active.

**Active vs Expanded**
- `isActive`: Which doc is shown in the editor (controlled by parent)
- `isExpanded`: Which tab's PromptPanel is visible (local state)
- Expanding always sets active, but clicking outside doesn't collapse

**Attribute Fields Location**
Attribute fields (TARGET AUDIENCE, CORE VALUE, etc.) appear only in Brief tab. Draft and Script tabs show only the PROMPT textarea since they build on previous content.

**Generate Dependency Chain**

| Doc Type | Requires | Generates |
|----------|----------|-----------|
| Brief | Attributes + Prompt | Story framework |
| Draft | Brief content + Prompt | Full narrative |
| Script | Draft content + Prompt | Scene-by-scene script |

### 4.2 Layout Constants

| Element | Value |
|---------|-------|
| Sidebar width | 280px |
| Tab header height | 44px |
| PromptPanel padding | 12px |
| Textarea min-height | 80px |
| Generate button height | 36px |

### 4.3 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Container | `navigation` | `aria-label="Document sidebar"` |
| DocTabItem header | `button` | `aria-expanded`, `aria-controls` |
| DocTabItem panel | `region` | `aria-labelledby` |
| Generate button | `button` | `aria-busy` when loading |

### 4.4 Keyboard Navigation

| Key | Action |
|-----|--------|
| `Tab` | Move between tabs |
| `Enter` / `Space` | Toggle expanded |
| `Arrow Up/Down` | Navigate tabs (within tablist) |

---
