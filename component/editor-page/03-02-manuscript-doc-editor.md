# ManuscriptDocEditor: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                      ManuscriptDocEditor                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  EditorToolbar                                             │  │
│  │  [B] [I] [U] [S] [❝] [≡] [1.] [@] [━]                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  EditorContent                                             │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  # Manuscript                                       │  │  │
│  │  │                                                     │  │  │
│  │  │  The mist clung to the jagged edges...             │  │  │
│  │  │                                                     │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                      ManuscriptDocEditor                          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Props: doc (ManuscriptDoc | null)                         │  │
│  │  Callback: onContentChange                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│         │                              │                         │
│         ▼                              ▼                         │
│  ┌───────────────┐              ┌───────────────┐                │
│  │ EditorToolbar │              │ EditorContent │                │
│  │               │              │               │                │
│  │ Props:        │              │ Props:        │                │
│  │ • editor ref  │              │ • content     │                │
│  │               │              │               │                │
│  │ Actions:      │              │ Callbacks:    │                │
│  │ • format cmds │───applies──▶│ • onChange    │                │
│  └───────────────┘              └───────────────┘                │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 Toolbar Actions Mapping

| Icon | Action | Markdown Output | Keyboard Shortcut |
|------|--------|-----------------|-------------------|
| **B** | Bold | `**text**` | Ctrl/Cmd + B |
| *I* | Italic | `*text*` | Ctrl/Cmd + I |
| <u>U</u> | Underline | `<u>text</u>` | Ctrl/Cmd + U |
| ~~S~~ | Strikethrough | `~~text~~` | Ctrl/Cmd + Shift + S |
| ❝ | Quote | `> text` | Ctrl/Cmd + Shift + Q |
| ≡ | Bullet list | `- item` | Ctrl/Cmd + Shift + 8 |
| 1. | Numbered list | `1. item` | Ctrl/Cmd + Shift + 7 |
| @ | Mention | `@entity_key` | @ |
| ━ | Horizontal rule | `---` | Ctrl/Cmd + Shift + - |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Rich text/Markdown editor cho các bước doc (Brief, Draft, Script). Cung cấp toolbar formatting và WYSIWYG editing experience.

**Shared Types:**

```typescript
interface ManuscriptDoc {
  type: 'brief' | 'draft' | 'script';
  content: string;
}

type FormatAction =
  | 'bold'
  | 'italic'
  | 'underline'
  | 'strikethrough'
  | 'quote'
  | 'bulletList'
  | 'numberedList'
  | 'mention'
  | 'horizontalRule';
```

### 2.2 Interface

```typescript
interface ManuscriptDocEditorProps {
  doc: ManuscriptDoc | null;
  onContentChange: (content: string) => void;
}

interface ManuscriptDocEditorState {
  // Editor instance (e.g., TipTap, ProseMirror, etc.)
  editor: EditorInstance | null;
}
```

### 2.3 Render Logic (pseudo)

```
ManuscriptDocEditor:
  IF doc === null:
    RENDER EmptyState "No document selected"
    RETURN

  RENDER EditorToolbar:
    - editor instance
    - format actions

  RENDER EditorContent:
    - content: doc.content
    - onChange: (newContent) => onContentChange(newContent)
```

### 2.4 Visual

**Normal State:**

```
┌─────────────────────────────────────────────────────────────────┐
│  B   I   U   S   ❝   ≡   1.   @   ━                             │  ← Toolbar
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  # Manuscript                                                    │
│                                                                  │
│  The mist clung to the jagged edges of the peaks like a         │
│  tattered shroud. Below, the valley remained a secret,          │
│  whispered only in the campfire tales of the bravest nomads.    │
│                                                                  │
│  **Characters present:**                                         │
│  • Elara (The Apprentice)                                        │
│  • Malakor (The Ancient)                                         │
│                                                                  │
│  ## Scene 1: The Arrival                                         │
│                                                                  │
│  Elara stepped cautiously over the mossy stones of the          │
│  forgotten path. Her breath came in short, white puffs.         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Empty State (no doc):**

```
┌─────────────────────────────────────────────────────────────────┐
│  B   I   U   S   ❝   ≡   1.   @   ━                             │  ← Disabled
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                                                                  │
│                   📄 No document selected                        │
│                                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**With Selection (formatting active):**

```
┌─────────────────────────────────────────────────────────────────┐
│ [B]  I   U   S   ❝   ≡   1.   @   ━                             │  ← B highlighted
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  The **|selected text|** was highlighted.                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Mention Popup (@):**

```
┌─────────────────────────────────────────────────────────────────┐
│  ... some text @mi|                                              │
│                ┌──────────────────┐                              │
│                │ 🐱 miu_cat       │                              │
│                │ 🎀 miu_bow       │                              │
│                │ 🧙 miu_wizard    │                              │
│                └──────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** EditorToolbar và EditorContent chứa các element cơ bản.
> Không cần thiết kế grandchild components.

### 3.1 EditorToolbar

**Mục đích:** Toolbar chứa các format buttons cho markdown editing.

**Elements (không phải components riêng):**

| Element | Type | Notes |
|---------|------|-------|
| Format buttons | `<button>` | Toggle formatting on selected text |
| Button group | `<div>` | Visual grouping of related buttons |

**Behavior:**
- Buttons highlight khi format đang active trên selection
- Click button → apply format to selection
- Keyboard shortcuts work anywhere in editor

### 3.2 EditorContent

**Mục đích:** Editable content area với markdown rendering.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Content editable | `<div>` | contentEditable or editor library |
| Mention popup | `<div>` | Popup list khi type @ |

**Behavior:**
- WYSIWYG editing
- Auto-save on change (debounced)
- Support paste with formatting

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Editor Library Choice**
Recommend using TipTap or ProseMirror for rich text editing. Lý do: Mature, extensible, good markdown support.

**Markdown Storage**
Content stored as markdown string, not rich text HTML. Lý do: Portable, version-friendly, AI-compatible.

**Debounced onChange**
`onContentChange` should be debounced (300ms) để tránh excessive updates. Lý do: Performance.

**Mention Entity Keys**
Mention format: `@entity_key` (e.g., `@miu_cat`). Keys are lowercase, underscore-separated. Lý do: Consistent với system convention.

### 4.2 Accessibility

```
Toolbar buttons:
  role="button"
  aria-pressed={isActive}
  aria-label="Bold" / "Italic" / etc.

Editor content:
  role="textbox"
  aria-multiline="true"
  aria-label="Manuscript content"
```

### 4.3 Keyboard Shortcuts

```typescript
const SHORTCUTS = {
  'Mod-b': 'bold',
  'Mod-i': 'italic',
  'Mod-u': 'underline',
  'Mod-Shift-s': 'strikethrough',
  'Mod-Shift-q': 'quote',
  'Mod-Shift-8': 'bulletList',
  'Mod-Shift-7': 'numberedList',
  'Mod-Shift--': 'horizontalRule',
};
// 'Mod' = Ctrl on Windows, Cmd on Mac
```
