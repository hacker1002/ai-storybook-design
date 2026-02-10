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
  ┌─────────────────┐     ┌─────────────────┐
  │   BookStore     │     │ DocCreativeSpace│
  │  (settings,     │     │    (parent)     │
  │   references)   │     └────────┬────────┘
  └────────┬────────┘              │ (props)
           │                       ▼
           │    ┌───────────────────────────────────────────────────────┐
           │    │                     DocSidebar                        │
           └───►│  ┌─────────────────────────────────────────────────┐  │
                │  │  Props: docs, activeDocIndex, callbacks         │  │
                │  │  Store: bookSettings, bookReferences            │  │
                │  │  Local: expandedIndex, editingTitleIndex, etc.  │  │
                │  └─────────────────────────────────────────────────┘  │
                │         │                                             │
                │         ▼                                             │
                │  ┌───────────────────────────────────────────────┐    │
                │  │              DocTabItem × N                   │    │
                │  │  Props: doc, isActive, isExpanded, etc.       │    │
                │  │  ┌─────────────────────────────────────────┐  │    │
                │  │  │  PromptPanel (Brief only: attributes)   │  │    │
                │  │  │  → uses bookSettings, bookReferences    │  │    │
                │  │  └─────────────────────────────────────────┘  │    │
                │  └───────────────────────────────────────────────┘    │
                └───────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Left sidebar với accordion-style tabs cho Brief/Draft/Script. Each tab expands to show PromptPanel for AI generation.

**Shared Types:**

```typescript
type DocType = 'brief' | 'draft' | 'script' | 'other';

interface ManuscriptDoc {
  type: DocType;
  title: string;    // User-defined title (editable for 'other' type)
  content: string;
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface DocSidebarProps {
  docs: ManuscriptDoc[];                     // All docs from store
  activeDocIndex: number;                    // Currently selected doc index
  onDocSelect: (index: number) => void;
  onAddDoc: () => void;                      // Add new 'other' type doc
  onUpdateDocTitle: (index: number, title: string) => void;
  onDeleteDoc: (index: number) => void;      // Only for 'other' type
}

interface DocSidebarState {
  expandedIndex: number | null;              // Which accordion is expanded
  editingTitleIndex: number | null;          // Which doc title is being edited
  promptInputs: Record<number, string>;      // Prompt per doc index
  isGenerating: Record<number, boolean>;     // Loading state per doc
}
```

**Store Integration:**

```typescript
// BookStore - for Brief tab attributes
bookSettings = useBookSettings();    // bookType, dimension, targetAudience, targetCoreValue, formatGenre, contentGenre, writingStyle
bookReferences = useBookReferences(); // eraId, locationId, artstyleId
{ updateSettings, updateReferences } = useBookActions();

// Generate action handled via callback or separate hook
```

### 2.3 Configuration

```typescript
// Fixed doc types (cannot delete, title not editable)
const FIXED_DOC_TYPES: DocType[] = ['brief', 'draft', 'script'];

// Default titles for fixed types
const DEFAULT_TITLES: Record<DocType, string> = {
  brief: 'Brief',
  draft: 'Draft',
  script: 'Script',
  other: 'Other',  // Default for new 'other' docs
};

// Check if doc type allows title editing
const isEditableTitle = (type: DocType) => type === 'other';

// Check if doc type allows deletion
const isDeletable = (type: DocType) => type === 'other';
```

### 2.4 Render Logic (pseudo)

```
DocSidebar:
  // Props from parent
  { docs, activeDocIndex, onDocSelect, onAddDoc, onUpdateDocTitle, onDeleteDoc } = props

  // Local state
  [expandedIndex, setExpandedIndex] = useState(activeDocIndex)
  [editingTitleIndex, setEditingTitleIndex] = useState(null)
  [promptInputs, setPromptInputs] = useState({})
  [isGenerating, setIsGenerating] = useState({})

  handleToggle(index):
    IF expandedIndex === index:
      setExpandedIndex(null)
    ELSE:
      setExpandedIndex(index)
      onDocSelect(index)  // Also set active when expanding

  handleAddDoc():
    onAddDoc()  // Parent adds new 'other' doc with title "Other"
    // After add, auto-expand and start editing title
    setEditingTitleIndex(docs.length)  // New doc index

  handleStartEditTitle(index):
    IF isEditableTitle(docs[index].type):
      setEditingTitleIndex(index)

  handleFinishEditTitle(index, newTitle):
    onUpdateDocTitle(index, newTitle || 'Other')  // Fallback if empty
    setEditingTitleIndex(null)

  handlePromptChange(index, value):
    setPromptInputs({ ...promptInputs, [index]: value })

  handleGenerate(index):
    setIsGenerating({ ...isGenerating, [index]: true })
    // Call generate API
    // On complete: setIsGenerating({ ...isGenerating, [index]: false })

  RENDER Container (flex column):

    // Header with Add button
    RENDER SidebarHeader:
      - title: "Docs"
      - onAddClick: handleAddDoc

    // Accordion tabs
    FOR EACH (doc, index) IN docs:
      isActive = activeDocIndex === index
      isExpanded = expandedIndex === index
      isEditingTitle = editingTitleIndex === index
      canEditTitle = isEditableTitle(doc.type)
      canDelete = isDeletable(doc.type)

      RENDER DocTabItem với:
        - doc
        - index
        - isActive
        - isExpanded
        - isEditingTitle
        - canEditTitle
        - canDelete
        - promptInput: promptInputs[index] || ''
        - isGenerating: isGenerating[index] || false
        - onToggle: () => handleToggle(index)
        - onStartEditTitle: () => handleStartEditTitle(index)
        - onFinishEditTitle: (title) => handleFinishEditTitle(index, title)
        - onDelete: () => onDeleteDoc(index)
        - onPromptChange: (v) => handlePromptChange(index, v)
        - onGenerate: () => handleGenerate(index)
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

**With "Other" docs:**

```
┌────────────────────────────────┐
│ Docs                      [+]  │  ← Click [+] to add new 'other' doc
│ 📄 Brief                    >  │
│ 📄 Draft                    >  │
│ 📄 Script                   >  │
│ 📄 Research Notes       [✎] >  │  ← 'other' type, hover shows edit icon
│ 📄 Character Bio        [✎] >  │
└────────────────────────────────┘
```

**Editing Title (hover → click pencil):**

```
┌────────────────────────────────┐
│ Docs                      [+]  │
│ 📄 Brief                    >  │
│ 📄 Draft                    >  │
│ 📄 Script                   >  │
│ ┌────────────────────────────┐ │
│ │ Research Notes         [✓] │ │  ← Input mode, confirm on blur/enter
│ └────────────────────────────┘ │
└────────────────────────────────┘
```

**New Doc Added (auto edit mode):**

```
┌────────────────────────────────┐
│ Docs                      [+]  │
│ 📄 Brief                    >  │
│ 📄 Draft                    >  │
│ 📄 Script                   >  │
│ ┌────────────────────────────┐ │
│ │ Other                  [✓] │ │  ← Auto-focus, select all text
│ └────────────────────────────┘ │
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

**Mục đích:** Header với title và add button.

**Props:**

```typescript
interface SidebarHeaderProps {
  title: string;
  onAddClick: () => void;
}
```

**Visual:**

```
┌────────────────────────────────┐
│ Docs                      [+]  │
└────────────────────────────────┘
```

| Element | Notes |
|---------|-------|
| Title | "Docs" |
| Add button `[+]` | Click → add new 'other' doc |

### 3.2 DocTabItem

**Mục đích:** Accordion tab item. Expandable với PromptPanel. Supports editable title for 'other' type.

**Props & Callbacks:**

```typescript
interface DocTabItemProps {
  doc: ManuscriptDoc;
  index: number;
  isActive: boolean;
  isExpanded: boolean;
  isEditingTitle: boolean;
  canEditTitle: boolean;        // true for 'other' type
  canDelete: boolean;           // true for 'other' type
  promptInput: string;
  isGenerating: boolean;
  onToggle: () => void;
  onStartEditTitle: () => void;
  onFinishEditTitle: (title: string) => void;
  onDelete: () => void;
  onPromptChange: (value: string) => void;
  onGenerate: () => void;
}
```

**Visual:**

```
Collapsed (fixed type):
┌────────────────────────────────────────────────────┐
│ 📄 Brief                                         > │
└────────────────────────────────────────────────────┘

Collapsed (other type - hover):
┌────────────────────────────────────────────────────┐
│ 📄 Research Notes                        [✎][🗑] > │  ← Hover shows edit + delete
└────────────────────────────────────────────────────┘

Editing Title:
┌────────────────────────────────────────────────────┐
│ 📄 [Research Notes________________]          [✓] > │  ← Input, Enter/blur to save
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

| Field | Required | Store Field | Options |
|-------|----------|-------------|---------|
| TARGET AUDIENCE | ✅ | `bookSettings.targetAudience` | Constants |
| CORE VALUE | ✅ | `bookSettings.targetCoreValue` | Constants |
| FORMAT GENRE | ✅ | `bookSettings.formatGenre` | Constants |
| CONTENT GENRE | ✅ | `bookSettings.contentGenre` | Constants |
| ERA | ❌ | `bookReferences.eraId` | DB lookup (eras table) |
| LOCATION | ❌ | `bookReferences.locationId` | DB lookup (locations table) |

**Store Binding:**
- Values read from `useBookSettings()` / `useBookReferences()`
- Changes update via `updateSettings()` / `updateReferences()`
- Changes persist to BookStore → saved to DB

> **Note:** Draft, Script, Other tabs show only PROMPT field (no attributes).

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Accordion Exclusivity**
Only one tab expanded at a time. Expanding a new tab collapses the previous one and sets it as active.

**Active vs Expanded**
- `isActive`: Which doc is shown in the editor (controlled by parent)
- `isExpanded`: Which tab's PromptPanel is visible (local state)
- Expanding always sets active, but clicking outside doesn't collapse

**Fixed vs Other Doc Types**

| Behavior | Fixed (brief/draft/script) | Other |
|----------|---------------------------|-------|
| Title editable | ❌ | ✅ (hover → pencil icon) |
| Deletable | ❌ | ✅ |
| PromptPanel | Full (Brief) / Prompt only (Draft/Script) | Prompt only |

**Add New Doc Flow**
1. Click [+] → add new doc with `type: 'other'`, `title: 'Other'`
2. Auto-expand new doc
3. Auto-enter title edit mode (input focused, text selected)
4. User types new title → blur/Enter to confirm

**Attribute Fields Location**
Attribute fields (TARGET AUDIENCE, CORE VALUE, etc.) appear only in Brief tab. Draft, Script, and Other tabs show only the PROMPT textarea.

**Generate Dependency Chain**

| Doc Type | Requires | Generates |
|----------|----------|-----------|
| Brief | Attributes + Prompt | Story framework |
| Draft | Brief content + Prompt | Full narrative |
| Script | Draft content + Prompt | Scene-by-scene script |
| Other | Prompt only | Freeform content |

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
