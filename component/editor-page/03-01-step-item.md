# StepItem: Component Design

> **Parent:** [ManuscriptCreativeSpace](component/editor-page/03-manuscript-creative-space.md) (rendered inline via STEPS_CONFIG iteration)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────┐
│                           StepItem                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  StepHeader (always visible)                               │  │
│  │  [Icon] [Label]                              [ChevronIcon] │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  PromptPanel (visible when isActive)                       │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  [TYPE selector - finalization only]                 │  │  │
│  │  │  [PROMPT label]                                      │  │  │
│  │  │  [Textarea]                                          │  │  │
│  │  │  [Generate Button]                                   │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 State Diagram

```
                    ┌──────────────┐
                    │   Collapsed  │
                    │   (default)  │
                    └──────┬───────┘
                           │
                    Click Header
                           │
                           ▼
                    ┌──────────────┐
                    │   Expanded   │
                    │   (active)   │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
        Click Header             Click Another Step
              │                         │
              ▼                         ▼
       ┌──────────────┐          ┌──────────────┐
       │   Collapsed  │          │   Collapsed  │
       └──────────────┘          │ (auto by     │
                                 │  parent)     │
                                 └──────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Single step item trong sidebar. Hiển thị step header và khi active, expand để hiện PromptPanel với prompt input và generate button.

**Shared Types:**

```typescript
type ManuscriptStepType = 'brief' | 'draft' | 'script' | 'prose_dummy' | 'poetry_dummy' | 'finalization';

type DummyType = 'prose' | 'poetry';

interface StepConfig {
  id: ManuscriptStepType;
  label: string;
  icon: 'doc' | 'grid' | 'finalize';
  showDummyTypeSelector: boolean;
}

const ICON_MAP = {
  doc: '📄',      // Brief, Draft, Script
  grid: '▦',      // Prose Dummy, Poetry Dummy
  finalize: '⊛',  // Finalization
};

const GENERATE_BUTTON_LABEL = {
  brief: 'Generate',
  draft: 'Generate',
  script: 'Generate',
  prose_dummy: 'Generate',
  poetry_dummy: 'Generate',
  finalization: 'Generate Art Direction',
};
```

### 2.2 Interface

```typescript
interface StepItemProps {
  step: StepConfig;
  isActive: boolean;
  promptInput: string;
  isGenerating: boolean;
  selectedDummyType: DummyType;
  onStepClick: () => void;
  onPromptChange: (prompt: string) => void;
  onGenerate: () => void;
  onDummyTypeChange: (type: DummyType) => void;
}
```

### 2.3 Render Logic (pseudo)

```
StepItem:
  icon = ICON_MAP[step.icon]
  chevron = isActive ? '∨' : '>'
  buttonLabel = GENERATE_BUTTON_LABEL[step.id]

  RENDER StepHeader:
    - onClick: onStepClick
    - [icon] [step.label] [chevron]
    - aria-expanded: isActive

  IF isActive:
    RENDER PromptPanel:
      IF step.showDummyTypeSelector:
        RENDER TYPE label
        RENDER Select dropdown:
          - value: selectedDummyType
          - options: ['prose', 'poetry']
          - onChange: onDummyTypeChange

      RENDER PROMPT label
      RENDER Textarea:
        - value: promptInput
        - placeholder: "Enter your prompt for this manuscript..."
        - onChange: onPromptChange
        - disabled: isGenerating

      RENDER Button:
        - label: buttonLabel (or "Generating..." if isGenerating)
        - onClick: onGenerate
        - disabled: isGenerating OR promptInput.trim() === ''
        - icon: ✨ (or spinner if generating)
```

### 2.4 Visual

**Collapsed State:**

```
┌──────────────────────────────────────┐
│  📄  Brief                         > │
└──────────────────────────────────────┘
     │     │                         │
   Icon  Label                   Chevron
```

**Expanded State (Standard - Brief/Draft/Script/Dummy):**

```
┌──────────────────────────────────────┐
│  📄  Brief                         ∨ │  ← Header (clickable)
│  ┌────────────────────────────────┐  │
│  │ PROMPT                         │  │
│  │ ┌────────────────────────────┐ │  │
│  │ │ Enter your prompt for this │ │  │  ← Textarea
│  │ │ manuscript...              │ │  │
│  │ │                            │ │  │
│  │ └────────────────────────────┘ │  │
│  │                                │  │
│  │ ┌────────────────────────────┐ │  │
│  │ │      ✨ Generate           │ │  │  ← Primary Button
│  │ └────────────────────────────┘ │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Expanded State (Finalization - has TYPE selector):**

```
┌──────────────────────────────────────┐
│  ⊛  Finalization                   ∨ │
│  ┌────────────────────────────────┐  │
│  │ TYPE                           │  │
│  │ ┌────────────────────────────┐ │  │
│  │ │ Prose                    ∨ │ │  │  ← Select dropdown
│  │ └────────────────────────────┘ │  │
│  │                                │  │
│  │ PROMPT                         │  │
│  │ ┌────────────────────────────┐ │  │
│  │ │ Enter your prompt...       │ │  │
│  │ └────────────────────────────┘ │  │
│  │                                │  │
│  │ ┌────────────────────────────┐ │  │
│  │ │  ⊛ Generate Art Direction  │ │  │  ← Different label
│  │ └────────────────────────────┘ │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Generating State:**

```
┌────────────────────────────────────┐
│ ┌────────────────────────────────┐ │
│ │ [disabled textarea]            │ │
│ └────────────────────────────────┘ │
│ ┌────────────────────────────────┐ │
│ │  ⏳ Generating...              │ │  ← Disabled, spinner icon
│ └────────────────────────────────┘ │
└────────────────────────────────────┘
```

**Button Disabled (empty prompt):**

```
┌────────────────────────────────────┐
│ ┌────────────────────────────────┐ │
│ │      ✨ Generate               │ │  ← Disabled (dimmed)
│ └────────────────────────────────┘ │
└────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** StepItem chỉ chứa các element cơ bản (header, textarea, button, select).
> Không cần thiết kế thêm child component level 4.

### 3.1 Elements (không phải components riêng)

| Element | Type | Notes |
|---------|------|-------|
| StepHeader | `<button>` | Clickable row với icon, label, chevron |
| PromptPanel | `<div>` | Container cho form elements |
| TYPE label | `<label>` | "TYPE" text |
| TYPE select | `<select>` | Dropdown với options prose/poetry |
| PROMPT label | `<label>` | "PROMPT" text |
| Textarea | `<textarea>` | Multi-line input |
| Generate button | `<button>` | Primary action button |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**PromptPanel là inline section, không phải component riêng**
PromptPanel được render trực tiếp trong StepItem khi `isActive`, không tách thành component riêng vì logic đơn giản (form elements). Lý do: KISS principle.

**TYPE selector chỉ cho Finalization**
Chỉ step Finalization cần chọn source dummy type. Lý do: Các step khác không cần select source.

**Generate button disabled khi empty prompt**
Button disabled khi `promptInput.trim() === ''` hoặc `isGenerating`. Lý do: Prevent empty submissions.

**Different button labels per step**
Finalization có label khác ("Generate Art Direction") vì action khác với các steps khác. Lý do: Clear communication of action.

### 4.2 Accessibility

```
StepHeader:
  role="button"
  aria-expanded={isActive}
  tabIndex={0}
  onKeyDown: Enter/Space → onStepClick

Textarea:
  aria-label="Prompt input"
  aria-describedby="prompt-hint"

Button:
  aria-busy={isGenerating}
  aria-disabled={isDisabled}
```

### 4.3 Animation (optional)

```
PromptPanel expand/collapse:
  - max-height transition (0 → auto)
  - opacity transition (0 → 1)
  - duration: 200ms ease-out
```
