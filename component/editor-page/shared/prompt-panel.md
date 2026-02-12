# PromptPanel: Component Design

> **Parent:** [DocSidebar](../doc-creative-space/01-doc-sidebar.md), reusable across doc tabs
> **Screenshot:** `../screenshots/manuscript-docs-space.png` (PROMPT section)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                         PromptPanel                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  PromptLabel + AttachmentButton           [PROMPT] [📎]   │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  FileChipList (when files attached)                       │  │
│  │  [file1.pdf ×] [image.png ×]                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  PromptTextarea                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  GenerateButton                    [✨ Generate]          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────┐
│                       PromptPanel                            │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Props: promptInput, attachments, isGenerating         │  │
│  │  Callbacks: onPromptChange, onAttachmentsChange,       │  │
│  │             onGenerate                                 │  │
│  └────────────────────────────────────────────────────────┘  │
│              │                              │                │
│              ▼                              ▼                │
│     ┌──────────────────┐          ┌──────────────┐           │
│     │  PromptInput     │          │GenerateButton│           │
│     │  + FileChips     │          │              │           │
│     └──────────────────┘          └──────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Muc dich:** Prompt input component voi file attachments va generate button cho AI generation.

**Shared Types:**

```typescript
interface AttachedFile {
  id: string;
  name: string;           // Original filename
  size: number;           // File size in bytes
  type: string;           // MIME type
  file: File;             // File object for upload
}
```

### 2.2 Interface

**Props & Callbacks:**

```typescript
interface PromptPanelProps {
  promptInput: string;
  attachments: AttachedFile[];
  isGenerating: boolean;
  onPromptChange: (value: string) => void;
  onAttachmentsChange: (files: AttachedFile[]) => void;
  onGenerate: () => void;
}
```

### 2.3 Configuration

```typescript
// File attachment constraints
const FILE_CONSTRAINTS = {
  maxFiles: 5,
  maxSizeBytes: 10 * 1024 * 1024,  // 10MB
  acceptedTypes: [
    'image/jpeg', 'image/png', 'image/gif', 'image/webp',
    'application/pdf', 'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'text/plain'
  ],
  acceptedExtensions: '.jpg,.jpeg,.png,.gif,.webp,.pdf,.doc,.docx,.txt',
};

// Filename truncation
const MAX_FILENAME_LENGTH = 15;
```

### 2.4 Render Logic (pseudo)

```
PromptPanel:
  { promptInput, attachments, isGenerating, onPromptChange, onAttachmentsChange, onGenerate } = props

  handleAddFiles(newFiles):
    validFiles = filterValidFiles(newFiles, FILE_CONSTRAINTS)
    combined = [...attachments, ...validFiles].slice(0, FILE_CONSTRAINTS.maxFiles)
    onAttachmentsChange(combined)

  handleRemoveFile(fileId):
    onAttachmentsChange(attachments.filter(f => f.id !== fileId))

  truncateFilename(name):
    IF name.length > MAX_FILENAME_LENGTH:
      RETURN name.slice(0, MAX_FILENAME_LENGTH - 3) + '...'
    RETURN name

  RENDER Container:
    // Prompt label + attachment button
    RENDER PromptLabel:
      - text: "PROMPT"
      - attachmentButton: onClick → open file picker

    // File chips (if any)
    IF attachments.length > 0:
      RENDER FileChipList:
        FOR EACH file IN attachments:
          RENDER FileChip với:
            - name: truncateFilename(file.name)
            - fullName: file.name (tooltip)
            - onRemove: () => handleRemoveFile(file.id)

    // Textarea
    RENDER PromptTextarea với:
      - value: promptInput
      - placeholder: "Enter your prompt for this manuscript..."
      - onChange: onPromptChange
      - disabled: isGenerating

    // Generate button
    RENDER GenerateButton với:
      - loading: isGenerating
      - disabled: isGenerating
      - onClick: onGenerate
```

### 2.5 Visual

**Default State (no attachments):**

```
┌──────────────────────────────────────────┐
│ PROMPT                               [📎]│
│ ┌──────────────────────────────────────┐ │
│ │ Enter your prompt for this           │ │
│ │ manuscript...                        │ │
│ └──────────────────────────────────────┘ │
│ ┌──────────────────────────────────────┐ │
│ │           ✨ Generate                │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**With Attachments:**

```
┌──────────────────────────────────────────┐
│ PROMPT                               [📎]│
│ ┌──────────────────────────────────────┐ │
│ │ ┌────────────────┐ ┌────────────────┐│ │
│ │ │📄 design.pdf [×]│ │📄 charac... [×]│ │
│ │ └────────────────┘ └────────────────┘│ │
│ │ ┌────────────────┐                   │ │
│ │ │📄 notes.txt [×] │                  │ │
│ │ └────────────────┘                   │ │
│ └──────────────────────────────────────┘ │
│ ┌──────────────────────────────────────┐ │
│ │ Generate a magical story about...    │ │
│ │                                      │ │
│ └──────────────────────────────────────┘ │
│ ┌──────────────────────────────────────┐ │
│ │           ✨ Generate                │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**Generating State:**

```
┌──────────────────────────────────────────┐
│ PROMPT                               [📎]│
│ ┌──────────────────────────────────────┐ │
│ │ Generate a magical story about...    │ │
│ └──────────────────────────────────────┘ │
│ ┌──────────────────────────────────────┐ │
│ │        ⏳ Generating...              │ │  ← Disabled, loading spinner
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

---

## 3. Sub-elements

> **Note:** These are inline elements, not separate child components.

### 3.1 FileChip

**Muc dich:** Display attached file with remove button.

**Visual:**

```
┌─────────────────────────┐
│ 📄 filename.pdf      [×]│
└─────────────────────────┘
     │
     └─ Max 15 chars, then "..." (e.g., "character_des...")
        Hover shows full filename tooltip
```

**Behavior:**

| Action | Result |
|--------|--------|
| Hover | Show full filename tooltip |
| Click [×] | Remove file from list |

### 3.2 AttachmentButton

**Muc dich:** Trigger file picker for attachments.

**Visual:**

```
[📎]  ← Paperclip icon button
```

**Behavior:**

| Action | Result |
|--------|--------|
| Click | Open file picker (multi-select) |
| Select files | Validate & add to attachments |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Shared Component**
PromptPanel is placed in `shared/` because it's reused across all doc tabs (Brief, Draft, Script, Other).

**File Attachment UX**
- Files displayed as horizontal-wrap chips above textarea
- Truncate long filenames for clean UI, full name on hover
- Max 5 files, 10MB each to prevent abuse

### 4.2 Supported File Types

| Category | Extensions | MIME Types |
|----------|------------|------------|
| Images | `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp` | `image/*` |
| Documents | `.pdf`, `.doc`, `.docx`, `.txt` | `application/pdf`, `application/msword`, `text/plain` |

### 4.4 Layout Constants

| Element | Value |
|---------|-------|
| Panel padding | 12px |
| Textarea min-height | 80px |
| Generate button height | 36px |
| File chip gap | 8px |
| Max filename display | 15 chars |

### 4.5 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Attachment button | `button` | `aria-label="Attach files"` |
| File chip remove | `button` | `aria-label="Remove {filename}"` |
| Textarea | `textbox` | `aria-label="Prompt input"` |
| Generate button | `button` | `aria-busy` when loading |
