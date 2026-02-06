# ManuscriptCreativeSpace: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           ManuscriptCreativeSpace                             │
│  ┌───────────────────────────┬─────────────────────────────────────────────┐  │
│  │   ManuscriptStepsSidebar  │              Main Content                   │  │
│  │  ┌──────────────────┐     │  ┌───────────────────────────────────────┐  │  │
│  │  │   StepList       │     │  │                                       │  │  │
│  │  │  • Brief      >  │     │  │   SWITCH activeStep:                  │  │  │
│  │  │  • Draft      >  │     │  │                                       │  │  │
│  │  │  • Script     >  │     │  │   'brief'|'draft'|'script':           │  │  │
│  │  │  • Prose Dummy > │     │  │     → ManuscriptDocEditor             │  │  │
│  │  │  • Poetry Dummy> │     │  │                                       │  │  │
│  │  │  • Finalization> │     │  │   'prose_dummy'|'poetry_dummy':       │  │  │
│  │  └──────────────────┘     │  │     → ManuscriptSpreadView (mode=dummy)│  │  │
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
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            ManuscriptCreativeSpace                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: manuscript, currentLanguage ⚡, onManuscriptUpdate                  │  │
│  ├────────────────────────────────────────────────────────────────────────────┤  │
│  │  State: activeStep, promptInput, isGenerating, selectedDummyType           │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│         │                      │                                │                │
│         ▼                      ▼                                ▼                │
│  ┌────────────────┐   ┌─────────────────┐   ┌───────────────────────────────┐   │
│  │ StepsSidebar   │   │   DocEditor     │   │   DummyView / Finalization    │   │
│  │                │   │                 │   │                               │   │
│  │ Props:         │   │ Props:          │   │ Props:                        │   │
│  │ • activeStep   │   │ • doc           │   │ • spreads                     │   │
│  │ • promptInput  │   │                 │   │ • currentLanguage ⚡           │   │
│  │ • isGenerating │   │ Callbacks:      │   │                               │   │
│  │ • selectedType │   │ • onContent     │   │ Callbacks:                    │   │
│  │                │   │   Change        │   │ • onSpreadSelect              │   │
│  │ Callbacks:     │   │                 │   │ • onSpreadAdd                 │   │
│  │ • onStepChange │   └─────────────────┘   │ • onSpreadUpdate              │   │
│  │ • onPrompt     │                         │ • onSpreadReorder             │   │
│  │   Change       │                         │                               │   │
│  │ • onGenerate   │                         │                               │   │
│  │ • onDummyType  │                         │                               │   │
│  │   Change       │                         │                               │   │
│  └────────────────┘                         └───────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step → Content Type Mapping

| Step | Content Type | Component | Description |
|------|--------------|-----------|-------------|
| `brief` | `doc` | ManuscriptDocEditor | Markdown editor cho ý tưởng truyện |
| `draft` | `doc` | ManuscriptDocEditor | Markdown editor cho bản nháp đầy đủ |
| `script` | `doc` | ManuscriptDocEditor | Markdown editor cho kịch bản scene-by-scene |
| `prose_dummy` | `spread` | ManuscriptSpreadView (mode=dummy) | Spread view + editor cho văn xuôi |
| `poetry_dummy` | `spread` | ManuscriptSpreadView (mode=dummy) | Spread view + editor cho thơ/vần |
| `finalization` | `spread` | ManuscriptSpreadView (mode=finalize) | Spread view + editor for final text adjustments |

### 1.4 manuscript{} Data Structure Reference

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

## 2. Root Component Design

### 2.1 Overview

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

### 2.2 Interface

```typescript
interface ManuscriptCreativeSpaceProps {
  manuscript: Manuscript;           // object, không phải array
  currentLanguage: Language;        // ⚡ language-aware
  onManuscriptUpdate: (manuscript: Manuscript) => void;
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
  onDummySpreadReorder: (dummyType: DummyType, oldIndex: number, newIndex: number) => void;
  onGenerateArtDirection: (sourceDummyType: DummyType, prompt: string) => Promise<void>;
}
```

### 2.3 Render Logic (pseudo)

```
ManuscriptCreativeSpace:
  RENDER ManuscriptStepsSidebar với:
    - activeStep, promptInput, isGenerating
    - selectedDummyType (visible only for finalization)
    - callbacks: onStepChange, onPromptChange, onGenerate, onDummyTypeChange

  SWITCH activeStep:
    'brief' | 'draft' | 'script':
      doc = GET doc from manuscript.docs WHERE type === activeStep
      RENDER ManuscriptDocEditor với doc, onContentChange

    'prose_dummy' | 'poetry_dummy':
      dummyType = activeStep === 'prose_dummy' ? 'prose' : 'poetry'
      dummy = GET dummy from manuscript.dummies WHERE type === dummyType
      RENDER ManuscriptSpreadView với:
        - spreads: dummy.spreads
        - mode: 'dummy'
        - currentLanguage
        - onSpreadSelect, onSpreadAdd, onSpreadUpdate, onSpreadReorder

    'finalization':
      spreads = GET snapshot.spreads[]  // Data from snapshot, NOT manuscript.dummies
      RENDER ManuscriptSpreadView với:
        - spreads (from snapshot.spreads[])
        - mode: 'finalize'
        - currentLanguage
        - onSpreadSelect, onSpreadUpdate, onSpreadReorder
        // NOTE: No onSpreadAdd in finalize mode
        // NOTE: Translation handled at EditorPage level via TranslationNotAvailableDialog
```

### 2.4 Visual

```
┌────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │ ◻ Brief          >  │  │                                              │ │
│  │ ◻ Draft          >  │  │      [Main Content Area]                     │ │
│  │ ◻ Script         >  │  │                                              │ │
│  │ ◼ Prose Dummy    ∨  │  │      - DocEditor for doc steps               │ │
│  │   ┌───────────────┐ │  │      - DummyView for dummy steps             │ │
│  │   │ PROMPT        │ │  │      - FinalizationView for final step       │ │
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
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 ManuscriptStepsSidebar

📄 **Doc:** [03-01-manuscript-steps-sidebar.md](component/editor-page/03-01-manuscript-steps-sidebar.md)

**Mục đích:** Left sidebar chứa step navigation và prompt input panel. Hiển thị danh sách các bước, cho phép chuyển đổi, và nhập prompt để generate content.

**Props & Callbacks:**

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
```

**Visual:**

```
┌──────────────────────────┐
│ Manuscript Steps         │
├──────────────────────────┤
│ 📄 Brief              >  │  ← Collapsed
├──────────────────────────┤
│ 📄 Draft              >  │
├──────────────────────────┤
│ ▦ Prose Dummy         ∨  │  ← Expanded
│  ┌────────────────────┐  │
│  │ PROMPT             │  │
│  ├────────────────────┤  │
│  │ Enter prompt...    │  │
│  ├────────────────────┤  │
│  │ ✨ Generate        │  │
│  └────────────────────┘  │
├──────────────────────────┤
│ ✨ Finalization       >  │
└──────────────────────────┘
```

---

### 3.2 ManuscriptDocEditor

📄 **Doc:** [03-02-manuscript-doc-editor.md](component/editor-page/03-02-manuscript-doc-editor.md)

**Mục đích:** Rich text/Markdown editor cho các bước doc (Brief, Draft, Script). Hỗ trợ formatting cơ bản.

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
│  B   I   U   S   ❝   ≡   1.  @   ━                                 │  ← Toolbar
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  # Manuscript                                                       │
│                                                                     │
│  The mist clung to the jagged edges of the peaks...                │
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

### 3.3 ManuscriptSpreadView

📄 **Doc:** [03-03-manuscript-spread-view.md](component/editor-page/03-03-manuscript-spread-view.md)

> **Note:** Unified component thay thế cả `ManuscriptDummyView` và `ManuscriptFinalizationView`. Used by Prose Dummy, Poetry Dummy, and Finalization steps.

**Screenshots:**
- Edit mode: `component/editor-page/screenshots/manuscript-edit-view.png`
- Grid mode: `component/editor-page/screenshots/manuscript-grid-view.png`

**Mục đích:** Unified spread view cho cả Dummy và Finalization steps. Hiển thị spread editor với horizontal filmstrip, hoặc grid view với adjustable columns.

**Data source:**
- `mode='dummy'`: `manuscript.dummies[].spreads[]` (drafting & layout planning)
- `mode='finalize'`: `snapshot.spreads[]` directly (NOT from manuscript - final assets for export pipelines)

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox text hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface ManuscriptSpreadViewProps {
  spreads: SpreadViewSpread[];
  mode: 'dummy' | 'finalize';
  currentLanguage: Language;

  onSpreadSelect?: (spreadIndex: number) => void;
  onSpreadAdd?: () => void;              // Not called in finalize mode
  onSpreadUpdate?: (spreadIndex: number, spread: SpreadViewSpread) => void;
  onSpreadReorder?: (oldIndex: number, newIndex: number) => void;
}
```

**Key Features:**

| Feature | mode='dummy' | mode='finalize' |
|---------|--------------|-----------------|
| Drag-drop reorder | ✅ Yes | ✅ Yes |
| Click to edit (inline) | ✅ Yes | ✅ Yes |
| Add spread | ✅ Button visible | ❌ No button |
| Image display | `art_note` | `visual_description` |

**View Modes:**
1. **Edit Mode** (default): SpreadCanvas + horizontal filmstrip, slider controls zoom (25%-200%)
2. **Grid Mode** (toggle): Grid view with all thumbnails, slider controls columns (1-6)

**Visual (Edit Mode):**

```
┌─────────────────────────────────────────────────────────────────────┐
│  [⚏]                                                 ─ ●──── + 100% │
│   ↑ toggle                                            └→ zoom       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│               ┌───────────────────────────────────┐                 │
│               │        SpreadEditorPanel          │                 │
│               │  ┌────────────────┬─────────────┐ │                 │
│               │  │   Left Page    │  Right Page │ │                 │
│               │  │   [Image]      │  [Textbox]  │ │                 │
│               │  │      2         │      3      │ │                 │
│               │  └────────────────┴─────────────┘ │                 │
│               └───────────────────────────────────┘                 │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────┐  ╔═════╗  ┌─────┐  ┌─────┐  ┌─────┐  ┌───────┐            │
│  │ 0-1 │  ║ 2-3 ║  │ 4-5 │  │ 6-7 │  │ 8-9 │  │  NEW  │            │
│  └─────┘  ╚═════╝  └─────┘  └─────┘  └─────┘  └───────┘            │
│              ↑ selected                        (dummy mode)         │
└─────────────────────────────────────────────────────────────────────┘
```

**Visual (Grid Mode):**

```
┌─────────────────────────────────────────────────────────────────────┐
│  [⚏] ☑ Show full spread view                         ─ ●──── +   4 │
│   ↑    ↑ checkbox                                     └→ columns   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                                │
│  │ 0-1 │  │ 2-3 │  │ 4-5 │  │ 6-7 │                                │
│  └─────┘  └─────┘  └─────┘  └─────┘                                │
│  ┌─────┐  ┌───────┐                                                │
│  │ 8-9 │  │  NEW  │   (dummy mode)                                 │
│  └─────┘  └───────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

**Child Components:**
- `SpreadViewHeader` - Header với toggle, dual-purpose slider (zoom/columns)
- `SpreadEditorPanel` - Inline editor (replaces SpreadEditModal)
- `SpreadThumbnailList` - Thumbnails (horizontal filmstrip or grid based on viewMode)

---

### 3.4 ~~ManuscriptFinalizationView~~ (DEPRECATED)

> **Deprecated:** Replaced by `ManuscriptSpreadView` với `mode='finalize'`.
>
> See [03-03-manuscript-spread-view.md](component/editor-page/03-03-manuscript-spread-view.md) for updated design.

**Migration:**
```typescript
// Before
<ManuscriptFinalizationView
  spreads={snapshotSpreads}
  currentLanguage={currentLanguage}
/>

// After
<ManuscriptSpreadView
  spreads={snapshotSpreads}
  mode="finalize"
  currentLanguage={currentLanguage}
  onSpreadSelect={handleSpreadSelect}
  onSpreadUpdate={handleSpreadUpdate}
  onSpreadReorder={handleSpreadReorder}
/>
// NOTE: Translation now handled at EditorPage level via TranslationNotAvailableDialog
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Sidebar Collapsible Steps**
Mỗi step trong sidebar có thể expand/collapse. Khi expand, hiển thị prompt input panel. Lý do: Tiết kiệm không gian, tập trung vào step đang làm việc.

**Single Active Step**
Chỉ một step được expand và active tại một thời điểm. Lý do: Đơn giản hóa UX, tránh nhầm lẫn.

**Dummy Type Selection for Finalization**
Finalization step có dropdown chọn source dummy (Prose/Poetry). Lý do: User có thể tạo cả 2 loại dummy và chọn 1 để finalize.

**Spread Grid Responsive**
`columnsPerRow` state cho phép user điều chỉnh số cột. Default 4. Lý do: Phù hợp với screen sizes khác nhau.

**Language-aware Textbox Display**
Textbox content được lấy theo `textbox[currentLanguage.code]`. Lý do: Hỗ trợ multi-language editing.

> **Note:** Translation is now handled at EditorPage level via `TranslationNotAvailableDialog`.
> See [01-04-translation-not-available-dialog.md](component/editor-page/01-04-translation-not-available-dialog.md).

### 4.2 Generate Flow

| Step | Generate Action | Output |
|------|-----------------|--------|
| Brief | AI generates story idea | `manuscript.docs[type='brief'].content` |
| Draft | AI generates full draft | `manuscript.docs[type='draft'].content` |
| Script | AI generates scene script | `manuscript.docs[type='script'].content` |
| Prose Dummy | AI generates spread layout | `manuscript.dummies[type='prose'].spreads[]` |
| Poetry Dummy | AI generates spread layout | `manuscript.dummies[type='poetry'].spreads[]` |
| Finalization | AI generates visual descriptions | `snapshot.spreads[]` (copied from dummy + visual_descriptions) |

### 4.3 Data Sync

**manuscript{} lives in snapshot**
- `manuscript` data là phần của `snapshot.manuscript` (object, không phải array)
- Changes được propagate qua `onManuscriptUpdate` callback lên EditorPage

**Finalization Output**
- Finalization đọc/ghi trực tiếp vào `snapshot.spreads[]`, KHÔNG liên quan đến `manuscript.dummies[]`
- Là bước cuối trong manuscript flow để tạo complete assets + spreads
- Output được sử dụng bởi các downstream pipelines: PDF export, ePub, Video, etc.

### 4.4 Spread Interaction (Future Design)

Khi click vào spread trong Dummy/Finalization view:
- Highlight selected spread
- Allow add text + drag/drop image and text

**Note:** Chi tiết spread editing interaction sẽ được thiết kế riêng.

### 4.5 Khi nào cần refactor?

Cân nhắc refactor nếu xuất hiện nhu cầu:
- Complex spread editing inline (không dùng modal)
- Real-time collaboration trên manuscripts
- Version history/undo-redo cho từng step
- AI streaming response hiển thị progressive content
