# EditorPage: Component Design

**Screenshots:**
- Manuscript: `screenshots/manuscript-docs-space.png`, `screenshots/manuscript-dummy-space.png`, `screenshots/manuscript-sketch-space.png`
- Illustration: `screenshots/Illustration-character-space.png`, `screenshots/Illustration-prop-space.png`, `screenshots/Illustration-stage-space.png`, `screenshots/Illustration-spread-space.png`
- Retouch: `screenshots/Retouch-object-space.png`, `screenshots/Retouch-animation-space.png`, `screenshots/Retouch-remix-space.png`
- Default: `screenshots/history-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                         │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                           EditorHeader                                    │  │
│  │  ┌────────┬─────────────┬──────────────────┬────────┬────────┬──────┐     │  │
│  │  │MenuBtn │ BookTitle   │ StepBreadcrumb   │SaveStat│LangSel │Notif │     │  │
│  │  └────────┴─────────────┴──────────────────┴────────┴────────┴──────┘     │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌────────┬───────────────────────────────────────────────────┬──────────────┐  │
│  │        │                                                   │              │  │
│  │        │  Conditional Render by activeCreativeSpace:       │   Right      │  │
│  │        │  ┌─────────────────────────────────────────────┐  │   Sidebar    │  │
│  │  Icon  │  │ MANUSCRIPT STEP:                            │  │     (AI)     │  │
│  │  Rail  │  │   DocCreativeSpace        (if doc)          │  │              │  │
│  │        │  │   DummyCreativeSpace      (if dummy)        │  │  ┌────────┐  │  │
│  │        │  │   SketchCreativeSpace     (if sketch)       │  │  │   X    │  │  │
│  │        │  │                                             │  │  │ close  │  │  │
│  │        │  │ ILLUSTRATION STEP:                          │  │  └────────┘  │  │
│  │        │  │   CharactersCreativeSpace (if character)    │  │              │  │
│  │        │  │   PropsCreativeSpace      (if prop)         │  │              │  │
│  │        │  │   StagesCreativeSpace     (if stage)        │  │              │  │
│  │        │  │   SpreadsCreativeSpace    (if spread) ⚡     │  │              │  │
│  │        │  │                                             │  │              │  │
│  │        │  │ RETOUCH STEP:                               │  │              │  │
│  │        │  │   ObjectsCreativeSpace    (if object)       │  │              │  │
│  │        │  │   AnimationsCreativeSpace (if animation) ⚡  │  │              │  │
│  │        │  │   RemixCreativeSpace      (if remix) ⚡      │  │              │  │
│  │        │  │                                             │  │              │  │
│  │        │  │ DEFAULT (all steps):                        │  │              │  │
│  │        │  │   HistoryCreativeSpace    (if history) ⚡    │  │              │  │
│  │        │  │   IssuesCreativeSpace     (if issue)        │  │              │  │
│  │        │  │   SharesCreativeSpace     (if share)        │  │              │  │
│  │        │  │   CollaboratorsCreativeSpace (if collaborator) │              │  │
│  │        │  │   ConfigCreativeSpace     (if setting)      │  │              │  │
│  │        │  └─────────────────────────────────────────────┘  │              │  │
│  └────────┴───────────────────────────────────────────────────┴──────────────┘  │
│                                                                                 │
│                                                        ┌─────────────────────┐  │
│                                                        │ 💬 AISidebarToggle  │  │
│                                                        │  (floating button)  │  │
│                                                        │  bottom-right       │  │
│                                                        └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘

⚡ = CreativeSpaces affected by currentLanguage
```

**Lưu ý:** Không có component MainCreativeSpace trung gian. EditorPage render trực tiếp creativeSpace tương ứng với `activeCreativeSpace`.

### 1.2 Data Flow

```
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
        ┌──────────────────────────────────┼──────────────────────────────┐
        │                                  │                              │
        ▼                                  ▼                              ▼
┌────────────────────┐         ┌───────────────────────┐        ┌───────────────┐
│   SnapshotStore    │         │ EditorSettingsStore   │        │  EditorPage   │
│ (Zustand global)   │         │ (Zustand global)      │        │ (local state) │
│                    │         │                       │        │               │
│ • manuscript       │         │ • currentLanguage ⚡   │        │ • book        │
│ • spreads          │         │ • currentStep         │        │ • flags       │
│ • characters       │         │                       │        │ • shareLinks  │
│ • props            │         │ Actions:              │        │ • collaborations│
│ • stages           │         │ • setCurrentLanguage  │        │ • activeCreativeSpace│
│ • meta (sync)      │         │ • setCurrentStep      │        │ • isSidebarOpen│
└────────┬───────────┘         │ • resetSettings       │        └───────┬───────┘
         │                     └───────────┬───────────┘                │
         │                                 │                            │
         │  ┌──────────────────────────────┘                            │
         │  │ (selectors: useCurrentLanguage, useCurrentStep)           │
         │  │                                       ┌───────────────────┘
         │  │                                       │ (props: activeCreativeSpace, etc.)
         ▼  ▼                                       ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │  CreativeSpaces (use selectors directly from both stores)                 │
  │  ┌─────────────────────────────────────────────────────────────────────┐  │
  │  │ • DocCreativeSpace         → useDocs()                              │  │
  │  │ • DummyCreativeSpace       → useDummies()                           │  │
  │  │ • SketchCreativeSpace      → useSketch()                            │  │
  │  │ • CharactersCreativeSpace  → useCharacters(), useCurrentStep()      │  │
  │  │ • PropsCreativeSpace       → useProps(), useCurrentStep()           │  │
  │  │ • StagesCreativeSpace      → useStages(), useCurrentStep()          │  │
  │  │ • SpreadsCreativeSpace ⚡   → useSpreads(), useCurrentLanguage()     │  │
  │  │ • ObjectsCreativeSpace     → useSpreads()                           │  │
  │  │ • AnimationsCreativeSpace ⚡ → useSpreads(), useCurrentLanguage()    │  │
  │  │ • OtherCreativeSpace       →  get data from table ()                │  │
  │  │                                                                     │  │
  │  │ Actions via: useSnapshotActions(), useEditorSettingsActions()       │  │
  │  └─────────────────────────────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step ↔ CreativeSpace Mapping

CreativeSpaces được render dựa trên `currentStep`. Mỗi step có bộ icons riêng, icons switch hoàn toàn khi đổi step:

| Step | Step-specific Spaces | Default Spaces (all steps) |
|------|---------------------|---------------------------|
| `manuscript` | doc, dummy⚡, sketch | history, issue, share, collaborator, setting |
| `illustration` | character, prop, stage, spread⚡ | history, issue, share, collaborator, setting |
| `retouch` | object, animation⚡, remix | history, issue, share, collaborator, setting |

⚡ = Language-aware creativeSpaces

**Logic:** IconRail renders step-specific icons based on `currentStep`, plus default icons always visible at bottom.

### 1.4 Language Impact Summary

| CreativeSpace | Receives `currentLanguage` | How it's used |
|-----------|---------------------------|---------------|
| DocCreativeSpace | ❌ | Document content not multilingual |
| DummyCreativeSpace | ❌ | Dummy spreads not multilingual |
| SketchCreativeSpace | ❌ | Sketch sheets not multilingual |
| CharactersCreativeSpace | ❌ | Character metadata not multilingual |
| PropsCreativeSpace | ❌ | Props metadata not multilingual |
| StagesCreativeSpace | ❌ | Stage metadata not multilingual |
| **SpreadsCreativeSpace** | ✅ | Filter `textbox.[language_code]` by `currentLanguage.code` |
| ObjectsCreativeSpace | ❌ | Only displays image objects |
| **AnimationsCreativeSpace** | ✅ | Show textbox names/preview in selected language |
| **RemixCreativeSpace** | ✅ | Asset swapping display with current language |
| **HistoryCreativeSpace** | ✅ | Version history display with current language |
| IssuesCreativeSpace | ❌ | Issues are language-agnostic |
| SharesCreativeSpace | ❌ | Share links are language-agnostic |
| CollaboratorsCreativeSpace | ❌ | Permissions reference languages but don't filter |
| ConfigCreativeSpace | ❌ | Manages `book.remix.languages[]` but doesn't filter |
| **RightSidebar** | ✅ | AI knows which language user is editing |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Container gốc điều phối toàn bộ Editor. Quản lý UI state, fetch data từ API, init SnapshotStore, và render trực tiếp các component con bao gồm creativeSpace.

**Shared Types:**

```typescript
interface Language {
  name: string;       // "English", "Tiếng Việt", "日本語", "한국어", "中文"
  code: string;       // "en_US", "vi_VN", "ja_JP", "ko_KR", "zh_CN"
}

type PipelineStep = 'manuscript' | 'illustration' | 'retouch';

// Step-specific creative spaces
type ManuscriptSpace = 'doc' | 'dummy' | 'sketch';
type IllustrationSpace = 'character' | 'prop' | 'stage' | 'spread';
type RetouchSpace = 'object' | 'animation' | 'remix';

// Default creative spaces (always available)
type DefaultSpace = 'history' | 'issue' | 'share' | 'collaborator' | 'setting';

type CreativeSpaceType = ManuscriptSpace | IllustrationSpace | RetouchSpace | DefaultSpace;

type SaveStatus = 'unsaved' | 'saving' | 'saved';

// constants/languages.ts
const AVAILABLE_LANGUAGES: Language[] = [
  { name: "English (US)", code: "en_US" },
  { name: "Tiếng Việt", code: "vi_VN" },
  { name: "日本語", code: "ja_JP" },
  { name: "한국어", code: "ko_KR" },
  { name: "中文 (简体)", code: "zh_CN" },
  ...
];
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface EditorPageProps {
  bookId: string;
}

interface EditorPageState {
  // Data NOT in SnapshotStore
  book: Book | null;
  issues: Issue[];
  shareLinks: ShareLink[];
  collaborations: Collaboration[];

  // UI State (local only)
  activeCreativeSpace: CreativeSpaceType;
  isLoading: boolean;
  isSidebarOpen: boolean;

  // Dialog State
  translationDialogState: TranslationDialogState | null;

  // NOTE: currentStep & currentLanguage moved to EditorSettingsStore
  // See: component/stores/editor-settings-store.md
}

interface TranslationDialogState {
  isOpen: boolean;
  targetLanguage: Language;
  sourceLanguage: Language;
}

interface EditorPageCallbacks {
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
  onSave: () => Promise<void>;
  onBookUpdate: (updates: Partial<Book>) => void;
  onToggleSidebar: () => void;
  onTranslateContent: (sourceLanguage: Language, targetLanguage: Language) => Promise<void>;

  // ⚡ EditorHeader callbacks (different validation patterns)
  onStepChange: (targetStep: PipelineStep) => void;
  // PRE-VALIDATION: validate → success → setCurrentStep()
  onLanguageChange: (newLang: Language, prevLang: Language) => void;
  // POST-VALIDATION: setCurrentLanguage() already done → check → show dialog if needed
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors
isDirty = useIsDirty();
isSaving = useIsSaving();
spreads = useSpreads();  // for translation check
spreadsCount = useSpreadCount();  // for CollaboratorsCreativeSpace

// SnapshotStore Actions (accessed directly from store)
initSnapshot = useSnapshotStore.getState().initSnapshot;
resetSnapshot = useSnapshotStore.getState().resetSnapshot;

// EditorSettingsStore (global UI state - no prop drilling)
currentLanguage = useCurrentLanguage();
currentStep = useCurrentStep();
{ setCurrentLanguage, setCurrentStep, resetSettings } = useEditorSettingsActions();
```

### 2.3 Render Logic (pseudo)

```
EditorPage:
  ON_MOUNT:
    fetchSnapshot(bookId) → initSnapshot(snapshot)
    resetSettings(book.original_language, 'manuscript')  // Init EditorSettingsStore

  // Derived save status from store
  saveStatus = isSaving ? 'saving' : (isDirty ? 'unsaved' : 'saved')

  // ============================================================
  // HANDLER: onStepChange (PRE-VALIDATION)
  // Validate BEFORE changing step
  // ============================================================
  handleStepChange(targetStep):
    validationResult = canTransitionToStep(currentStep, targetStep, book, snapshot)
    IF validationResult.valid:
      setCurrentStep(targetStep)
      // Also switch to default creativeSpace for new step
      setActiveCreativeSpace(getDefaultCreativeSpace(targetStep))
    ELSE:
      showToast(validationResult.message)  // e.g. "Complete manuscript before moving to illustration"

  // ============================================================
  // HANDLER: onLanguageChange (POST-VALIDATION)
  // Language already changed in store, now check translation
  // ============================================================
  handleLanguageChange(newLang, prevLang):
    // newLang already set in EditorSettingsStore by LanguageSelector
    // Now check if translation exists
    IF newLang.code === book.original_language:
      RETURN  // Skip check for original language

    hasTranslation = spreads.some(spread =>
      spread.textboxes.some(tb => tb[newLang.code] != null)
    )

    IF NOT hasTranslation AND spreads.length > 0:
      // Show dialog to prompt translation
      setTranslationDialogState({
        isOpen: true,
        targetLanguage: newLang,
        sourceLanguage: prevLang
      })

  // NOTE: currentStep, currentLanguage now from EditorSettingsStore - no props needed
  RENDER EditorHeader với bookTitle, saveStatus, onStepChange=handleStepChange, onLanguageChange=handleLanguageChange, ...callbacks
  RENDER IconRail với activeCreativeSpace

  SWITCH activeCreativeSpace:
    // MANUSCRIPT STEP
    'doc'         → RENDER DocCreativeSpace
    'dummy'       → RENDER DummyCreativeSpace
    'sketch'      → RENDER SketchCreativeSpace

    // ILLUSTRATION STEP
    'character'   → RENDER CharactersCreativeSpace
    'prop'        → RENDER PropsCreativeSpace
    'stage'       → RENDER StagesCreativeSpace
    'spread'      → RENDER SpreadsCreativeSpace

    // RETOUCH STEP
    'object'      → RENDER ObjectsCreativeSpace
    'animation'   → RENDER AnimationsCreativeSpace
    'remix'       → RENDER RemixCreativeSpace

    // DEFAULT (all steps)
    'history'     → RENDER HistoryCreativeSpace
    'issue'       → RENDER IssuesCreativeSpace với issues
    'share'       → RENDER SharesCreativeSpace với shareLinks
    'collaborator'→ RENDER CollaboratorsCreativeSpace với collaborations, spreadsCount
    'setting'     → RENDER ConfigCreativeSpace với book

  IF isSidebarOpen:
    RENDER RightSidebar với bookId, activeCreativeSpace, onClose  // uses stores internally
  ELSE:
    RENDER AISidebarToggle với onToggle (floating button bottom-right)

  IF translationDialogState?.isOpen:
    RENDER TranslationNotAvailableDialog với targetLanguage, sourceLanguage, onCancel, onTranslate
```

### 2.4 Visual

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  [≡] [The Hidden Valley  ]   [Manuscript > Illustration > Retouch] [💾][🌐][🔔]│
├───────────┬────────────────────────────────────────────────────┬───────────────┤
│           │                                                    │               │
│  TOP      │                                                    │   AI Chat     │
│ (step     │            CreativeSpace Content Area              │   Sidebar     │
│ icons)    │                                                    │               │
│           │           (conditional based on selection)         │  [Close X]    │
│           │                                                    │               │
│ spacer    │                                                    │               │
│           │                                                    │               │
│ BOTTOM    │                                                    │               │
│ 🕐 histor │                                                    │               │
│ ⚠️ issue   │                                                    │               │
│ 🔗 share  │                                                    │               │
│ 👥 collab │                                                    │               │
│ ────────  │                                                    │               │
│ ⚙️ setting │                                                    │               │
└───────────┴────────────────────────────────────────────────────┴───────────────┘
                                                               ┌─────┐
                                                               │ 💬  │ ← Toggle
                                                               └─────┘   (if sidebar closed)
```

**Visual States:**

```
Loading:                          Error:
┌─────────────────────────────┐   ┌─────────────────────────────┐
│         ⏳ Loading...       │   │    ⚠️ Failed to load book    │
│                             │   │       [Retry] [Home]        │
└─────────────────────────────┘   └─────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - **KHÔNG** thiết kế store integration tại đây — child component tự thiết kế trong file riêng

### 3.1 EditorHeader

📄 **Doc:** [component/editor-page/01-editor-header.md](component/editor-page/01-editor-header.md)

**Mục đích:** Top navigation bar. Hiển thị book info, step navigation, language selector, và quick actions.

**Props & Callbacks:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  // currentStep, currentLanguage via useEditorSettingsStore (READ only)
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;
  onTitleEdit: (newTitle: string) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
  onStepChange: (targetStep: PipelineStep) => void;
  // ⚡ PRE-VALIDATION: EditorPage validates BEFORE setCurrentStep()
  onLanguageChange: (newLang: Language, prevLang: Language) => void;
  // ⚡ POST-VALIDATION: setCurrentLanguage() already done, EditorPage checks translation
}
```

---

### 3.2 IconRail

📄 **Doc:** [component/editor-page/02-icon-rail.md](component/editor-page/02-icon-rail.md)

**Mục đích:** Sidebar navigation dọc bên trái chứa icons để chuyển giữa các CreativeSpace.

**Props & Callbacks:**

```typescript
interface IconRailProps {
  activeCreativeSpace: CreativeSpaceType;
  // currentStep via useCurrentStep()
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
}
```

---

### 3.3 DocCreativeSpace

📄 **Doc:** [component/editor-page/doc-creative-space/00-doc-creative-space.md](doc-creative-space/00-doc-creative-space.md)

📸 **Screenshot:** `screenshots/manuscript-docs-space.png`

**Mục đích:** Soạn thảo manuscript documents: Brief, Draft, Script. Left panel với document tabs, right panel với rich text editor.

**Props & Callbacks:**

```typescript
interface DocCreativeSpaceProps {
  // No props needed - pure store consumer (useDocs())
}
```

---

### 3.4 DummyCreativeSpace

📄 **Doc:** [component/editor-page/dummy-creative-space/00-dummy-creative-space.md](dummy-creative-space/00-dummy-creative-space.md)

📸 **Screenshot:** `screenshots/manuscript-dummy-space.png`

**Mục đích:** Dummy layout editor. Prose/Poetry dummy types với spread grid. Quản lý textboxes và art notes.

**Props & Callbacks:**

```typescript
interface DummyCreativeSpaceProps {
  // No props needed - pure store consumer
}
```

---

### 3.5 SketchCreativeSpace

📄 **Doc:** [component/editor-page/sketch-creative-space/00-sketch-creative-space.md](sketch-creative-space/00-sketch-creative-space.md)

📸 **Screenshot:** `screenshots/manuscript-sketch-space.png`

**Mục đích:** Sketch sheets viewer. Hiển thị Characters, Props, Spreads sheets được generate từ dummy.

**Props & Callbacks:**

```typescript
interface SketchCreativeSpaceProps {
  // No props needed - pure store consumer (useSketch())
}
```

---

### 3.6 CharactersCreativeSpace

📄 **Doc:** [component/editor-page/06-characters-creative-space.md](component/editor-page/06-characters-creative-space.md)

📸 **Screenshot:** `screenshots/Illustration-character-space.png`

**Mục đích:** Quản lý nhân vật: thông tin cơ bản, variants, voices, crops.

**Props & Callbacks:**

```typescript
interface CharactersCreativeSpaceProps {
  // No props needed - pure store consumer (useCharacters())
}
```

---

### 3.7 PropsCreativeSpace

📄 **Doc:** [component/editor-page/07-props-creative-space.md](component/editor-page/07-props-creative-space.md)

📸 **Screenshot:** `screenshots/Illustration-prop-space.png`

**Mục đích:** Quản lý đạo cụ: states, sounds, crops.

**Props & Callbacks:**

```typescript
interface PropsCreativeSpaceProps {
  // No props needed - pure store consumer (useProps())
}
```

---

### 3.8 StagesCreativeSpace

📄 **Doc:** [component/editor-page/08-stages-creative-space.md](component/editor-page/08-stages-creative-space.md)

📸 **Screenshot:** `screenshots/Illustration-stage-space.png`

**Mục đích:** Quản lý bối cảnh: settings (temporal, sensory, emotional), sounds.

**Props & Callbacks:**

```typescript
interface StagesCreativeSpaceProps {
  // No props needed - pure store consumer (useStages())
}
```

---

### 3.9 SpreadsCreativeSpace ⚡

📄 **Doc:** [component/editor-page/09-spreads-creative-space.md](component/editor-page/09-spreads-creative-space.md)

📸 **Screenshot:** `screenshots/Illustration-spread-space.png`

**Mục đích:** Layout visual editor cho các trang đôi (spread). Quản lý Elements tree (background, images, textboxes).

**Special Impact:** ✅ Textbox content hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface SpreadsCreativeSpaceProps {
  // currentLanguage via useCurrentLanguage() - no prop drilling
}
```

**Data Structure (textbox language):**

```json
{
  "textboxes": [
    {
      "id": "tb_001",
      "title": "Opening narration",
      "en_US": { "text": "Once upon a time...", "geometry": {...}, "typography": {...} },
      "vi_VN": { "text": "Ngày xửa ngày xưa...", "geometry": {...}, "typography": {...} }
    }
  ]
}
```

---

### 3.10 ObjectsCreativeSpace

📄 **Doc:** [component/editor-page/10-objects-creative-space.md](component/editor-page/10-objects-creative-space.md)

📸 **Screenshot:** `screenshots/Retouch-object-space.png`

**Mục đích:** Retouch layer management. Điều chỉnh vị trí, kích thước, z-index các object trên spread.

**Props & Callbacks:**

```typescript
interface ObjectsCreativeSpaceProps {
  // No props needed - pure store consumer (useSpreads())
}
```

---

### 3.11 AnimationsCreativeSpace ⚡

📄 **Doc:** [component/editor-page/11-animations-creative-space.md](component/editor-page/11-animations-creative-space.md)

📸 **Screenshot:** `screenshots/Retouch-animation-space.png`

**Mục đích:** Timeline editor cho animations. Quản lý trigger, delay, duration, effect types.

**Special Impact:** ✅ Animation list hiển thị textbox name/content theo `currentLanguage`

**Props & Callbacks:**

```typescript
interface AnimationsCreativeSpaceProps {
  // currentLanguage via useCurrentLanguage() - no prop drilling
}
```

---

### 3.12 RemixCreativeSpace ⚡

📄 **Doc:** [component/editor-page/12-remix-creative-space.md](component/editor-page/12-remix-creative-space.md)

📸 **Screenshot:** `screenshots/Retouch-remix-space.png`

**Mục đích:** Asset remixing interface. Cho phép swap characters và props trong book từ thư viện assets hoặc other books.

**Special Impact:** ✅ Textbox content hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface RemixCreativeSpaceProps {
  // No props needed - consumes book.remix and character/prop data from store
}
```

---

### 3.13 HistoryCreativeSpace ⚡

📄 **Doc:** [component/editor-page/13-history-creative-space.md](component/editor-page/13-history-creative-space.md)

📸 **Screenshot:** `screenshots/history-space.png`

**Mục đích:** Version history viewer. Hiển thị danh sách snapshots đã lưu với timestamp, tag, và cho phép restore.

**Special Impact:** ✅ Textbox content hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface HistoryCreativeSpaceProps {
  // No props needed - fetches snapshots for current book
}
```

---

### 3.14 IssuesCreativeSpace

📄 **Doc:** [component/editor-page/14-issues-creative-space.md](component/editor-page/14-issues-creative-space.md)

**Mục đích:** Hiển thị và xử lý các vấn đề (quality warnings, consistency issues).

**Props & Callbacks:**

```typescript
interface IssuesCreativeSpaceProps {
  issues: Issue[];
  onIssuesUpdate: (issues: Issue[]) => void;
  onNavigateToIssue: (issue: Issue) => void;
}
```

---

### 3.15 SharesCreativeSpace

📄 **Doc:** [component/editor-page/15-shares-creative-space.md](component/editor-page/15-shares-creative-space.md)

**Mục đích:** Quản lý share links (public preview, client review, team draft).

**Props & Callbacks:**

```typescript
interface SharesCreativeSpaceProps {
  shareLinks: ShareLink[];
  onShareLinksUpdate: (links: ShareLink[]) => void;
}
```

---

### 3.16 CollaboratorsCreativeSpace

📄 **Doc:** [component/editor-page/16-collaborators-creative-space.md](component/editor-page/16-collaborators-creative-space.md)

**Mục đích:** Quản lý collaborators và permissions (languages, steps, spreads access).

**Props & Callbacks:**

```typescript
interface CollaboratorsCreativeSpaceProps {
  collaborations: Collaboration[];
  spreadsCount: number;
  onCollaborationsUpdate: (collabs: Collaboration[]) => void;
}
```

---

### 3.17 ConfigCreativeSpace

📄 **Doc:** [component/editor-page/17-config-creative-space.md](component/editor-page/17-config-creative-space.md)

**Mục đích:** Cấu hình book: general, creative, typography, layout, remix, export.

**Props & Callbacks:**

```typescript
interface ConfigCreativeSpaceProps {
  book: Book;
  onBookUpdate: (updates: Partial<Book>) => void;
}
```

---

### 3.18 RightSidebar (AI Assistant) ⚡

📄 **Doc:** [component/editor-page/18-right-sidebar.md](component/editor-page/18-right-sidebar.md)

**Mục đích:** Panel AI Assistant hỗ trợ người dùng. Contextual với creativeSpace hiện tại.

**Special Impact:** ✅ AI knows which language user is editing.

**Props & Callbacks:**

```typescript
interface RightSidebarProps {
  isOpen: boolean;
  bookId: string;
  // currentStep via useCurrentStep()
  activeCreativeSpace: CreativeSpaceType;
  // currentLanguage via useCurrentLanguage() - no prop drilling
  contextData?: {
    selectedCharacter?: Character;
    selectedProp?: Prop;
    selectedStage?: Stage;
    selectedSpread?: Spread;
  };
  onClose: () => void;
}
```

---

### 3.19 AISidebarToggle

📄 **Doc:** *(inline — không cần file riêng)*

**Mục đích:** Floating button ở góc dưới bên phải để mở AI Assistant sidebar.

**Props & Callbacks:**

```typescript
interface AISidebarToggleProps {
  onToggle: () => void;
}
```

**Visual:**

```
┌─────────────────────────────────────────┐
│                                         │
│                                 ┌─────┐ │
│                                 │ 💬  │ │  ← Floating button
│                                 └─────┘ │     position: fixed
│                                         │     bottom-right
└─────────────────────────────────────────┘
```

---

### 3.20 TranslationNotAvailableDialog

📄 **Doc:** [component/editor-page/20-translation-not-available-dialog.md](component/editor-page/20-translation-not-available-dialog.md)

**Mục đích:** Dialog xác nhận khi user chọn language mà chưa có translation trong textboxes.

**Trigger:** Khi `onLanguageChange` được gọi và `spreads[].textboxes[]` chưa có key `[language_code]`.

**Props & Callbacks:**

```typescript
interface TranslationNotAvailableDialogProps {
  isOpen: boolean;
  targetLanguage: Language;
  sourceLanguage: Language;
  onCancel: () => void;
  onTranslate: () => void;
}
```

**Visual:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                         [X] │
│  Translation Not Available                                  │
│                                                             │
│  The translation for **Tiếng Việt** is not available yet.   │
│  Would you like to translate your content to this           │
│  language?                                                  │
│                                                             │
│                    ┌──────────┐  ┌────────────────────────┐ │
│                    │  Cancel  │  │  ✨ Translate          │ │
│                    └──────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**No Intermediate MainCreativeSpace Component**
EditorPage render trực tiếp creativeSpace dựa trên `activeCreativeSpace`. Lý do: MainCreativeSpace không có responsibility riêng ngoài routing, giảm props drilling, code đơn giản hơn.

**State Split: Stores vs Local State**
| Data | Location | Reason |
|------|----------|--------|
| docs, dummies, sketch, spreads, characters, props, stages | SnapshotStore | Shared across CreativeSpaces, persist to DB |
| currentStep, currentLanguage | EditorSettingsStore | Global UI state, avoid prop drilling |
| book, issues, shareLinks, collaborations | Local state | Not part of snapshot, different update patterns |
| activeCreativeSpace, isSidebarOpen | Local state | UI-only, single component usage |

**Language as UI State**
`currentLanguage` là UI state (view preference), không phải data state. Nó quyết định ngôn ngữ nào được hiển thị trong editor, nhưng không thay đổi data structure của textbox.

**AI Sidebar Toggle as Floating Button**
`AISidebarToggle` là floating button ở góc dưới bên phải, hiển thị khi sidebar đóng. Khi sidebar mở, button ẩn đi và thay bằng nút X trong sidebar header.

**Static Language List**
Danh sách available languages lấy từ constant tĩnh, không phải từ `book.remix.languages[]`.

**CreativeSpace Isolation**
Mỗi creativeSpace có local state riêng (selected item, active tab, filter). State này không cần sync lên EditorPage.

**Conditional Rendering**
Render duy nhất một creativeSpace tại một thời điểm. Unmount creativeSpace cũ khi chuyển. Data persists in SnapshotStore.

### 4.2 SnapshotStore Integration

| Component | Selectors | Actions |
|-----------|-----------|---------|
| EditorPage | `useIsDirty()`, `useIsSaving()`, `useSpreads()`, `useSpreadCount()` | `initSnapshot()`*, `resetSnapshot()`* |
| DocCreativeSpace | `useDocs()` | `updateDoc()`, etc. |
| DummyCreativeSpace | `useDummies()`, `useDummySpreads(type)` | `addDummySpread()`, `updateDummySpread()`, etc. |
| SketchCreativeSpace | `useSketch()` | (read-only from generate) |
| CharactersCreativeSpace | `useCharacters()`, `useCharacterByKey()` | `addCharacter()`, `updateCharacter()`, etc. |
| PropsCreativeSpace | `useProps()`, `usePropByKey()` | `addProp()`, `updateProp()`, etc. |
| StagesCreativeSpace | `useStages()`, `useStageByKey()` | `addStage()`, `updateStage()`, etc. |
| SpreadsCreativeSpace | `useSpreads()`, `useSpreadById()`, `useCharacters()`, `useProps()`, `useStages()` | Spread CRUD, textbox/image actions |
| ObjectsCreativeSpace | `useSpreads()`, `useSpreadById()` | `updateSpreadObject()`, etc. |
| AnimationsCreativeSpace | `useSpreads()`, `useSpreadById()` | `addSpreadAnimation()`, `updateSpreadAnimation()`, etc. |
| RemixCreativeSpace | `useCharacters()`, `useProps()` | `swapCharacter()`, `swapProp()`, etc. |
| HistoryCreativeSpace | (external API for snapshots) | `restoreSnapshot()` |

\* = Top-level store actions (accessed via `useSnapshotStore.getState()`, not `useSnapshotActions()`)

### 4.3 Step Transition Validation (PRE-VALIDATION)

> **Pattern:** Validate BEFORE changing step. EditorHeader calls `onStepChange()` callback, EditorPage validates, only `setCurrentStep()` if success.

```typescript
interface ValidationResult {
  valid: boolean;
  missingFields?: string[];
  message?: string;
}

function canTransitionToStep(
  from: PipelineStep,
  to: PipelineStep,
  book: Book,
  snapshot: Snapshot
): ValidationResult;
```

**Flow:**

```
EditorHeader: user clicks step
       │
       ▼
onStepChange(targetStep)  ← callback to EditorPage
       │
       ▼
EditorPage: canTransitionToStep(current, target, book, snapshot)
       │
   ┌───┴───┐
valid:true  valid:false
   │            │
   ▼            ▼
setCurrentStep()  showToast(message)
setActiveCreativeSpace()  // NO step change
```

**Lưu ý:** EditorHeader KHÔNG gọi trực tiếp `setCurrentStep()`. Chỉ EditorPage mới có quyền thay đổi step sau khi validate.

### 4.4 Translation Check Flow (POST-VALIDATION)

> **Pattern:** Change language FIRST, then check translation. LanguageSelector calls `setCurrentLanguage()` trước, sau đó callback `onLanguageChange()` để EditorPage check.

Khi user chọn language mới trong LanguageSelector:

```
LanguageSelector: user selects language
       │
       ├─→ setCurrentLanguage(newLang)  ← UPDATE store FIRST
       │
       └─→ onLanguageChange(newLang, prevLang)  ← callback to EditorPage
              │
              ▼
       EditorPage: handleLanguageChange(newLang, prevLang)
              │
              ├─→ IF newLang === original_language → RETURN (no check)
              │
              └─→ Check hasTranslation for newLang.code
                     │
                 ┌───┴───┐
             has:true   has:false
                 │           │
                 ▼           ▼
             (nothing)   Show TranslationNotAvailableDialog
                         (user đã thấy empty state)
```

**Lưu ý:** User sẽ thấy empty state TRƯỚC khi dialog xuất hiện. Đây là by design để user thấy ngay context.

**Edge cases:**

| Case | Behavior |
|------|----------|
| Select original language | No check, no dialog |
| All spreads empty (no textboxes) | No dialog (nothing to translate) |
| Some textboxes have translation | No dialog (partial OK) |
| User cancels dialog | Close dialog, language already changed, user sees empty/partial content |

### 4.5 Initial Language

Khi load Editor, `currentLanguage` mặc định là `book.original_language` hoặc language đầu tiên trong `AVAILABLE_LANGUAGES`.

### 4.6 When to Extract MainCreativeSpace thành component

Cân nhắc tách MainCreativeSpace wrapper component nếu xuất hiện nhu cầu:
- Transition animation giữa các creativeSpace
- Shared toolbar/header riêng cho creativeSpace area
- Error boundary riêng (crash creativeSpace không crash toàn app)
- Lazy loading creativeSpaces (code splitting với Suspense)
- CreativeSpace state persistence (giữ state khi switch, không unmount)
