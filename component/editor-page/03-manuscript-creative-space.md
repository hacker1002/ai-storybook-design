# ManuscriptCreativeSpace: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           ManuscriptCreativeSpace                          │
│  ┌────────────────────────┬─────────────────────────────────────────────┐  │
│  │   ManuscriptStepsSidebar│              Main Content                  │  │
│  │  ┌──────────────────┐  │  ┌───────────────────────────────────────┐  │  │
│  │  │   StepList       │  │  │                                       │  │  │
│  │  │  • Brief      >  │  │  │   SWITCH activeStep:                  │  │  │
│  │  │  • Draft      >  │  │  │                                       │  │  │
│  │  │  • Script     >  │  │  │   'brief'|'draft'|'script':           │  │  │
│  │  │  • Prose Dummy ∨ │  │  │     → ManuscriptDocEditor             │  │  │
│  │  │  • Poetry Dummy> │  │  │                                       │  │  │
│  │  │  • Finalization> │  │  │   'prose_dummy'|'poetry_dummy':       │  │  │
│  │  └──────────────────┘  │  │     → ManuscriptDummyView             │  │  │
│  │  ┌──────────────────┐  │  │                                       │  │  │
│  │  │   PromptPanel    │  │  │   'finalization':                     │  │  │
│  │  │  ┌────────────┐  │  │  │     → ManuscriptFinalizationView      │  │  │
│  │  │  │TYPE (final)│  │  │  │                                       │  │  │
│  │  │  └────────────┘  │  │  └───────────────────────────────────────┘  │  │
│  │  │  ┌────────────┐  │  │                                             │  │
│  │  │  │  PROMPT    │  │  │                                             │  │
│  │  │  │  textarea  │  │  │                                             │  │
│  │  │  └────────────┘  │  │                                             │  │
│  │  │  ┌────────────┐  │  │                                             │  │
│  │  │  │ Generate ✨ │  │  │                                             │  │
│  │  │  └────────────┘  │  │                                             │  │
│  │  └──────────────────┘  │                                             │  │
│  └────────────────────────┴─────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
                                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                           ManuscriptCreativeSpace                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  State: activeStep, promptInput, isGenerating                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                    │                              │               │
│         ▼                    ▼                              ▼               │
│  ┌───────────────┐   ┌───────────────────┐   ┌──────────────────────────┐  │
│  │ StepsSidebar  │   │  DocEditor        │   │ DummyView/Finalization   │  │
│  │               │   │                   │   │                          │  │
│  │ Props:        │   │ Props:            │   │ Props:                   │  │
│  │ • activeStep  │   │ • doc             │   │ • dummy                  │  │
│  │ • stepConfig  │   │ • onContentChange │   │ • columnsPerRow          │  │
│  │ • promptInput │   │                   │   │ • currentLanguage ⚡      │  │
│  │ • isGenerating│   │                   │   │                          │  │
│  │               │   │                   │   │ Callbacks:               │  │
│  │ Callbacks:    │   │                   │   │ • onSpreadSelect         │  │
│  │ • onStepChange│   │                   │   │ • onSpreadAdd            │  │
│  │ • onPrompt    │   │                   │   │ • onSpreadUpdate         │  │
│  │    Change     │   │                   │   │ • onTranslate            │  │
│  │ • onGenerate  │   │                   │   │ • onGenerateArtDirection │  │
│  └───────────────┘   └───────────────────┘   └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step → Content Type Mapping

| Step | Content Type | Component | Description |
|------|--------------|-----------|-------------|
| `brief` | `doc` | ManuscriptDocEditor | Markdown editor cho ý tưởng truyện |
| `draft` | `doc` | ManuscriptDocEditor | Markdown editor cho bản nháp đầy đủ |
| `script` | `doc` | ManuscriptDocEditor | Markdown editor cho kịch bản scene-by-scene |
| `prose_dummy` | `dummy` | ManuscriptDummyView | Spread grid cho văn xuôi |
| `poetry_dummy` | `dummy` | ManuscriptDummyView | Spread grid cho thơ/vần |
| `finalization` | `dummy` | ManuscriptFinalizationView | Spread grid + Type selector + Translate |

### 1.4 manuscripts[] Data Structure Reference

```json
{
  "docs[]": [
    { "type": "brief", "content": "markdown text..." },
    { "type": "draft", "content": "markdown text..." },
    { "type": "script", "content": "markdown text..." }
  ],
  "dummies[]": [
    {
      "type": "prose",
      "spreads[]": [
        {
          "layout": "uuid",
          "left_page": { "number": 0, "type": "normal_page", "layout": "uuid" },
          "right_page": { "number": 1, "type": "normal_page", "layout": "uuid" },
          "images[]": [{ "geometry": {...}, "art_note": "..." }],
          "textboxes[]": [{ "[lang_code]": { "text": "...", "geometry": {...}, "typography": {...} } }]
        }
      ]
    },
    { "type": "poetry", "spreads[]": [...] }
  ]
}
```

---

## 2. Component Designs

### 2.1 ManuscriptCreativeSpace (Root Component)

**Mục đích:** Container chính cho manuscript editing workflow. Quản lý navigation giữa các bước, prompt input, và render content tương ứng với step.

**Shared Types:**

```typescript
type ManuscriptStepType = 'brief' | 'draft' | 'script' | 'prose_dummy' | 'poetry_dummy' | 'finalization';

type DummyType = 'prose' | 'poetry';

interface ManuscriptDoc {
  type: 'brief' | 'draft' | 'script';
  content: string;
}

interface ManuscriptDummy {
  type: DummyType;
  spreads: DummySpread[];
}

interface DummySpread {
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

interface Geometry {
  x: number;
  y: number;
  w: number;
  h: number;
  rotation?: number;
}

interface Manuscript {
  docs: ManuscriptDoc[];
  dummies: ManuscriptDummy[];
}
```

**Interface:**

```typescript
interface ManuscriptCreativeSpaceProps {
  manuscripts: Manuscript;
  currentLanguage: Language;
  onManuscriptsUpdate: (manuscripts: Manuscript) => void;
}

interface ManuscriptCreativeSpaceState {
  activeStep: ManuscriptStepType;
  promptInput: string;
  isGenerating: boolean;
  selectedDummyType: DummyType;  // For Finalization step source selection
}

interface ManuscriptCreativeSpaceCallbacks {
  onStepChange: (step: ManuscriptStepType) => void;
  onPromptChange: (prompt: string) => void;
  onGenerate: (step: ManuscriptStepType, prompt: string) => Promise<void>;
  onDocContentChange: (docType: string, content: string) => void;
  onDummyUpdate: (dummyType: DummyType, spreads: DummySpread[]) => void;
  onGenerateArtDirection: (sourceDummyType: DummyType, prompt: string) => Promise<void>;
  onTranslate: (targetLanguage: Language) => Promise<void>;
}
```

**Render Logic (pseudo):**

```
ManuscriptCreativeSpace:
  RENDER ManuscriptStepsSidebar với:
    - activeStep, promptInput, isGenerating
    - selectedDummyType (visible only for finalization)
    - callbacks: onStepChange, onPromptChange, onGenerate, onDummyTypeChange

  SWITCH activeStep:
    'brief' | 'draft' | 'script':
      doc = GET doc from manuscripts.docs WHERE type === activeStep
      RENDER ManuscriptDocEditor với doc, onContentChange

    'prose_dummy' | 'poetry_dummy':
      dummyType = activeStep === 'prose_dummy' ? 'prose' : 'poetry'
      dummy = GET dummy from manuscripts.dummies WHERE type === dummyType
      RENDER ManuscriptDummyView với dummy, currentLanguage, onSpreadUpdate

    'finalization':
      dummy = GET dummy from manuscripts.dummies WHERE type === selectedDummyType
      RENDER ManuscriptFinalizationView với:
        - dummy, currentLanguage
        - onGenerateArtDirection, onTranslate
```

---

### 2.2 ManuscriptStepsSidebar

**Mục đích:** Left sidebar chứa step navigation và prompt input panel. Hiển thị danh sách các bước, cho phép chuyển đổi, và nhập prompt để generate content.

**Interface:**

```typescript
interface ManuscriptStepsSidebarProps {
  activeStep: ManuscriptStepType;
  promptInput: string;
  isGenerating: boolean;
  selectedDummyType: DummyType;
  onStepChange: (step: ManuscriptStepType) => void;
  onPromptChange: (prompt: string) => void;
  onGenerate: () => void;
  onDummyTypeChange: (type: DummyType) => void;
}

interface ManuscriptStepsSidebarState {
  expandedStep: ManuscriptStepType | null;
}
```

**Configuration:**

```typescript
interface StepConfig {
  id: ManuscriptStepType;
  icon: string;
  label: string;
  generateLabel: string;
  showTypeSelector: boolean;
}

const MANUSCRIPT_STEPS: StepConfig[] = [
  { id: 'brief',        icon: 'FileText',  label: 'Brief',         generateLabel: 'Generate',             showTypeSelector: false },
  { id: 'draft',        icon: 'FileText',  label: 'Draft',         generateLabel: 'Generate',             showTypeSelector: false },
  { id: 'script',       icon: 'FileText',  label: 'Script',        generateLabel: 'Generate',             showTypeSelector: false },
  { id: 'prose_dummy',  icon: 'Grid',      label: 'Prose Dummy',   generateLabel: 'Generate',             showTypeSelector: false },
  { id: 'poetry_dummy', icon: 'Grid',      label: 'Poetry Dummy',  generateLabel: 'Generate',             showTypeSelector: false },
  { id: 'finalization', icon: 'Sparkles',  label: 'Finalization',  generateLabel: 'Generate Art Direction', showTypeSelector: true },
];
```

**Visual:**

```
┌──────────────────────────┐
│ Manuscript Steps         │
├──────────────────────────┤
│ 📄 Brief              >  │  ← Collapsed (chevron right)
├──────────────────────────┤
│ 📄 Draft              >  │
├──────────────────────────┤
│ 📄 Script             >  │
├──────────────────────────┤
│ ▦ Prose Dummy         ∨  │  ← Expanded (chevron down)
│  ┌────────────────────┐  │
│  │ PROMPT             │  │
│  ├────────────────────┤  │
│  │ Enter your prompt  │  │
│  │ for this          │  │
│  │ manuscript...      │  │
│  ├────────────────────┤  │
│  │ ✨ Generate        │  │  ← Blue button
│  └────────────────────┘  │
├──────────────────────────┤
│ ▦ Poetry Dummy        >  │
├──────────────────────────┤
│ ✨ Finalization       >  │
└──────────────────────────┘

Finalization Expanded:
┌──────────────────────────┐
│ ✨ Finalization       ∨  │
│  ┌────────────────────┐  │
│  │ TYPE               │  │
│  ├────────────────────┤  │
│  │ Prose           ∨  │  │  ← Dropdown
│  └────────────────────┘  │
│  ┌────────────────────┐  │
│  │ PROMPT             │  │
│  ├────────────────────┤  │
│  │ Enter your prompt  │  │
│  │ ...                │  │
│  ├────────────────────┤  │
│  │ ✨ Generate Art    │  │
│  │    Direction       │  │
│  └────────────────────┘  │
└──────────────────────────┘
```

---

### 2.3 ManuscriptDocEditor

**Mục đích:** Rich text/Markdown editor cho các bước doc (Brief, Draft, Script). Hỗ trợ formatting cơ bản.

**Interface:**

```typescript
interface ManuscriptDocEditorProps {
  doc: ManuscriptDoc | null;
  onContentChange: (content: string) => void;
}

interface ManuscriptDocEditorState {
  // Local editor state managed by editor library
}
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  B   I   U   S   ❝   ≡   1.  @   ━                                 │  ← Toolbar
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  # Manuscript                                                       │
│                                                                     │
│  The mist clung to the jagged edges of the peaks like a            │
│  tattered shroud. Below, the valley remained a secret,              │
│  whispered only in the campfire tales of the bravest nomads.        │
│                                                                     │
│  **Characters present:**                                            │
│  • Elara (The Apprentice)                                           │
│  • Malakor (The Ancient)                                            │
│                                                                     │
│  ## Scene 1: The Arrival                                            │
│                                                                     │
│  Elara stepped cautiously over the mossy stones of the             │
│  forgotten path. Her breath came in short, white puffs.            │
│                                                                     │
│  > "Do not look back, child," Malakor's voice rasped from          │
│  > the shadows of his heavy cowl. "The past here has teeth."       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Toolbar Actions:**

| Icon | Action | Markdown |
|------|--------|----------|
| B | Bold | `**text**` |
| I | Italic | `*text*` |
| U | Underline | `<u>text</u>` |
| S | Strikethrough | `~~text~~` |
| ❝ | Quote | `> text` |
| ≡ | Bullet list | `- item` |
| 1. | Numbered list | `1. item` |
| @ | Mention | `@entity_key` |
| ━ | Horizontal rule | `---` |

---

### 2.4 ManuscriptDummyView

**Mục đích:** Grid view hiển thị page spreads cho Prose Dummy và Poetry Dummy steps. Cho phép add/select/edit spreads.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox text hiển thị theo `currentLanguage.code`

**Interface:**

```typescript
interface ManuscriptDummyViewProps {
  dummy: ManuscriptDummy | null;
  currentLanguage: Language;
  onSpreadSelect: (spreadIndex: number) => void;
  onSpreadAdd: () => void;
  onSpreadUpdate: (spreadIndex: number, spread: DummySpread) => void;
}

interface ManuscriptDummyViewState {
  columnsPerRow: number;
  selectedSpreadIndex: number | null;
}
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  ─  4 / row  +                                                      │  ← Columns control
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │ 0 │ 1   │  │ 2 │ 3   │  │ 4 │ 5   │  │ 6 │ 7   │                │
│  │   │     │  │   │     │  │   │     │  │   │     │                │
│  │   │     │  │   │     │  │   │     │  │   │     │                │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘                │
│   Page 1-2     Page 3-4     Page 5-6     Page 7-8                   │
│                                                                     │
│  ┌─────────┐  ┌───────────────┐                                     │
│  │ 8 │ 9   │  │               │                                     │
│  │   │     │  │      +        │  ← New Spread button                │
│  │   │     │  │               │                                     │
│  └─────────┘  └───────────────┘                                     │
│   Page 9-10    New Spread                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Selected Spread:
┌─────────────────┐
│ ┌─────┬───────┐ │  ← Border highlight (blue)
│ │ 0   │   1   │ │
│ │     │       │ │
│ │     │       │ │
│ └─────┴───────┘ │
└─────────────────┘
```

**Spread Card Content:**

```
┌───────────────────────────┐
│ ┌──────────┬────────────┐ │
│ │ Page 0   │  Page 1    │ │
│ │ ┌──────┐ │ ┌────────┐ │ │  ← Image placeholders
│ │ │ img  │ │ │  img   │ │ │
│ │ └──────┘ │ └────────┘ │ │
│ │          │ ┌────────┐ │ │  ← Textbox preview
│ │          │ │ text...│ │ │
│ │          │ └────────┘ │ │
│ └──────────┴────────────┘ │
└───────────────────────────┘
```

---

### 2.5 ManuscriptFinalizationView

**Mục đích:** View cho Finalization step. Hiển thị spread grid từ selected dummy source, có thêm Translate button ở header. Generate Art Direction sẽ tạo visual_description và save ra snapshot.spreads[].

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox text hiển thị và translate theo `currentLanguage.code`

**Interface:**

```typescript
interface ManuscriptFinalizationViewProps {
  dummy: ManuscriptDummy | null;
  currentLanguage: Language;
  availableLanguages: Language[];
  onSpreadSelect: (spreadIndex: number) => void;
  onSpreadAdd: () => void;
  onSpreadUpdate: (spreadIndex: number, spread: DummySpread) => void;
  onTranslate: (targetLanguage: Language) => Promise<void>;
}

interface ManuscriptFinalizationViewState {
  columnsPerRow: number;
  selectedSpreadIndex: number | null;
  isTranslating: boolean;
  translateTargetLanguage: Language | null;
}
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  ─  4 / row  +                                     🌐 Translate     │  ← Header with Translate button
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │ 0 │ 1   │  │ 2 │ 3   │  │ 4 │ 5   │  │ 6 │ 7   │                │
│  │   │     │  │   │     │  │   │     │  │   │     │                │
│  │   │     │  │   │     │  │   │     │  │   │     │                │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘                │
│   Page 1-2     Page 3-4     Page 5-6     Page 7-8                   │
│                                                                     │
│  ┌─────────┐  ┌───────────────┐                                     │
│  │ 8 │ 9   │  │               │                                     │
│  │   │     │  │      +        │                                     │
│  │   │     │  │               │                                     │
│  └─────────┘  └───────────────┘                                     │
│   Page 9-10    New Spread                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Translate Button Flow:**

```
Click "Translate"
    → Show language dropdown (available languages - currentLanguage)
    → Select target language
    → Confirm dialog: "Translate all textboxes to {language}?"
    → onTranslate(targetLanguage)
    → AI translates all textboxes
    → Adds new language entry to each textbox
```

**Generate Art Direction Flow:**

```
Click "Generate Art Direction" (in sidebar)
    → isGenerating = true
    → API call: generate visual_description for each image
    → Copy dummy spreads to snapshot.spreads[]
    → Save visual_descriptions to images
    → isGenerating = false
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

**Sidebar Collapsible Steps**
Mỗi step trong sidebar có thể expand/collapse. Khi expand, hiển thị prompt input panel. Lý do: Tiết kiệm không gian, tập trung vào step đang làm việc.

**Single Active Step**
Chỉ một step được expand và active tại một thời điểm. Lý do: Đơn giản hóa UX, tránh nhầm lẫn.

**Dummy Type Selection for Finalization**
Finalization step có dropdown chọn source dummy (Prose/Poetry). Lý do: User có thể tạo cả 2 loại dummy và chọn 1 để finalize.

**Spread Grid Responsive**
`columnsPerRow` state cho phép user điều chỉnh số cột. Default 4. Lý do: Phù hợp với screen sizes khác nhau, dễ overview hoặc focus.

**Language-aware Textbox Display**
Textbox content được lấy theo `textbox[currentLanguage.code]`. Lý do: Hỗ trợ multi-language editing.

### 3.2 Generate Flow

| Step | Generate Action | Output |
|------|-----------------|--------|
| Brief | AI generates story idea | `manuscripts.docs[type='brief'].content` |
| Draft | AI generates full draft | `manuscripts.docs[type='draft'].content` |
| Script | AI generates scene script | `manuscripts.docs[type='script'].content` |
| Prose Dummy | AI generates spread layout | `manuscripts.dummies[type='prose'].spreads[]` |
| Poetry Dummy | AI generates spread layout | `manuscripts.dummies[type='poetry'].spreads[]` |
| Finalization | AI generates visual descriptions | `snapshot.spreads[]` (copied from dummy + visual_descriptions) |

### 3.3 Data Sync

**manuscripts[] lives in snapshot**
- `manuscripts` data là phần của `snapshot.manuscripts[]`
- Changes được propagate qua `onManuscriptsUpdate` callback lên EditorPage

**Finalization Output**
- Finalization step output đi vào `snapshot.spreads[]`, KHÔNG thay đổi `manuscripts.dummies[]`
- Là bước chuyển từ manuscript creativeSpace → spreads creativeSpace

### 3.4 Spread Interaction (Future Design)

Khi click vào spread trong Dummy/Finalization view:
- Highlight selected spread
- Allow add text + drag/drop image and text

**Note:** Chi tiết spread editing interaction sẽ được thiết kế riêng trong component design khác (e.g., `02-dummy-spread-editor.md`).

### 3.5 Khi nào cần refactor?

Cân nhắc refactor nếu xuất hiện nhu cầu:
- Complex spread editing inline (không dùng modal)
- Real-time collaboration trên manuscripts
- Version history/undo-redo cho từng step
- AI streaming response hiển thị progressive content
