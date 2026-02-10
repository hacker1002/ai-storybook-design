# EditableTextbox: Component Design

> **Parent:** [SpreadEditorPanel](./02-spread-editor-panel.md)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        EditableTextbox                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    TextboxContainer                       │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  IF isEditing:                                      │  │  │
│  │  │    <div contentEditable>{editText}</div>            │  │  │
│  │  │  ELSE:                                              │  │  │
│  │  │    <div>{textbox.text}</div>                        │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        EditableTextbox                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Props: textbox, index, isSelected, isEditable             │ │
│  │  Callbacks: onSelect, onTextChange                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│              ┌───────────────┼───────────────┐                  │
│              ▼                               ▼                  │
│       ┌─────────────┐                 ┌─────────────┐           │
│       │ Local State │                 │ Typography  │           │
│       │ • isEditing │                 │  Renderer   │           │
│       │ • editText  │                 │             │           │
│       │ • isHovered │                 │ fontFamily, │           │
│       └─────────────┘                 │ fontSize,   │           │
│                                       │ textAlign...│           │
│                                       └─────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Mode Transitions

```
                    ┌─────────────────┐
                    │    NORMAL       │
                    │  (not selected) │
                    └────────┬────────┘
                             │ click
                             ▼
                    ┌─────────────────┐
      ESC/blur      │    SELECTED     │
    ┌───────────────│  (can be moved/ │
    │               │   resized via   │
    │               │  SelectionFrame)│
    │               └────────┬────────┘
    │                        │ double-click / Enter
    │                        ▼
    │               ┌─────────────────┐
    └───────────────│    EDITING      │
                    │ (contenteditable│
                    │   active)       │
                    └─────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Editable textbox trong canvas. Handles selection và inline text editing.

> **Note:** Drag/resize được handle bởi SelectionFrame overlay, không phải component này.

**Shared Types:**

```typescript
interface TextboxContent {
  text: string;
  geometry: Geometry;
  typography: Typography;
}

interface Typography {
  fontFamily?: string;
  fontSize?: number;
  fontWeight?: number;
  textAlign?: 'left' | 'center' | 'right';
  lineHeight?: number;
  color?: string;
}
```

### 2.2 Interface

**Props:**

```typescript
interface EditableTextboxProps {
  textbox: SpreadViewTextbox;
  index: number;
  isSelected: boolean;
  isEditable: boolean;

  onSelect: () => void;
  onTextChange: (text: string) => void;
  onEditingChange: (isEditing: boolean) => void;  // Notify parent to hide SelectionFrame handles
  // NO onDrag/onResize - handled by SelectionFrame
}
```

**Local State:**

```typescript
interface EditableTextboxState {
  isEditing: boolean;
  isHovered: boolean;
  editText: string;            // Local edit buffer
}
```

**Store Integration:**

```typescript
// No store access needed - parent provides pre-extracted content for current language
```

### 2.3 Typography Mapping

```typescript
mapTypographyToCSS(typography: Typography): CSSProperties
  return {
    fontFamily: typography.fontFamily || 'inherit',
    fontSize: typography.fontSize ? `${typography.fontSize}px` : 'inherit',
    fontWeight: typography.fontWeight || 'normal',
    textAlign: typography.textAlign || 'left',
    lineHeight: typography.lineHeight || 1.5,
    color: typography.color || 'inherit',
  }
```

### 2.4 Render Logic (pseudo)

```
EditableTextbox:
  editableRef = useRef()
  typographyStyle = mapTypographyToCSS(textbox.typography)

  handleClick(e):
    e.stopPropagation()
    IF isEditable && !isEditing:
      onSelect()

  handleDoubleClick(e):
    e.stopPropagation()
    IF isEditable && isSelected:
      enterEditMode()

  handleKeyDown(e):
    IF isSelected && !isEditing:
      IF e.key === 'Enter':
        enterEditMode()

    IF isEditing:
      IF e.key === 'Escape':
        exitEditMode(save: false)

  handleBlur():
    IF isEditing:
      exitEditMode(save: true)

  enterEditMode():
    setEditText(textbox.text)
    setIsEditing(true)
    onEditingChange(true)  // Parent hides SelectionFrame handles
    requestAnimationFrame(() => editableRef.current?.focus())

  exitEditMode(save: boolean):
    IF save && editText !== textbox.text:
      onTextChange(editText)
    setIsEditing(false)
    onEditingChange(false)  // Parent shows SelectionFrame handles

  RENDER TextboxContainer (position: absolute):
    style:
      left: textbox.geometry.x + '%'
      top: textbox.geometry.y + '%'
      width: textbox.geometry.width + '%'
      height: textbox.geometry.height + '%'
      cursor: isEditable ? 'pointer' : 'default'
      outline: isHovered && !isSelected ? '1px dashed #bdbdbd' : 'none'
      ...typographyStyle

    onClick: handleClick
    onDoubleClick: handleDoubleClick
    onKeyDown: handleKeyDown
    onMouseEnter: () => setHovered(true)
    onMouseLeave: () => setHovered(false)

    IF isEditing:
      RENDER <div
        ref={editableRef}
        contentEditable
        suppressContentEditableWarning
        onInput: (e) => setEditText(e.target.innerText)
        onBlur: handleBlur
        style: { background: 'rgba(33, 150, 243, 0.05)' }
      >
        {editText}
      </div>
    ELSE:
      IF textbox.text:
        RENDER <div>{textbox.text}</div>
      ELSE:
        RENDER <div style={{ fontStyle: 'italic', color: '#9e9e9e' }}>
          Click to add text
        </div>
```

### 2.5 Visual

**Normal (not selected):**

```
┌─────────────────────────────────┐
│  Once upon a time, there was    │
│  a brave little cat who...      │
│                                 │
└─────────────────────────────────┘
   transparent border, typography applied
```

**Hovered:**

```
┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐
┆  Once upon a time, there was    ┆
┆  a brave little cat who...      ┆
┆                                 ┆
└┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
   dashed outline (hover hint)
```

**Selected (not editing):**

```
╔═══════════════════════════════════╗
║  Once upon a time, there was      ║
║  a brave little cat who...        ║
║                                   ║
╚═══════════════════════════════════╝
   SelectionFrame rendered by parent
   Can drag/resize via SelectionFrame handles
```

**Editing:**

```
╔═══════════════════════════════════╗
║  Once upon a time, there was      ║
║  a brave little cat who...█       ║← cursor
║                                   ║
╚═══════════════════════════════════╝
   Blue tint background
   SelectionFrame handles hidden
   Text is contenteditable
```

**Empty Text:**

```
┌─────────────────────────────────┐
│  [Click to add text]            │← placeholder (italic, gray)
│                                 │
└─────────────────────────────────┘
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Edit Mode | contenteditable | Seamless inline UX, no textarea overlay |
| Text Save | On blur or Enter | Natural save trigger |
| Placeholder | Inline text | Clear affordance |
| Drag/Resize | Delegated to SelectionFrame | Single interaction handler |

### 3.2 ContentEditable Handling

```typescript
// Prevent paste with formatting - use Selection API (execCommand deprecated)
handlePaste(e):
  e.preventDefault()
  text = e.clipboardData.getData('text/plain')
  // Insert via Selection API
  selection = window.getSelection()
  range = selection.getRangeAt(0)
  range.deleteContents()
  range.insertNode(document.createTextNode(text))
  range.collapse(false)

// Enter key: save and exit. Shift+Enter: newline
handleKeyDown(e):
  IF e.key === 'Enter' && !e.shiftKey:
    e.preventDefault()
    exitEditMode(save: true)
  // Shift+Enter allows default behavior (newline)
```

### 3.3 Constants

```typescript
const TEXT_CHANGE_DEBOUNCE = 300;   // ms
const PLACEHOLDER_TEXT = 'Click to add text';
const EDIT_MODE_BG = 'rgba(33, 150, 243, 0.05)';
const PLACEHOLDER_COLOR = '#9e9e9e';
```

### 3.4 Accessibility

```typescript
<div
  role="textbox"
  aria-label="Textbox content"
  aria-multiline="true"
  tabIndex={isEditable ? 0 : -1}
>
```

### 3.5 Performance

- Debounce `onTextChange` (300ms)
- Use `white-space: pre-wrap` for proper line breaks

---
