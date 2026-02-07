# ManuscriptCreativeSpace: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           ManuscriptCreativeSpace                             │
│  ┌───────────────────────────┬─────────────────────────────────────────────┐  │
│  │   Steps Sidebar (inline)  │              Main Content                   │  │
│  │  ┌──────────────────┐     │  ┌───────────────────────────────────────┐  │  │
│  │  │   StepItem[]     │     │  │                                       │  │  │
│  │  │  • Brief      >  │     │  │   SWITCH activeStep:                  │  │  │
│  │  │  • Draft      >  │     │  │                                       │  │  │
│  │  │  • Script     >  │     │  │   'brief'|'draft'|'script':           │  │  │
│  │  │  • Prose Dummy > │     │  │     → ManuscriptDocEditor             │  │  │
│  │  │  • Poetry Dummy> │     │  │                                       │  │  │
│  │  │  • Finalization> │     │  │   'prose_dummy'|'poetry_dummy':       │  │  │
│  │  └──────────────────┘     │  │     → ManuscriptSpreadView (mode=dummy)│ │  │
│  │                           │  │                                       │  │  │
│  │                           │  │   'finalization':                     │  │  │
│  │                           │  │     → ManuscriptSpreadView (mode=finalize)│ │
│  │                           │  │                                       │  │  │
│  │                           │  └───────────────────────────────────────┘  │  │
│  └───────────────────────────┴─────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                      ┌─────────────────────┐
                      │    SnapshotStore    │
                      │   (Zustand global)  │
                      └──────────┬──────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │ (selectors)           │                       │ (actions)
         ▼                       ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            ManuscriptCreativeSpace                           │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: currentLanguage ⚡                                              │  │
│  │                                                                        │  │
│  │  Local State: activeStep, promptInput, isGenerating, selectedDummyType │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│         │                      │                                │            │
│         ▼                      ▼                                ▼            │
│  ┌────────────────┐   ┌─────────────────┐   ┌───────────────────────────┐    │
│  │ StepItem[]     │   │ DocEditor       │   │ ManuscriptSpreadView      │    │
│  │ (inline)       │   │                 │   │                           │    │
│  │                │   │ Props:          │   │ Props:                    │    │
│  │ Props:         │   │ • doc           │   │ • mode                    │    │
│  │ • step         │   │                 │   │ • dummyType (dummy mode)  │    │
│  │ • isActive     │   │ Callbacks:      │   │ • currentLanguage ⚡       │    │
│  │ • promptInput  │   │ • onContent     │   │                           │    │
│  │ • isGenerating │   │   Change        │   │ (uses store internally)   │    │
│  │                │   │                 │   │                           │    │
│  │ Callbacks:     │   └─────────────────┘   └───────────────────────────┘    │
│  │ • onStepClick  │                                                          │
│  │ • onPrompt     │                                                          │
│  │   Change       │                                                          │
│  │ • onGenerate   │                                                          │
│  │ • onDummyType  │                                                          │
│  │   Change       │                                                          │
│  └────────────────┘                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step → Content Type Mapping

| Step | Content Type | Component | Description |
|------|--------------|-----------|-------------|
| `brief` | `doc` | ManuscriptDocEditor | Markdown editor cho ý tưởng |
| `draft` | `doc` | ManuscriptDocEditor | Markdown editor cho bản nháp |
| `script` | `doc` | ManuscriptDocEditor | Markdown editor cho kịch bản |
| `prose_dummy` | `spread` | ManuscriptSpreadView (mode=dummy) | Spread view cho văn xuôi |
| `poetry_dummy` | `spread` | ManuscriptSpreadView (mode=dummy) | Spread view cho thơ/vần |
| `finalization` | `spread` | ManuscriptSpreadView (mode=finalize) | Final text adjustments |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Container chính cho manuscript editing workflow. Quản lý navigation giữa các bước, prompt input, và render content tương ứng với step.

**Shared Types:**

```typescript
type ManuscriptStepType = 'brief' | 'draft' | 'script' | 'prose_dummy' | 'poetry_dummy' | 'finalization';
type DummyType = 'prose' | 'poetry';
type DocType = 'brief' | 'draft' | 'script';

interface ManuscriptDoc {
  type: DocType;
  content: string;
}

interface DummySpread {
  id: string;
  layout: string | null;
  left_page: DummyPage;
  right_page: DummyPage;
  images: DummyImage[];
  textboxes: DummyTextbox[];
}

interface DummyPage {
  number: number;
  type: string;
  layout: string | null;
}

interface DummyImage {
  geometry: Geometry;
  art_note: string;
  visual_description?: string;
}

interface DummyTextbox {
  [languageCode: string]: {
    text: string;
    geometry: Geometry;
    typography: Typography;
  };
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface ManuscriptCreativeSpaceProps {
  currentLanguage: Language;  // ⚡ language-aware
}

interface ManuscriptCreativeSpaceState {
  activeStep: ManuscriptStepType;
  promptInput: string;
  isGenerating: boolean;
  selectedDummyType: DummyType;  // For Finalization step source selection
}

const STEPS_CONFIG: StepConfig[] = [
  { id: 'brief', label: 'Brief', icon: 'doc', showDummyTypeSelector: false },
  { id: 'draft', label: 'Draft', icon: 'doc', showDummyTypeSelector: false },
  { id: 'script', label: 'Script', icon: 'doc', showDummyTypeSelector: false },
  { id: 'prose_dummy', label: 'Prose Dummy', icon: 'grid', showDummyTypeSelector: false },
  { id: 'poetry_dummy', label: 'Poetry Dummy', icon: 'grid', showDummyTypeSelector: false },
  { id: 'finalization', label: 'Finalization', icon: 'finalize', showDummyTypeSelector: true },
];
```

**Store Integration:**

```typescript
// State Selectors
manuscript = useManuscript();
doc = useDoc(activeStep);           // For doc steps
dummySpreads = useDummySpreads(dummyType);  // For dummy mode
spreads = useSpreads();             // For finalize mode

// Actions
{ updateDoc } = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
ManuscriptCreativeSpace:
  manuscript = useManuscript()
  { updateDoc } = useSnapshotActions()

  // Sidebar (inline)
  RENDER aside.manuscript-steps-sidebar:
    FOR EACH step IN STEPS_CONFIG:
      RENDER StepItem với step, isActive, promptInput, callbacks

  // Main content area
  SWITCH activeStep:
    'brief' | 'draft' | 'script':
      doc = GET doc WHERE type === activeStep
      RENDER ManuscriptDocEditor với doc, onContentChange

    'prose_dummy' | 'poetry_dummy':
      dummyType = activeStep === 'prose_dummy' ? 'prose' : 'poetry'
      RENDER ManuscriptSpreadView với mode='dummy', dummyType, currentLanguage

    'finalization':
      RENDER ManuscriptSpreadView với mode='finalize', currentLanguage
```

### 2.4 Visual

```
┌────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │ ◻ Brief          >  │  │                                              │ │
│  │ ◻ Draft          >  │  │      [Main Content Area]                     │ │
│  │ ◻ Script         >  │  │                                              │ │
│  │ ◼ Prose Dummy    ∨  │  │      - DocEditor for doc steps               │ │
│  │   ┌───────────────┐ │  │      - SpreadView for dummy steps            │ │
│  │   │ PROMPT        │ │  │      - SpreadView for finalization           │ │
│  │   │ ...           │ │  │                                              │ │
│  │   │ [Generate ✨] │ │  │                                              │ │
│  │   └───────────────┘ │  │                                              │ │
│  │ ◻ Poetry Dummy   >  │  │                                              │ │
│  │ ◻ Finalization   >  │  │                                              │ │
│  └─────────────────────┘  └──────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết sẽ được thiết kế trong file riêng của component đó.

### 3.1 StepItem

📄 **Doc:** [`component/editor-page/03-01-step-item.md`](component/editor-page/03-01-step-item.md)

**Mục đích:** Single step row với collapsible PromptPanel.

**Props & Callbacks:**

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

**Visual:**

```
Collapsed:                      Expanded:
┌─────────────────────────┐     ┌─────────────────────────┐
│ 📄 Brief             >  │     │ 📄 Brief             ∨  │
└─────────────────────────┘     │  ┌────────────────────┐ │
                                │  │ PROMPT             │ │
                                │  │ [textarea]         │ │
                                │  │ [✨ Generate]      │ │
                                │  └────────────────────┘ │
                                └─────────────────────────┘
```

---

### 3.2 ManuscriptDocEditor

📄 **Doc:** [`component/editor-page/03-02-manuscript-doc-editor.md`](component/editor-page/03-02-manuscript-doc-editor.md)

**Mục đích:** Rich text/Markdown editor cho các bước doc (Brief, Draft, Script).

**Props & Callbacks:**

```typescript
interface ManuscriptDocEditorProps {
  doc: ManuscriptDoc | null;
  onContentChange: (content: string) => void;
}
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  B   I   U   S   ❝   ≡   1.  @   ━                                  │  ← Toolbar
├─────────────────────────────────────────────────────────────────────┤
│  # Manuscript                                                       │
│  The mist clung to the jagged edges of the peaks...                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 3.3 ManuscriptSpreadView

📄 **Doc:** [`component/editor-page/03-03-manuscript-spread-view.md`](component/editor-page/03-03-manuscript-spread-view.md)

**Screenshots:**
- Edit mode: `component/editor-page/screenshots/manuscript-edit-view.png`
- Grid mode: `component/editor-page/screenshots/manuscript-grid-view.png`

**Mục đích:** Unified spread view cho Dummy và Finalization steps. Edit mode với horizontal filmstrip, hoặc grid view.

**Special Impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface ManuscriptSpreadViewProps {
  mode: 'dummy' | 'finalize';
  dummyType?: DummyType;        // Required for dummy mode
  currentLanguage: Language;    // ⚡ language-aware
  // No spreads prop - uses store selectors
  // No callbacks - uses store actions directly
}
```

**Store Integration (inside component):**

```typescript
// Data source based on mode
const spreads = mode === 'dummy'
  ? useDummySpreads(dummyType)
  : useSpreads();

// Actions
const { updateDummySpread, updateSpread, ... } = useSnapshotActions();
```

**Visual (Edit Mode):**

```
┌─────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                 ─ ●──── + 100% │
├─────────────────────────────────────────────────────────────────────┤
│               ┌───────────────────────────────────┐                 │
│               │        SpreadEditorPanel          │                 │
│               │  ┌────────────────┬─────────────┐ │                 │
│               │  │   Left Page    │  Right Page │ │                 │
│               │  └────────────────┴─────────────┘ │                 │
│               └───────────────────────────────────┘                 │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────┐  ╔═════╗  ┌─────┐  ┌─────┐  ┌───────┐                      │
│  │ 0-1 │  ║ 2-3 ║  │ 4-5 │  │ 6-7 │  │  NEW  │ (dummy only)         │
│  └─────┘  ╚═════╝  └─────┘  └─────┘  └───────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**SnapshotStore Integration**
Data via selectors (`useManuscript()`, `useDummySpreads()`, `useSpreads()`), mutations via `useSnapshotActions()`. ManuscriptSpreadView uses store directly - no props drilling.

**Sidebar Collapsible Steps**
Mỗi step có thể expand/collapse, hiển thị prompt input khi expand. Lý do: Tiết kiệm không gian.

**Single Active Step**
Chỉ một step được active tại một thời điểm. Lý do: Đơn giản hóa UX.

**Language-aware Textbox**
Textbox content lấy theo `textbox[currentLanguage.code]`. Translation handled at EditorPage level via `TranslationNotAvailableDialog`.

### 4.2 Generate Flow

| Step | Generate Action | Output Location |
|------|-----------------|-----------------|
| Brief | AI generates story idea | `manuscript.docs[type='brief']` |
| Draft | AI generates full draft | `manuscript.docs[type='draft']` |
| Script | AI generates scene script | `manuscript.docs[type='script']` |
| Prose Dummy | AI generates spread layout | `manuscript.dummies[type='prose'].spreads[]` |
| Poetry Dummy | AI generates spread layout | `manuscript.dummies[type='poetry'].spreads[]` |
| Finalization | AI generates visual descriptions | `snapshot.spreads[]` |

### 4.3 Data Sync

**manuscript{} in SnapshotStore**
- Changes via: `updateDoc()`, `updateDummySpread()`, etc.
- Single source of truth - no callbacks to parent

**Finalization Output**
- Reads/writes via `useSpreads()` và `updateSpread()`
- Separate from `manuscript.dummies[]` - final assets for export

**Autosave**
- `useIsDirty()` tracks changes
- `useAutosave()` debounces save (default 60s)

### 4.4 Related Docs

- Store Design: [snapshot-store.md](component/stores/snapshot-store.md)
- EditorPage: [00-editor-page.md](component/editor-page/00-editor-page.md)
