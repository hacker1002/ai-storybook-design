# Editor Page: Top-Level Component Architecture

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
│  │  Icon  │  │ ManuscriptCreativeSpace   (if manuscripts) ⚡│  │     (AI)     │  │
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
│  │ •step     │  │  CreativeSpace│  │  • ManuscriptCreativeSpace ⚡     │ │ •bookId │ │
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
| `idea` | manuscripts, flags, shares, collabs, config | manuscripts, flags, shares, collabs, config |
| `sketch` | characters, props, stages, spreads⚡ | manuscripts, characters, props, stages, spreads⚡, flags, shares, collabs, config |
| `illustration` | (none) | manuscripts, characters, props, stages, spreads⚡, flags, shares, collabs, config |
| `retouch` | objects, animations⚡ | manuscripts, characters, props, stages, spreads⚡, objects, animations⚡, flags, shares, collabs, config |

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

## 2. Component Designs

### 2.1 EditorPage (Root Component)

**Mục đích:** Container gốc điều phối toàn bộ Editor. Quản lý state toàn cục, fetch data từ API, và render trực tiếp các component con bao gồm creativeSpace.

**Shared Types:**

```typescript
interface Language {
  name: string;       // "English", "Tiếng Việt", "日本語", "한국어", "中文"
  code: string;       // "en_US", "vi_VN", "ja_JP", "ko_KR", "zh_CN"
}

type Step = 'idea' | 'sketch' | 'illustration' | 'retouch';

type CreativeSpaceType =
  | 'manuscripts' | 'characters' | 'props' | 'stages' | 'spreads'
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

**Interface:**

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

**Render Logic (pseudo):**
```
EditorPage:
  RENDER EditorHeader với bookTitle, currentStep, currentLanguage, callbacks
  RENDER IconRail với activeCreativeSpace, currentStep

  SWITCH activeCreativeSpace:
    'manuscripts' → RENDER ManuscriptCreativeSpace với manuscripts, currentLanguage ⚡
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

### 2.2 EditorHeader

**Mục đích:** Navigation bar phía trên. Hiển thị thông tin book, điều hướng giữa các step, language selector, và các action nhanh (save, notifications). Chứa Menu popover hiển thị points, home link, và editor mode (display only).

**Interface:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  currentStep: Step;
  currentLanguage: Language;
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;             // Display only in menu
  onLanguageChange: (language: Language) => void;
  onTitleEdit: (newTitle: string) => void;
  onStepChange: (step: Step) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
}

interface EditorHeaderLocalState {
  isEditingTitle: boolean;
  isSaving: boolean;
  isMenuOpen: boolean;
}
```

---

### 2.3 IconRail

**Mục đích:** Sidebar dọc bên trái chứa các icon navigation đến creativeSpace khác nhau. Highlight active creativeSpace. Disable các creativeSpace chưa được unlock theo step.

**Interface:**

```typescript
interface IconRailProps {
  activeCreativeSpace: CreativeSpaceType;
  currentStep: Step;
  onCreativeSpaceChange: (creativeSpace: CreativeSpaceType) => void;
}

interface IconRailItem {
  id: CreativeSpaceType;
  icon: string;
  label: string;
  enabledFromStep: Step;
}
```

**Configuration:**

```typescript
const STEP_ORDER: Record<Step, number> = {
  idea: 0,
  sketch: 1,
  illustration: 2,
  retouch: 3,
};

const ICON_RAIL_ITEMS: IconRailItem[] = [
  { id: 'manuscripts', icon: 'FileText',   label: 'Manuscripts',   enabledFromStep: 'idea' },
  { id: 'characters',  icon: 'Smile',      label: 'Characters',    enabledFromStep: 'sketch' },
  { id: 'props',       icon: 'Box',        label: 'Props',         enabledFromStep: 'sketch' },
  { id: 'stages',      icon: 'Mountain',   label: 'Stages',        enabledFromStep: 'sketch' },
  { id: 'spreads',     icon: 'BookOpen',   label: 'Spreads',       enabledFromStep: 'sketch' },
  { id: 'objects',     icon: 'Layers',     label: 'Objects',       enabledFromStep: 'retouch' },
  { id: 'animations',  icon: 'Zap',        label: 'Animations',    enabledFromStep: 'retouch' },
  { id: 'flags',       icon: 'Flag',       label: 'Flags',         enabledFromStep: 'idea' },
  { id: 'shares',      icon: 'Share2',     label: 'Share Links',   enabledFromStep: 'idea' },
  { id: 'collabs',     icon: 'Users',      label: 'Collaborators', enabledFromStep: 'idea' },
  { id: 'config',      icon: 'Settings',   label: 'Settings',      enabledFromStep: 'idea' },
];

function isCreativeSpaceEnabled(item: IconRailItem, currentStep: Step): boolean {
  return STEP_ORDER[currentStep] >= STEP_ORDER[item.enabledFromStep];
}
```

---

### 2.4 CreativeSpace Components

#### 2.4.1 ManuscriptCreativeSpace

**Mục đích:** Soạn thảo manuscript theo các bước: Brief → Draft → Script → Prose Dummy → Poetry Dummy → Finalization.

**Language impact:** ✅ Manuscript finalization step translate need current language

**Interface:**

```typescript
type ManuscriptStepType = 'brief' | 'draft' | 'script' | 'prose_dummy' | 'poetry_dummy' | 'finalization';

interface ManuscriptCreativeSpaceProps {
  manuscripts: Manuscript[];
  onManuscriptsUpdate: (manuscripts: Manuscript[]) => void;
}

interface ManuscriptCreativeSpaceState {
  activeStep: ManuscriptStepType;
  editorContent: string;
  promptInput: string;         // For Brief step AI generation
  isGenerating: boolean;
}
```

**Manuscript Steps:**

| Step | Type | Description |
|------|------|-------------|
| Brief | doc | Prompt input + AI generate story idea |
| Draft | doc | Full narrative draft |
| Script | doc | Scene-by-scene breakdown |
| Prose Dummy | dummy | Spread layout với prose text |
| Poetry Dummy | dummy | Spread layout với poetry text |
| Finalization | dummy | Visual direction notes cho dummy |

---

#### 2.4.2 CharactersCreativeSpace

**Mục đích:** Quản lý nhân vật: thông tin cơ bản, variants, voices, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface CharactersCreativeSpaceProps {
  characters: Character[];
  currentStep: Step;
  onCharactersUpdate: (chars: Character[]) => void;
}

interface CharactersCreativeSpaceState {
  selectedCharacterKey: string | null;
  activeTab: 'variants' | 'voices' | 'crops';
}
```

---

#### 2.4.3 PropsCreativeSpace

**Mục đích:** Quản lý đạo cụ: states, sounds, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface PropsCreativeSpaceProps {
  props: Prop[];
  currentStep: Step;
  onPropsUpdate: (props: Prop[]) => void;
}

interface PropsCreativeSpaceState {
  selectedPropKey: string | null;
  activeTab: 'states' | 'sounds' | 'crops';
}
```

---

#### 2.4.4 StagesCreativeSpace

**Mục đích:** Quản lý bối cảnh: settings (temporal, sensory, emotional), sounds.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface StagesCreativeSpaceProps {
  stages: Stage[];
  currentStep: Step;
  onStagesUpdate: (stages: Stage[]) => void;
}

interface StagesCreativeSpaceState {
  selectedStageKey: string | null;
  activeTab: 'settings' | 'sounds';
}
```

---

#### 2.4.5 SpreadsCreativeSpace ⚡

**Mục đích:** Layout visual editor cho các trang đôi (spread). Quản lý images, textboxes.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content hiển thị theo `currentLanguage`. CreativeSpace lọc `textbox.language[]` và hiển thị entry có `code === currentLanguage.code`.

**Interface:**

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

interface SpreadsCreativeSpaceState {
  selectedSpreadNumber: number;
  selectedElementId: string | null;
  zoom: number;
}
```

**Textbox Language Structure:**

```json
{
  "textboxes": [
    {
      "id": "tb_001",
      "title": "Opening narration",
      "en_US": {
        "text": "Once upon a time...",
        "geometry": { "x": 10, "y": 80, "w": 80, "h": 15, "rotation": 0 },
        "typography": { "size": 16, "font": "...", "color": "..." }
      },
      "vi_VN": {
        "text": "Ngày xửa ngày xưa...",
        "geometry": { "x": 10, "y": 80, "w": 80, "h": 15, "rotation": 0 },
        "typography": { "size": 16, "font": "...", "color": "..." }
      }
    }
  ]
}
```

**Note:** Language content accessed via `textbox[currentLanguage.code]` instead of filtering array.

---

#### 2.4.6 ObjectsCreativeSpace

**Mục đích:** Retouch layer management. Điều chỉnh vị trí, kích thước, z-index các object (image) trên spread.

**Language impact:** ❌ Không bị ảnh hưởng (chỉ hiển thị image objects, không hiển thị textbox)

**Interface:**

```typescript
interface ObjectsCreativeSpaceProps {
  spreads: Spread[];
  onSpreadsUpdate: (spreads: Spread[]) => void;
}

interface ObjectsCreativeSpaceState {
  selectedSpreadNumber: number;
  selectedObjectId: string | null;
  zoom: number;
}
```

---

#### 2.4.7 AnimationsCreativeSpace ⚡

**Mục đích:** Timeline editor cho animations. Quản lý trigger, delay, duration, effect types.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Animation list hiển thị textbox name/content theo `currentLanguage`.

**Interface:**

```typescript
interface AnimationsCreativeSpaceProps {
  spreads: Spread[];
  currentLanguage: Language;  // ⚡ language-aware
  onSpreadsUpdate: (spreads: Spread[]) => void;
}

interface AnimationsCreativeSpaceState {
  selectedSpreadNumber: number;
  selectedAnimationIndex: number | null;
  isPreviewPlaying: boolean;
}
```

**Animation Effect Structure:**

```json
{
  "animations": [
    {
      "target_id": "img_001",
      "trigger": "tap",
      "delay": 0,
      "duration": 500,
      "loop": 1,
      "effect": {
        "type": "moving",
        "geometry": { "x": 100, "y": 50, "w": 200, "h": 150 }
      }
    }
  ]
}
```

**Effect Types:** `fade_in`, `fade_out`, `scale`, `rotate`, `moving`

---

#### 2.4.8 FlagsCreativeSpace

**Mục đích:** Hiển thị và xử lý các vấn đề (quality warnings, consistency issues).

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface FlagsCreativeSpaceProps {
  flags: Flag[];
  onFlagsUpdate: (flags: Flag[]) => void;
  onNavigateToIssue: (flag: Flag) => void;
}

interface FlagsCreativeSpaceState {
  filterType: FlagType | 'all';
  filterStatus: FlagStatus | 'all';
}
```

---

#### 2.4.9 SharesCreativeSpace

**Mục đích:** Quản lý share links (public preview, client review, team draft).

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface SharesCreativeSpaceProps {
  shareLinks: ShareLink[];
  onShareLinksUpdate: (links: ShareLink[]) => void;
}

interface SharesCreativeSpaceState {
  selectedLinkId: string | null;
  isCreatingNew: boolean;
}
```

---

#### 2.4.10 CollaboratorsCreativeSpace

**Mục đích:** Quản lý collaborators và permissions (languages, steps, spreads access).

**Language impact:** ❌ Không bị ảnh hưởng (nhưng hiển thị danh sách languages trong permissions)

**Interface:**

```typescript
interface CollaboratorsCreativeSpaceProps {
  collaborations: Collaboration[];
  spreadsCount: number;
  onCollaborationsUpdate: (collabs: Collaboration[]) => void;
}

interface CollaboratorsCreativeSpaceState {
  selectedCollabId: string | null;
  isInviting: boolean;
}
```

---

#### 2.4.11 ConfigCreativeSpace

**Mục đích:** Cấu hình book: general, creative, typography, layout, remix, export.

**Language impact:** ❌ Không bị ảnh hưởng (nhưng Remix section quản lý `book.remix.languages[]`)

**Interface:**

```typescript
interface ConfigCreativeSpaceProps {
  book: Book;
  onBookUpdate: (updates: Partial<Book>) => void;
}

interface ConfigCreativeSpaceState {
  activeSection: 'general' | 'creative' | 'typography' | 'layout' | 'remix' | 'export';
}
```

---

### 2.5 RightSidebar (AI Assistant)

**Mục đích:** Panel AI Assistant hỗ trợ người dùng. Contextual với creativeSpace hiện tại. Hiển thị khi `isSidebarOpen = true`, có nút X để đóng.

**Interface:**

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

interface RightSidebarState {
  conversationId: string | null;
  messages: AIMessage[];
  isLoading: boolean;
  inputValue: string;
}
```

---

### 2.6 AISidebarToggle

**Mục đích:** Floating button ở góc dưới bên phải để mở AI Assistant sidebar. Hiển thị khi right sidebar đang đóng, ẩn đi khi right sidebar open.

**Interface:**

```typescript
interface AISidebarToggleProps {
  onToggle: () => void;
}
```

**Visual:**

```
┌─────────────────────────────────────────┐
│                                         │
│                                         │
│                                         │
│                                         │
│                                         │
│                                 ┌─────┐ │
│                                 │ 💬  │ │  ← Floating button
│                                 └─────┘ │     position: fixed
│                                         │     bottom-right
└─────────────────────────────────────────┘
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

**No Intermediate MainCreativeSpace Component**
EditorPage render trực tiếp creativeSpace dựa trên `activeCreativeSpace`. Lý do: MainCreativeSpace không có responsibility riêng ngoài routing, giảm props drilling, code đơn giản hơn.

**State Management**
EditorPage giữ toàn bộ state chính (book, snapshot, currentLanguage). Các creativeSpace nhận data qua props và báo thay đổi qua callbacks. Đảm bảo single source of truth và dễ implement autosave.

**Language as UI State**
`currentLanguage` là UI state (view preference), không phải data state. Nó quyết định ngôn ngữ nào được hiển thị trong editor, nhưng không thay đổi data structure của textbox.

**Menu State is Local**
`isMenuOpen` là local state của EditorHeader, không cần lift lên EditorPage vì menu chỉ ảnh hưởng trong phạm vi EditorHeader.

**Language Selector on Header**
Language selector đặt trực tiếp trên header (không trong menu) vì là action thường xuyên sử dụng khi edit multi-language content. Giảm số click cần thiết từ 3 xuống 2.

**AI Sidebar Toggle as Floating Button**
`AISidebarToggle` là floating button ở góc dưới bên phải, hiển thị khi sidebar đóng. Khi sidebar mở, button ẩn đi và thay bằng nút X trong sidebar header để đóng. Pattern này phổ biến cho chat/assistant UI.

**Static Language List**
Danh sách available languages lấy từ constant tĩnh (định nghĩa riêng), không phải từ `book.remix.languages[]`. Đơn giản hóa logic và không phụ thuộc vào book data.

**CreativeSpace Isolation**
Mỗi creativeSpace có local state riêng (selected item, active tab, filter). State này không cần sync lên EditorPage vì chỉ phục vụ UI của creativeSpace đó.

**Conditional Rendering**
Render duy nhất một creativeSpace tại một thời điểm. Unmount creativeSpace cũ khi chuyển, nhưng EditorPage giữ data nên không mất state.

### 3.2 Step Transition Validation

**Mục đích:** Kiểm tra dữ liệu bắt buộc trước khi cho phép chuyển step trong pipeline.

**Interface:**

```typescript
interface ValidationResult {
  valid: boolean;
  missingFields?: string[];  // Field IDs để UI highlight
  message?: string;          // User-facing message
}

interface StepTransitionRule {
  from: Step;
  to: Step;
  validate: (book: Book, snapshot: Snapshot) => ValidationResult;
}

// Helper function
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

**Note:** Validation rules được định nghĩa sau. Chi tiết các required fields cho mỗi transition sẽ được thiết kế riêng.

### 3.3 Initial Language
Khi load Editor, `currentLanguage` mặc định là `book.original_language` hoặc language đầu tiên trong `AVAILABLE_LANGUAGES`.

### 3.4 Khi nào cần refactor thêm MainCreativeSpace?

Cân nhắc tách MainCreativeSpace nếu xuất hiện nhu cầu:
- Transition animation giữa các creativeSpace
- Shared toolbar/header riêng cho creativeSpace area
- Error boundary riêng (crash creativeSpace không crash toàn app)
- Lazy loading creativeSpaces (code splitting với Suspense)
- CreativeSpace state persistence (giữ state khi switch, không unmount)
