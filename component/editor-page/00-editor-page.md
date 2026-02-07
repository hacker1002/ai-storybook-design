# EditorPage: Component Design

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
│  │        │  Conditional Render:                              │   Right      │  │
│  │        │  ┌─────────────────────────────────────────────┐  │   Sidebar    │  │
│  │  Icon  │  │ ManuscriptCreativeSpace   (if manuscript) ⚡ │  │     (AI)     │  │
│  │  Rail  │  │ CharactersCreativeSpace   (if characters)   │  │              │  │
│  │        │  │ PropsCreativeSpace        (if props)        │  │  ┌────────┐  │  │
│  │        │  │ StagesCreativeSpace       (if stages)       │  │  │   X    │  │  │
│  │        │  │ SpreadsCreativeSpace      (if spreads) ⚡    │  │  │ close  │  │  │
│  │        │  │ ObjectsCreativeSpace      (if objects)      │  │  └────────┘  │  │
│  │        │  │ AnimationsCreativeSpace   (if animations) ⚡ │  │              │  │
│  │        │  │ FlagsCreativeSpace        (if flags)        │  │              │  │
│  │        │  │ SharesCreativeSpace       (if shares)       │  │              │  │
│  │        │  │ CollaboratorsCreativeSpace(if collabs)      │  │              │  │
│  │        │  │ ConfigCreativeSpace       (if config)       │  │              │  │
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
  │  │ • ManuscriptCreativeSpace ⚡ → useManuscript(), useCurrentLanguage() │  │
  │  │ • CharactersCreativeSpace  → useCharacters(), useCurrentStep()      │  │
  │  │ • PropsCreativeSpace       → useProps(), useCurrentStep()           │  │
  │  │ • StagesCreativeSpace      → useStages(), useCurrentStep()          │  │
  │  │ • SpreadsCreativeSpace ⚡   → useSpreads(), useCurrentLanguage()     │  │
  │  │ • ObjectsCreativeSpace     → useSpreads()                           │  │
  │  │ • AnimationsCreativeSpace ⚡ → useSpreads(), useCurrentLanguage()    │  │
  │  │                                                                     │  │
  │  │ Actions via: useSnapshotActions(), useEditorSettingsActions()       │  │
  │  └─────────────────────────────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step ↔ CreativeSpace Mapping

CreativeSpaces được enable dựa trên nguyên tắc "từ step X trở đi" (progressive unlock):

| Step | Newly Enabled | All Available CreativeSpaces |
|------|---------------|--------------------------|
| `idea` | manuscript, flags, shares, collabs, config | manuscript, flags, shares, collabs, config |
| `sketch` | characters, props, stages, spreads⚡ | manuscript, characters, props, stages, spreads⚡, flags, shares, collabs, config |
| `illustration` | (none) | manuscript, characters, props, stages, spreads⚡, flags, shares, collabs, config |
| `retouch` | objects, animations⚡ | manuscript, characters, props, stages, spreads⚡, objects, animations⚡, flags, shares, collabs, config |

⚡ = Language-aware creativeSpaces

**Logic:** `currentStep >= enabledFromStep`

### 1.4 Language Impact Summary

| CreativeSpace | Receives `currentLanguage` | How it's used |
|-----------|---------------------------|---------------|
| ManuscriptCreativeSpace | ✅ | Finalization step needs current language |
| CharactersCreativeSpace | ❌ | Character metadata not multilingual |
| PropsCreativeSpace | ❌ | Props metadata not multilingual |
| StagesCreativeSpace | ❌ | Stage metadata not multilingual |
| **SpreadsCreativeSpace** | ✅ | Filter `textbox.[language_code]` by `currentLanguage.code` |
| ObjectsCreativeSpace | ❌ | Only displays image objects |
| **AnimationsCreativeSpace** | ✅ | Show textbox names/preview in selected language |
| FlagsCreativeSpace | ❌ | Flags are language-agnostic |
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

type Step = 'idea' | 'sketch' | 'illustration' | 'retouch';

type CreativeSpaceType =
  | 'manuscript' | 'characters' | 'props' | 'stages' | 'spreads'
  | 'objects' | 'animations' | 'flags' | 'shares' | 'collabs' | 'config';

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
  flags: Flag[];
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
  // NOTE: onStepChange & onLanguageChange now via EditorSettingsStore actions
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
  onSave: () => Promise<void>;
  onBookUpdate: (updates: Partial<Book>) => void;
  onToggleSidebar: () => void;
  onTranslateContent: (sourceLanguage: Language, targetLanguage: Language) => Promise<void>;
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
    resetSettings(book.original_language, 'idea')  // Init EditorSettingsStore

  // Derived save status from store
  saveStatus = isSaving ? 'saving' : (isDirty ? 'unsaved' : 'saved')

  // NOTE: currentStep, currentLanguage now from EditorSettingsStore - no props needed
  RENDER EditorHeader với bookTitle, saveStatus, callbacks
  RENDER IconRail với activeCreativeSpace

  SWITCH activeCreativeSpace:
    'manuscript' → RENDER ManuscriptCreativeSpace  // uses useCurrentLanguage() internally
    'characters'  → RENDER CharactersCreativeSpace  // uses useCurrentStep() internally
    'props'       → RENDER PropsCreativeSpace
    'stages'      → RENDER StagesCreativeSpace
    'spreads'     → RENDER SpreadsCreativeSpace  // uses both stores internally
    'objects'     → RENDER ObjectsCreativeSpace
    'animations'  → RENDER AnimationsCreativeSpace  // uses useCurrentLanguage() internally
    'flags'       → RENDER FlagsCreativeSpace với flags
    'shares'      → RENDER SharesCreativeSpace với shareLinks
    'collabs'     → RENDER CollaboratorsCreativeSpace với collaborations, spreadsCount
    'config'      → RENDER ConfigCreativeSpace với book

  IF isSidebarOpen:
    RENDER RightSidebar với bookId, activeCreativeSpace, onClose  // uses stores internally
  ELSE:
    RENDER AISidebarToggle với onToggle (floating button bottom-right)

  IF translationDialogState?.isOpen:
    RENDER TranslationNotAvailableDialog với targetLanguage, sourceLanguage, onCancel, onTranslate
```

### 2.4 Visual

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [MenuBtn] [BookTitle  ]     [Step1→Step2→Step3→Step4]       [💾] [🌐] [🔔] │
├────────┬────────────────────────────────────────────────────┬───────────────┤
│  📝    │                                                    │               │
│  👥    │                                                    │   AI Chat     │
│  🎨    │            CreativeSpace Content Area              │   Sidebar     │
│  🏛️    │                                                    │               │
│  📄    │           (conditional based on selection)         │  [Close X]    │
│  🎭    │                                                    │               │
│  ✨    │                                                    │               │
│  ⚠️     │                                                    │               │
│  🔗    │                                                    │               │
│  👤    │                                                    │               │
│  ⚙️     │                                                    │               │
└────────┴────────────────────────────────────────────────────┴───────────────┘
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
  // currentStep, currentLanguage via useEditorSettingsStore
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;
  // onLanguageChange, onStepChange via useEditorSettingsActions
  onTitleEdit: (newTitle: string) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
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

### 3.3 ManuscriptCreativeSpace ⚡

📄 **Doc:** [component/editor-page/03-manuscript-creative-space.md](component/editor-page/03-manuscript-creative-space.md)

**Mục đích:** Soạn thảo manuscript theo các bước: Brief → Draft → Script → Prose Dummy → Poetry Dummy → Finalization.

**Special Impact:** ✅ Finalization step needs `currentLanguage` for translation.

**Props & Callbacks:**

```typescript
interface ManuscriptCreativeSpaceProps {
  // currentLanguage via useCurrentLanguage() - no prop drilling
}
```

---

### 3.4 CharactersCreativeSpace

📄 **Doc:** [component/editor-page/04-characters-creative-space.md](component/editor-page/04-characters-creative-space.md)

**Mục đích:** Quản lý nhân vật: thông tin cơ bản, variants, voices, crops.

**Props & Callbacks:**

```typescript
interface CharactersCreativeSpaceProps {
  // currentStep via useCurrentStep() - no prop drilling
}
```

---

### 3.5 PropsCreativeSpace

📄 **Doc:** [component/editor-page/05-props-creative-space.md](component/editor-page/05-props-creative-space.md)

**Mục đích:** Quản lý đạo cụ: states, sounds, crops.

**Props & Callbacks:**

```typescript
interface PropsCreativeSpaceProps {
  // currentStep via useCurrentStep() - no prop drilling
}
```

---

### 3.6 StagesCreativeSpace

📄 **Doc:** [component/editor-page/06-stages-creative-space.md](component/editor-page/06-stages-creative-space.md)

**Mục đích:** Quản lý bối cảnh: settings (temporal, sensory, emotional), sounds.

**Props & Callbacks:**

```typescript
interface StagesCreativeSpaceProps {
  // currentStep via useCurrentStep() - no prop drilling
}
```

---

### 3.7 SpreadsCreativeSpace ⚡

📄 **Doc:** [component/editor-page/07-spreads-creative-space.md](component/editor-page/07-spreads-creative-space.md)

**Mục đích:** Layout visual editor cho các trang đôi (spread). Quản lý images, textboxes.

**Special Impact:** ✅ Textbox content hiển thị theo `currentLanguage.code`

**Props & Callbacks:**

```typescript
interface SpreadsCreativeSpaceProps {
  // currentStep via useCurrentStep()
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

### 3.8 ObjectsCreativeSpace

📄 **Doc:** [component/editor-page/08-objects-creative-space.md](component/editor-page/08-objects-creative-space.md)

**Mục đích:** Retouch layer management. Điều chỉnh vị trí, kích thước, z-index các object (image) trên spread.

**Props & Callbacks:**

```typescript
interface ObjectsCreativeSpaceProps {
  // No props needed - pure store consumer
}
```

---

### 3.9 AnimationsCreativeSpace ⚡

📄 **Doc:** [component/editor-page/09-animations-creative-space.md](component/editor-page/09-animations-creative-space.md)

**Mục đích:** Timeline editor cho animations. Quản lý trigger, delay, duration, effect types.

**Special Impact:** ✅ Animation list hiển thị textbox name/content theo `currentLanguage`

**Props & Callbacks:**

```typescript
interface AnimationsCreativeSpaceProps {
  // currentLanguage via useCurrentLanguage() - no prop drilling
}
```

---

### 3.10 FlagsCreativeSpace

📄 **Doc:** [component/editor-page/10-flags-creative-space.md](component/editor-page/10-flags-creative-space.md)

**Mục đích:** Hiển thị và xử lý các vấn đề (quality warnings, consistency issues).

**Props & Callbacks:**

```typescript
interface FlagsCreativeSpaceProps {
  flags: Flag[];
  onFlagsUpdate: (flags: Flag[]) => void;
  onNavigateToIssue: (flag: Flag) => void;
}
```

---

### 3.11 SharesCreativeSpace

📄 **Doc:** [component/editor-page/11-shares-creative-space.md](component/editor-page/11-shares-creative-space.md)

**Mục đích:** Quản lý share links (public preview, client review, team draft).

**Props & Callbacks:**

```typescript
interface SharesCreativeSpaceProps {
  shareLinks: ShareLink[];
  onShareLinksUpdate: (links: ShareLink[]) => void;
}
```

---

### 3.12 CollaboratorsCreativeSpace

📄 **Doc:** [component/editor-page/12-collaborators-creative-space.md](component/editor-page/12-collaborators-creative-space.md)

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

### 3.13 ConfigCreativeSpace

📄 **Doc:** [component/editor-page/13-config-creative-space.md](component/editor-page/13-config-creative-space.md)

**Mục đích:** Cấu hình book: general, creative, typography, layout, remix, export.

**Props & Callbacks:**

```typescript
interface ConfigCreativeSpaceProps {
  book: Book;
  onBookUpdate: (updates: Partial<Book>) => void;
}
```

---

### 3.14 RightSidebar (AI Assistant) ⚡

📄 **Doc:** [component/editor-page/14-right-sidebar.md](component/editor-page/14-right-sidebar.md)

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

### 3.15 AISidebarToggle

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

### 3.16 TranslationNotAvailableDialog

📄 **Doc:** [component/editor-page/16-translation-not-available-dialog.md](component/editor-page/16-translation-not-available-dialog.md)

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
| manuscript, spreads, characters, props, stages | SnapshotStore | Shared across CreativeSpaces, persist to DB |
| currentStep, currentLanguage | EditorSettingsStore | Global UI state, avoid prop drilling |
| book, flags, shareLinks, collaborations | Local state | Not part of snapshot, different update patterns |
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
| ManuscriptCreativeSpace | `useManuscript()`, `useDummySpreads(type)`, `useSpreads()` | `updateDoc()`, `addDummySpread()`, etc. |
| CharactersCreativeSpace | `useCharacters()`, `useCharacterByKey()` | `addCharacter()`, `updateCharacter()`, etc. |
| PropsCreativeSpace | `useProps()`, `usePropByKey()` | `addProp()`, `updateProp()`, etc. |
| StagesCreativeSpace | `useStages()`, `useStageByKey()` | `addStage()`, `updateStage()`, etc. |
| SpreadsCreativeSpace | `useSpreads()`, `useSpreadById()`, `useCharacters()`, `useProps()`, `useStages()` | Spread CRUD, textbox/image actions |
| ObjectsCreativeSpace | `useSpreads()`, `useSpreadById()` | `updateSpreadObject()`, etc. |
| AnimationsCreativeSpace | `useSpreads()`, `useSpreadById()` | `addSpreadAnimation()`, `updateSpreadAnimation()`, etc. |

\* = Top-level store actions (accessed via `useSnapshotStore.getState()`, not `useSnapshotActions()`)

### 4.3 Step Transition Validation

```typescript
interface ValidationResult {
  valid: boolean;
  missingFields?: string[];
  message?: string;
}

function canTransitionToStep(
  from: Step,
  to: Step,
  book: Book,
  snapshot: Snapshot
): ValidationResult;
```

**Flow:**

```
onStepChange(targetStep)
       │
       ▼
canTransitionToStep(current, target, book, snapshot)
       │
   ┌───┴───┐
valid:true  valid:false
   │            │
   ▼            ▼
Update     Show feedback
step       (toast/modal)
```

### 4.4 Translation Check Flow

Khi user chọn language mới trong LanguageSelector:

```
handleLanguageChange(language):
  1. Always update currentLanguage (allow viewing empty state)
  2. Skip check if selecting original language
  3. Check if any textbox has translation for language.code
  4. If no translation found → show TranslationNotAvailableDialog
```

**Edge cases:**

| Case | Behavior |
|------|----------|
| Select original language | No dialog |
| All spreads empty (no textboxes) | No dialog (nothing to translate) |
| Some textboxes have translation | No dialog (partial OK) |
| User cancels dialog | Close dialog, language already changed |

### 4.5 Initial Language

Khi load Editor, `currentLanguage` mặc định là `book.original_language` hoặc language đầu tiên trong `AVAILABLE_LANGUAGES`.

### 4.6 When to Extract MainCreativeSpace thành component

Cân nhắc tách MainCreativeSpace wrapper component nếu xuất hiện nhu cầu:
- Transition animation giữa các creativeSpace
- Shared toolbar/header riêng cho creativeSpace area
- Error boundary riêng (crash creativeSpace không crash toàn app)
- Lazy loading creativeSpaces (code splitting với Suspense)
- CreativeSpace state persistence (giữ state khi switch, không unmount)
