# EditorPage: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           EditorHeader                                 │  │
│  │  ┌────────┬─────────────┬──────────────────┬────────┬────────┬──────┐ │  │
│  │  │MenuBtn │ BookTitle   │ StepBreadcrumb   │SaveStat│LangSel │Notif │ │  │
│  │  └────────┴─────────────┴──────────────────┴────────┴────────┴──────┘ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────┬──────────────────────────────────────────────┬──────────────┐  │
│  │        │                                               │              │  │
│  │        │  Conditional Render (trực tiếp):              │   Right      │  │
│  │        │  ┌─────────────────────────────────────────┐  │   Sidebar    │  │
│  │  Icon  │  │ ManuscriptCreativeSpace   (if manuscript) ⚡│  │     (AI)     │  │
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
│  │        │  └─────────────────────────────────────────┘  │              │  │
│  └────────┴──────────────────────────────────────────────┴──────────────┘  │
│                                                                              │
│                                                    ┌─────────────────────┐  │
│                                                    │ 💬 AISidebarToggle  │  │
│                                                    │  (floating button)  │  │
│                                                    │  bottom-right       │  │
│                                                    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘

⚡ = CreativeSpaces affected by currentLanguage
```

**Lưu ý:** Không có component MainCreativeSpace trung gian. EditorPage render trực tiếp creativeSpace tương ứng với `activeCreativeSpace`.

### 1.2 Data Flow

```
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  State: book, snapshot, flags, shareLinks, collaborations               │ │
│  │         currentStep, activeCreativeSpace, currentLanguage, saveStatus       │ │
│  │         isSidebarOpen                                                   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│         │              │                              │                │     │
│         ▼              ▼                              ▼                ▼     │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────────────────┐ ┌─────────┐ │
│  │  Editor   │  │  Icon     │  │     CreativeSpace (1 of 11)      │ │ Right   │ │
│  │  Header   │  │  Rail     │  │                              │ │ Sidebar │ │
│  │           │  │           │  │  Rendered directly based on  │ │         │ │
│  │ Props:    │  │ Props:    │  │  activeCreativeSpace state:      │ │ Props:  │ │
│  │ •bookTitle│  │ •active   │  │                              │ │ •isOpen │ │
│  │ •step     │  │CreativeSpace│ │ • ManuscriptCreativeSpace ⚡     │ │ •bookId │ │
│  │ •language │  │ •step     │  │  • CharactersCreativeSpace       │ │ •step   │ │
│  │ •saveStat │  │           │  │  • PropsCreativeSpace            │ │ •lang   │ │
│  │           │  │ Callback: │  │  • StagesCreativeSpace           │ │ •context│ │
│  │ Callbacks:│  │ •onChange │  │  • SpreadsCreativeSpace    ⚡     │ │         │ │
│  │ •onSave   │  │           │  │  • ObjectsCreativeSpace          │ │Callback:│ │
│  │ •onStep   │  │           │  │  • AnimationsCreativeSpace ⚡     │ │ •onClose│ │
│  │  Change   │  │           │  │  • FlagsCreativeSpace            │ └─────────┘ │
│  │ •onLang   │  │           │  │  • SharesCreativeSpace           │             │
│  │  Change   │  │           │  │  • CollaboratorsCreativeSpace    │ ┌─────────┐ │
│  │           │  │           │  │  • ConfigCreativeSpace           │ │AISidebar│ │
│  └───────────┘  └───────────┘  │                              │ │ Toggle  │ │
│                                │  ⚡ = receives currentLanguage│ │(floating│ │
│                                └──────────────────────────────┘ │ button) │ │
│                                                                 └─────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
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
| ManuscriptCreativeSpace | ✅ | Manuscript finalization step translate need current language |
| CharactersCreativeSpace | ❌ | Character metadata not multilingual |
| PropsCreativeSpace | ❌ | Props metadata not multilingual |
| StagesCreativeSpace | ❌ | Stage metadata not multilingual |
| **SpreadsCreativeSpace** | ✅ | Filter `textbox.[language_code]` by `currentLanguage.code` |
| ObjectsCreativeSpace | ❌ | Only displays image objects, not textboxes |
| **AnimationsCreativeSpace** | ✅ | Show textbox names/preview in selected language |
| FlagsCreativeSpace | ❌ | Flags are language-agnostic |
| SharesCreativeSpace | ❌ | Share links are language-agnostic |
| CollaboratorsCreativeSpace | ❌ | Permissions reference languages but don't filter by current |
| ConfigCreativeSpace | ❌ | Manages `book.remix.languages[]` but doesn't filter |
| **RightSidebar** | ✅ | AI knows which language user is editing |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Container gốc điều phối toàn bộ Editor. Quản lý state toàn cục, fetch data từ API, và render trực tiếp các component con bao gồm creativeSpace.

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
];
```

### 2.2 Interface

```typescript
interface EditorPageProps {
  bookId: string;
}

interface EditorPageState {
  // Data
  book: Book | null;
  snapshot: Snapshot | null;
  flags: Flag[];
  shareLinks: ShareLink[];
  collaborations: Collaboration[];

  // UI State
  currentStep: Step;
  activeCreativeSpace: CreativeSpaceType;
  currentLanguage: Language;
  saveStatus: SaveStatus;
  isLoading: boolean;
  isSidebarOpen: boolean;
}

interface EditorPageCallbacks {
  onStepChange: (step: Step) => void;
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
  onLanguageChange: (language: Language) => void;
  onSave: () => Promise<void>;
  onBookUpdate: (updates: Partial<Book>) => void;
  onSnapshotUpdate: (updates: Partial<Snapshot>) => void;
  onToggleSidebar: () => void;
}
```

### 2.3 Render Logic (pseudo)

```
EditorPage:
  RENDER EditorHeader với bookTitle, currentStep, currentLanguage, callbacks
  RENDER IconRail với activeCreativeSpace, currentStep

  SWITCH activeCreativeSpace:
    'manuscript' → RENDER ManuscriptCreativeSpace với manuscript, currentLanguage ⚡
    'characters'  → RENDER CharactersCreativeSpace với characters, currentStep
    'props'       → RENDER PropsCreativeSpace với props, currentStep
    'stages'      → RENDER StagesCreativeSpace với stages, currentStep
    'spreads'     → RENDER SpreadsCreativeSpace với spreads, characters, props, stages, currentStep, currentLanguage ⚡
    'objects'     → RENDER ObjectsCreativeSpace với spreads
    'animations'  → RENDER AnimationsCreativeSpace với spreads, currentLanguage ⚡
    'flags'       → RENDER FlagsCreativeSpace với flags
    'shares'      → RENDER SharesCreativeSpace với shareLinks
    'collabs'     → RENDER CollaboratorsCreativeSpace với collaborations, spreadsCount
    'config'      → RENDER ConfigCreativeSpace với book

  IF isSidebarOpen:
    RENDER RightSidebar với bookId, currentStep, activeCreativeSpace, currentLanguage, contextData, onClose
  ELSE:
    RENDER AISidebarToggle với onToggle (floating button bottom-right)
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 EditorHeader

📄 **Doc:** [01-editor-header.md](component/editor-page/01-editor-header.md)

**Mục đích:** Top navigation bar. Hiển thị book info, step navigation, language selector, và quick actions.

**Props & Callbacks:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  currentStep: Step;
  currentLanguage: Language;
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;
  onLanguageChange: (language: Language) => void;
  onTitleEdit: (newTitle: string) => void;
  onStepChange: (step: Step) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
}
```

---

### 3.2 IconRail

📄 **Doc:** [02-icon-rail.md](component/editor-page/02-icon-rail.md)

**Mục đích:** Sidebar navigation dọc bên trái chứa icons để chuyển giữa các CreativeSpace.

**Props & Callbacks:**

```typescript
interface IconRailProps {
  activeCreativeSpace: CreativeSpaceType;
  currentStep: Step;
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
}
```

---

### 3.3 ManuscriptCreativeSpace ⚡

📄 **Doc:** [03-manuscript-creative-space.md](component/editor-page/03-manuscript-creative-space.md)

**Mục đích:** Soạn thảo manuscript theo các bước: Brief → Draft → Script → Prose Dummy → Poetry Dummy → Finalization.

**Language impact:** ✅ Manuscript finalization step translate need current language

**Props & Callbacks:**

```typescript
interface ManuscriptCreativeSpaceProps {
  manuscript: Manuscript;
  currentLanguage: Language;  // ⚡ language-aware
  onManuscriptUpdate: (manuscript: Manuscript) => void;
}
```

---

### 3.4 CharactersCreativeSpace

📄 **Doc:** [04-characters-creative-space.md](component/editor-page/04-characters-creative-space.md)

**Mục đích:** Quản lý nhân vật: thông tin cơ bản, variants, voices, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Props & Callbacks:**

```typescript
interface CharactersCreativeSpaceProps {
  characters: Character[];
  currentStep: Step;
  onCharactersUpdate: (chars: Character[]) => void;
}
```

---

### 3.5 PropsCreativeSpace

📄 **Doc:** [05-props-creative-space.md](component/editor-page/05-props-creative-space.md)

**Mục đích:** Quản lý đạo cụ: states, sounds, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Props & Callbacks:**

```typescript
interface PropsCreativeSpaceProps {
  props: Prop[];
  currentStep: Step;
  onPropsUpdate: (props: Prop[]) => void;
}
```

---

### 3.6 StagesCreativeSpace

📄 **Doc:** [06-stages-creative-space.md](component/editor-page/06-stages-creative-space.md)

**Mục đích:** Quản lý bối cảnh: settings (temporal, sensory, emotional), sounds.

**Language impact:** ❌ Không bị ảnh hưởng

**Props & Callbacks:**

```typescript
interface StagesCreativeSpaceProps {
  stages: Stage[];
  currentStep: Step;
  onStagesUpdate: (stages: Stage[]) => void;
}
```

---

### 3.7 SpreadsCreativeSpace ⚡

📄 **Doc:** [07-spreads-creative-space.md](component/editor-page/07-spreads-creative-space.md)

**Mục đích:** Layout visual editor cho các trang đôi (spread). Quản lý images, textboxes.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content hiển thị theo `currentLanguage`

**Props & Callbacks:**

```typescript
interface SpreadsCreativeSpaceProps {
  spreads: Spread[];
  characters: Character[];
  props: Prop[];
  stages: Stage[];
  currentStep: Step;
  currentLanguage: Language;  // ⚡ language-aware
  onSpreadsUpdate: (spreads: Spread[]) => void;
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

📄 **Doc:** [08-objects-creative-space.md](component/editor-page/08-objects-creative-space.md)

**Mục đích:** Retouch layer management. Điều chỉnh vị trí, kích thước, z-index các object (image) trên spread.

**Language impact:** ❌ Không bị ảnh hưởng (chỉ image objects)

**Props & Callbacks:**

```typescript
interface ObjectsCreativeSpaceProps {
  spreads: Spread[];
  onSpreadsUpdate: (spreads: Spread[]) => void;
}
```

---

### 3.9 AnimationsCreativeSpace ⚡

📄 **Doc:** [09-animations-creative-space.md](component/editor-page/09-animations-creative-space.md)

**Mục đích:** Timeline editor cho animations. Quản lý trigger, delay, duration, effect types.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Animation list hiển thị textbox name/content theo `currentLanguage`

**Props & Callbacks:**

```typescript
interface AnimationsCreativeSpaceProps {
  spreads: Spread[];
  currentLanguage: Language;  // ⚡ language-aware
  onSpreadsUpdate: (spreads: Spread[]) => void;
}
```

---

### 3.10 FlagsCreativeSpace

📄 **Doc:** [10-flags-creative-space.md](component/editor-page/10-flags-creative-space.md)

**Mục đích:** Hiển thị và xử lý các vấn đề (quality warnings, consistency issues).

**Language impact:** ❌ Không bị ảnh hưởng

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

📄 **Doc:** [11-shares-creative-space.md](component/editor-page/11-shares-creative-space.md)

**Mục đích:** Quản lý share links (public preview, client review, team draft).

**Language impact:** ❌ Không bị ảnh hưởng

**Props & Callbacks:**

```typescript
interface SharesCreativeSpaceProps {
  shareLinks: ShareLink[];
  onShareLinksUpdate: (links: ShareLink[]) => void;
}
```

---

### 3.12 CollaboratorsCreativeSpace

📄 **Doc:** [12-collaborators-creative-space.md](component/editor-page/12-collaborators-creative-space.md)

**Mục đích:** Quản lý collaborators và permissions (languages, steps, spreads access).

**Language impact:** ❌ Không bị ảnh hưởng

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

📄 **Doc:** [13-config-creative-space.md](component/editor-page/13-config-creative-space.md)

**Mục đích:** Cấu hình book: general, creative, typography, layout, remix, export.

**Language impact:** ❌ Không bị ảnh hưởng (Remix section quản lý `book.remix.languages[]`)

**Props & Callbacks:**

```typescript
interface ConfigCreativeSpaceProps {
  book: Book;
  onBookUpdate: (updates: Partial<Book>) => void;
}
```

---

### 3.14 RightSidebar (AI Assistant) ⚡

📄 **Doc:** [14-right-sidebar.md](component/editor-page/14-right-sidebar.md)

**Mục đích:** Panel AI Assistant hỗ trợ người dùng. Contextual với creativeSpace hiện tại.

**Language impact:** ✅ AI biết user đang edit ngôn ngữ nào

**Props & Callbacks:**

```typescript
interface RightSidebarProps {
  isOpen: boolean;
  bookId: string;
  currentStep: Step;
  activeCreativeSpace: CreativeSpaceType;
  currentLanguage: Language;
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

📄 **Doc:** *(inline, không cần file riêng)*

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

## 4. Technical Notes

### 4.1 Key Design Decisions

**No Intermediate MainCreativeSpace Component**
EditorPage render trực tiếp creativeSpace dựa trên `activeCreativeSpace`. Lý do: MainCreativeSpace không có responsibility riêng ngoài routing, giảm props drilling, code đơn giản hơn.

**State Management**
EditorPage giữ toàn bộ state chính (book, snapshot, currentLanguage). Các creativeSpace nhận data qua props và báo thay đổi qua callbacks. Đảm bảo single source of truth và dễ implement autosave.

**Language as UI State**
`currentLanguage` là UI state (view preference), không phải data state. Nó quyết định ngôn ngữ nào được hiển thị trong editor, nhưng không thay đổi data structure của textbox.

**AI Sidebar Toggle as Floating Button**
`AISidebarToggle` là floating button ở góc dưới bên phải, hiển thị khi sidebar đóng. Khi sidebar mở, button ẩn đi và thay bằng nút X trong sidebar header để đóng.

**Static Language List**
Danh sách available languages lấy từ constant tĩnh, không phải từ `book.remix.languages[]`.

**CreativeSpace Isolation**
Mỗi creativeSpace có local state riêng (selected item, active tab, filter). State này không cần sync lên EditorPage.

**Conditional Rendering**
Render duy nhất một creativeSpace tại một thời điểm. Unmount creativeSpace cũ khi chuyển, nhưng EditorPage giữ data nên không mất state.

### 4.2 Step Transition Validation

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

### 4.3 Initial Language

Khi load Editor, `currentLanguage` mặc định là `book.original_language` hoặc language đầu tiên trong `AVAILABLE_LANGUAGES`.

### 4.4 Khi nào cần refactor thêm MainCreativeSpace?

Cân nhắc tách MainCreativeSpace nếu xuất hiện nhu cầu:
- Transition animation giữa các creativeSpace
- Shared toolbar/header riêng cho creativeSpace area
- Error boundary riêng (crash creativeSpace không crash toàn app)
- Lazy loading creativeSpaces (code splitting với Suspense)
- CreativeSpace state persistence (giữ state khi switch, không unmount)
