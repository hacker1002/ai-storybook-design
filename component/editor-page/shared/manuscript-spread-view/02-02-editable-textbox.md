# EditableTextbox: Utility Component Design

> **Parent:** [SpreadEditorPanel](./02-spread-editor-panel.md)
>
> **Role:** Optional utility component. Consumer can import and use inside `renderTextItem` render prop.

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
│  │  │    <div>{text}</div>                                │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Usage Pattern

```typescript
// Consumer uses inside renderTextItem prop
<ManuscriptSpreadView
  spreads={spreads}
  renderItems={['image', 'text']}
  renderTextItem={(context) => (
    <EditableTextbox
      text={context.item.text}
      geometry={context.item.geometry}
      typography={context.item.typography}
      index={context.itemIndex}
      isSelected={context.isSelected}
      isEditable={true}
      onSelect={context.onSelect}
      onTextChange={context.onTextChange}
      onEditingChange={(editing) => { /* notify parent */ }}
    />
  )}
  // ...
/>
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

## 2. Component Design

### 2.1 Overview

**Mục đích:** Default text renderer for ManuscriptSpreadView. Handles selection và inline text editing.

> **Note:** Drag/resize handled by SelectionFrame, không phải component này.

**Shared Types:**

```typescript
interface Geometry {
  x: number;
  y: number;
  width: number;
  height: number;
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
  // Core data (flattened for easy mapping)
  text: string;
  geometry: Geometry;
  typography?: Typography;

  // State
  index: number;
  isSelected: boolean;
  isEditable: boolean;

  // Callbacks
  onSelect: () => void;
  onTextChange: (text: string) => void;
  onEditingChange: (isEditing: boolean) => void;  // Notify parent to hide handles
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

### 2.3 Typography Mapping

```typescript
mapTypographyToCSS(typography: Typography): CSSProperties
  return {
    fontFamily: typography?.fontFamily || 'inherit',
    fontSize: typography?.fontSize ? `${typography.fontSize}px` : 'inherit',
    fontWeight: typography?.fontWeight || 'normal',
    textAlign: typography?.textAlign || 'left',
    lineHeight: typography?.lineHeight || 1.5,
    color: typography?.color || 'inherit',
  }
```

### 2.4 Render Logic (pseudo)

```
EditableTextbox:
  editableRef = useRef()
  typographyStyle = mapTypographyToCSS(typography)

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
    setEditText(text)
    setIsEditing(true)
    onEditingChange(true)  // Parent hides SelectionFrame handles
    requestAnimationFrame(() => editableRef.current?.focus())

  exitEditMode(save: boolean):
    IF save && editText !== text:
      onTextChange(editText)
    setIsEditing(false)
    onEditingChange(false)  // Parent shows SelectionFrame handles

  RENDER TextboxContainer (position: absolute):
    style:
      left: geometry.x + '%'
      top: geometry.y + '%'
      width: geometry.width + '%'
      height: geometry.height + '%'
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
      IF text:
        RENDER <div>{text}</div>
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

**Editing:**

```
╔═══════════════════════════════════╗
║  Once upon a time, there was      ║
║  a brave little cat who...█       ║← cursor
║                                   ║
╚═══════════════════════════════════╝
   Blue tint background
   SelectionFrame handles hidden
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
| Utility component | Optional import | Consumer can use custom |
| Flattened props | text, geometry, typography | Easy mapping from context |
| Edit Mode | contenteditable | Seamless inline UX |
| Text Save | On blur or Enter | Natural save trigger |

### 3.2 ContentEditable Handling

```typescript
// Prevent paste with formatting
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

---
