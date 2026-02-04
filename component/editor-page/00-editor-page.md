# Editor Page: Top-Level Component Architecture

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           EditorHeader                                 │  │
│  │  ┌────────┬─────────────┬──────────────────┬────────┬──────┬───────┐  │  │
│  │  │MenuBtn │ BookTitle   │ Pipeline (Step)  │SaveStat│Notif │MsgBtn │  │  │
│  │  └────────┴─────────────┴──────────────────┴────────┴──────┴───────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────┬──────────────────────────────────────────────┬──────────────┐  │
│  │        │                                               │              │  │
│  │        │  Conditional Render (trực tiếp):              │              │  │
│  │        │  ┌─────────────────────────────────────────┐  │   Right      │  │
│  │  Icon  │  │ DocumentWorkspace      (if docs)        │  │   Sidebar    │  │
│  │  Rail  │  │ CharactersWorkspace    (if characters)  │  │     (AI)     │  │
│  │        │  │ PropsWorkspace         (if props)       │  │              │  │
│  │        │  │ StagesWorkspace        (if stages)      │  │              │  │
│  │        │  │ SpreadsWorkspace       (if spreads) ⚡  │  │              │  │
│  │        │  │ ObjectsWorkspace       (if objects)     │  │              │  │
│  │        │  │ AnimationsWorkspace    (if animations)⚡ │  │              │  │
│  │        │  │ FlagsWorkspace         (if flags)       │  │              │  │
│  │        │  │ SharesWorkspace        (if shares)      │  │              │  │
│  │        │  │ CollaboratorsWorkspace (if collabs)     │  │              │  │
│  │        │  │ SettingsWorkspace      (if settings)    │  │              │  │
│  │        │  └─────────────────────────────────────────┘  │              │  │
│  └────────┴──────────────────────────────────────────────┴──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘

⚡ = Workspaces affected by currentLanguage
```

**Lưu ý:** Không có component MainWorkspace trung gian. EditorPage render trực tiếp workspace tương ứng với `activeWorkspace`.

### 1.2 Data Flow

```
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  State: book, snapshot, flags, shareLinks, collaborations               │ │
│  │         currentStep, activeWorkspace, currentLanguage,                  │ │
│  │         isSidebarOpen, hasUnsavedChanges                                │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│         │              │                              │                │     │
│         ▼              ▼                              ▼                ▼     │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────────────────┐ ┌─────────┐ │
│  │  Editor   │  │  Icon     │  │     Workspace (1 of 11)      │ │ Right   │ │
│  │  Header   │  │  Rail     │  │                              │ │ Sidebar │ │
│  │           │  │           │  │  Rendered directly based on  │ │         │ │
│  │ Props:    │  │ Props:    │  │  activeWorkspace state:      │ │ Props:  │ │
│  │ •bookTitle│  │ •active   │  │                              │ │ •isOpen │ │
│  │ •step     │  │  Workspace│  │  • DocumentWorkspace         │ │ •bookId │ │
│  │ •language │  │ •step     │  │  • CharactersWorkspace       │ │ •step   │ │
│  │ •unsaved  │  │           │  │  • PropsWorkspace            │ │ •active │ │
│  │           │  │ Callback: │  │  • StagesWorkspace           │ │  Work.. │ │
│  │ Callbacks:│  │ •onChange │  │  • SpreadsWorkspace    ⚡    │ │ •lang   │ │
│  │ •onSave   │  │           │  │  • ObjectsWorkspace          │ │ •context│ │
│  │ •onStep   │  │           │  │  • AnimationsWorkspace ⚡    │ │         │ │
│  │  Change   │  │           │  │  • FlagsWorkspace            │ │         │ │
│  │ •onLang   │  │           │  │  • SharesWorkspace           │ │         │ │
│  │  Change   │  │           │  │  • CollaboratorsWorkspace    │ └─────────┘ │
│  │ •onToggle │  │           │  │  • SettingsWorkspace         │             │
│  │  Sidebar  │  │           │  │                              │             │
│  └───────────┘  └───────────┘  │  ⚡ = receives currentLanguage│             │
│                                └──────────────────────────────┘             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step ↔ Workspace Mapping

Workspaces được enable dựa trên nguyên tắc "từ step X trở đi" (progressive unlock):

| Step | Newly Enabled | All Available Workspaces |
|------|---------------|--------------------------|
| `manuscript` | docs, spreads⚡, flags, shares, collabs, settings | docs, flags, shares, collabs, settings |
| `sketch` | characters, props, stages | docs, characters, props, stages, spreads⚡, flags, shares, collabs, settings |
| `illustration` | (none) | docs, characters, props, stages, spreads⚡, flags, shares, collabs, settings |
| `retouch` | objects, animations⚡ | docs, characters, props, stages, spreads⚡, objects, animations⚡, flags, shares, collabs, settings |

⚡ = Language-aware workspaces

**Logic:** `currentStep >= enabledFromStep`

### 1.4 Language Impact Summary

| Workspace | Receives `currentLanguage` | How it's used |
|-----------|---------------------------|---------------|
| DocumentWorkspace | ❌ | Docs are in original language only |
| CharactersWorkspace | ❌ | Character metadata not multilingual |
| PropsWorkspace | ❌ | Props metadata not multilingual |
| StagesWorkspace | ❌ | Stage metadata not multilingual |
| **SpreadsWorkspace** | ✅ | Filter `textbox.language[]` by `currentLanguage.code` |
| ObjectsWorkspace | ❌ | Only displays image objects, not textboxes |
| **AnimationsWorkspace** | ✅ | Show textbox names/preview in selected language |
| FlagsWorkspace | ❌ | Flags are language-agnostic |
| SharesWorkspace | ❌ | Share links are language-agnostic |
| CollaboratorsWorkspace | ❌ | Permissions reference languages but don't filter by current |
| SettingsWorkspace | ❌ | Manages `book.remix.languages[]` but doesn't filter |
| **RightSidebar** | ✅ | AI knows which language user is editing |

---

## 2. Component Designs

### 2.1 EditorPage (Root Component)

**Mục đích:** Container gốc điều phối toàn bộ Editor. Quản lý state toàn cục, fetch data từ API, và render trực tiếp các component con bao gồm workspace.

**Shared Types:**

```typescript
interface Language {
  name: string;       // "English", "Tiếng Việt", "日本語", "한국어", "中文"
  code: string;       // "en_US", "vi_VN", "ja_JP", "ko_KR", "zh_CN"
}

type Step = 'manuscript' | 'sketch' | 'illustration' | 'retouch';

type WorkspaceType =
  | 'docs' | 'characters' | 'props' | 'stages' | 'spreads'
  | 'objects' | 'animations' | 'flags' | 'shares' | 'collabs' | 'settings';

// constants/languages.ts
const AVAILABLE_LANGUAGES: Language[] = [
  { name: "English", code: "en_US" },
  { name: "Tiếng Việt", code: "vi_VN" },
  { name: "日本語", code: "ja_JP" },
  { name: "한국어", code: "ko_KR" },
  { name: "中文", code: "zh_CN" }
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
  activeWorkspace: WorkspaceType;
  currentLanguage: Language;
  isSidebarOpen: boolean;
  hasUnsavedChanges: boolean;
  isLoading: boolean;
}

interface EditorPageCallbacks {
  onStepChange: (step: Step) => void;
  onWorkspaceChange: (workspace: WorkspaceType) => void;
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
  RENDER IconRail với activeWorkspace, currentStep

  SWITCH activeWorkspace:
    'docs'        → RENDER DocumentWorkspace với documents
    'characters'  → RENDER CharactersWorkspace với characters, currentStep
    'props'       → RENDER PropsWorkspace với props, currentStep
    'stages'      → RENDER StagesWorkspace với stages, currentStep
    'spreads'     → RENDER SpreadsWorkspace với spreads, characters, props, stages, currentStep, currentLanguage ⚡
    'objects'     → RENDER ObjectsWorkspace với spreads
    'animations'  → RENDER AnimationsWorkspace với spreads, currentLanguage ⚡
    'flags'       → RENDER FlagsWorkspace với flags
    'shares'      → RENDER SharesWorkspace với shareLinks
    'collabs'     → RENDER CollaboratorsWorkspace với collaborations, spreadsCount
    'settings'    → RENDER SettingsWorkspace với book

  IF isSidebarOpen:
    RENDER RightSidebar với bookId, currentStep, activeWorkspace, currentLanguage, contextData
```

---

### 2.2 EditorHeader

**Mục đích:** Navigation bar phía trên. Hiển thị thông tin book, điều hướng giữa các step, và các action nhanh (save, notifications, toggle AI). Chứa Menu popover để chọn language.

**Interface:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  currentStep: Step;
  currentLanguage: Language;
  hasUnsavedChanges: boolean;
  notificationCount: number;
  onLanguageChange: (language: Language) => void;
  onTitleEdit: (newTitle: string) => void;
  onStepChange: (step: Step) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onToggleSidebar: () => void;
}

interface EditorHeaderLocalState {
  isEditingTitle: boolean;
  isSaving: boolean;
  isMenuOpen: boolean;
  activeSubmenu: 'language' | 'editor_mode' | null;
}
```

---

### 2.3 IconRail

**Mục đích:** Sidebar dọc bên trái chứa các icon navigation đến workspace khác nhau. Highlight active workspace. Disable các workspace chưa được unlock theo step.

**Interface:**

```typescript
interface IconRailProps {
  activeWorkspace: WorkspaceType;
  currentStep: Step;
  onWorkspaceChange: (workspace: WorkspaceType) => void;
}

interface IconRailItem {
  id: WorkspaceType;
  icon: string;
  label: string;
  enabledFromStep: Step;
}
```

**Configuration:**

```typescript
const STEP_ORDER: Record<Step, number> = {
  manuscript: 0,
  sketch: 1,
  illustration: 2,
  retouch: 3,
};

const ICON_RAIL_ITEMS: IconRailItem[] = [
  { id: 'docs',       icon: 'FileText',    label: 'Documents',     enabledFromStep: 'manuscript' },
  { id: 'characters', icon: 'Users',       label: 'Characters',    enabledFromStep: 'sketch' },
  { id: 'props',      icon: 'Package',     label: 'Props',         enabledFromStep: 'sketch' },
  { id: 'stages',     icon: 'Map',         label: 'Stages',        enabledFromStep: 'sketch' },
  { id: 'spreads',    icon: 'BookOpen',    label: 'Spreads',       enabledFromStep: 'manuscript' },
  { id: 'objects',    icon: 'Layers',      label: 'Objects',       enabledFromStep: 'retouch' },
  { id: 'animations', icon: 'Play',        label: 'Animations',    enabledFromStep: 'retouch' },
  { id: 'flags',      icon: 'Flag',        label: 'Flags',         enabledFromStep: 'manuscript' },
  { id: 'shares',     icon: 'Share',       label: 'Share Links',   enabledFromStep: 'manuscript' },
  { id: 'collabs',    icon: 'UserPlus',    label: 'Collaborators', enabledFromStep: 'manuscript' },
  { id: 'settings',   icon: 'Settings',    label: 'Settings',      enabledFromStep: 'manuscript' },
];

function isWorkspaceEnabled(item: IconRailItem, currentStep: Step): boolean {
  return STEP_ORDER[currentStep] >= STEP_ORDER[item.enabledFromStep];
}
```

---

### 2.4 Workspace Components

#### 2.4.1 DocumentWorkspace

**Mục đích:** Soạn thảo manuscript và các tài liệu hỗ trợ (story outline, author notes, research).

**Language impact:** ❌ Không bị ảnh hưởng (docs là ngôn ngữ gốc)

**Interface:**

```typescript
interface DocumentWorkspaceProps {
  documents: Doc[];
  onDocumentsUpdate: (docs: Doc[]) => void;
}

interface DocumentWorkspaceState {
  activeDocId: string;
  editorContent: string;
}
```

---

#### 2.4.2 CharactersWorkspace

**Mục đích:** Quản lý nhân vật: thông tin cơ bản, variants, voices, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface CharactersWorkspaceProps {
  characters: Character[];
  currentStep: Step;
  onCharactersUpdate: (chars: Character[]) => void;
}

interface CharactersWorkspaceState {
  selectedCharacterKey: string | null;
  activeTab: 'variants' | 'voices' | 'crops';
}
```

---

#### 2.4.3 PropsWorkspace

**Mục đích:** Quản lý đạo cụ: states, sounds, crops.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface PropsWorkspaceProps {
  props: Prop[];
  currentStep: Step;
  onPropsUpdate: (props: Prop[]) => void;
}

interface PropsWorkspaceState {
  selectedPropKey: string | null;
  activeTab: 'states' | 'sounds' | 'crops';
}
```

---

#### 2.4.4 StagesWorkspace

**Mục đích:** Quản lý bối cảnh: settings (temporal, sensory, emotional), sounds.

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface StagesWorkspaceProps {
  stages: Stage[];
  currentStep: Step;
  onStagesUpdate: (stages: Stage[]) => void;
}

interface StagesWorkspaceState {
  selectedStageKey: string | null;
  activeTab: 'settings' | 'sounds';
}
```

---

#### 2.4.5 SpreadsWorkspace ⚡

**Mục đích:** Layout visual editor cho các trang đôi (spread). Quản lý images, textboxes.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox content hiển thị theo `currentLanguage`. Workspace lọc `textbox.language[]` và hiển thị entry có `code === currentLanguage.code`.

**Interface:**

```typescript
interface SpreadsWorkspaceProps {
  spreads: Spread[];
  characters: Character[];
  props: Prop[];
  stages: Stage[];
  currentStep: Step;
  currentLanguage: Language;  // ⚡ language-aware
  onSpreadsUpdate: (spreads: Spread[]) => void;
}

interface SpreadsWorkspaceState {
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
      "key": "tb_001",
      "language": [
        {
          "code": "en_US",
          "text": "Once upon a time...",
          "geometry": { "x": 10, "y": 80, "w": 80, "h": 15, "rotation": 0 },
          "typography": { "size": 16, "font": "...", "color": "..." }
        },
        {
          "code": "vi_VN",
          "text": "Ngày xửa ngày xưa...",
          "geometry": { "x": 10, "y": 80, "w": 80, "h": 15, "rotation": 0 },
          "typography": { "size": 16, "font": "...", "color": "..." }
        }
      ]
    }
  ]
}
```

---

#### 2.4.6 ObjectsWorkspace

**Mục đích:** Retouch layer management. Điều chỉnh vị trí, kích thước, z-index các object (image) trên spread.

**Language impact:** ❌ Không bị ảnh hưởng (chỉ hiển thị image objects, không hiển thị textbox)

**Interface:**

```typescript
interface ObjectsWorkspaceProps {
  spreads: Spread[];
  onSpreadsUpdate: (spreads: Spread[]) => void;
}

interface ObjectsWorkspaceState {
  selectedSpreadNumber: number;
  selectedObjectId: string | null;
  zoom: number;
}
```

---

#### 2.4.7 AnimationsWorkspace ⚡

**Mục đích:** Timeline editor cho animations. Quản lý trigger, delay, duration.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Animation list hiển thị textbox name/content theo `currentLanguage`.

**Interface:**

```typescript
interface AnimationsWorkspaceProps {
  spreads: Spread[];
  currentLanguage: Language;  // ⚡ language-aware
  onSpreadsUpdate: (spreads: Spread[]) => void;
}

interface AnimationsWorkspaceState {
  selectedSpreadNumber: number;
  selectedAnimationIndex: number | null;
  isPreviewPlaying: boolean;
}
```

---

#### 2.4.8 FlagsWorkspace

**Mục đích:** Hiển thị và xử lý các vấn đề (quality warnings, consistency issues).

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface FlagsWorkspaceProps {
  flags: Flag[];
  onFlagsUpdate: (flags: Flag[]) => void;
  onNavigateToIssue: (flag: Flag) => void;
}

interface FlagsWorkspaceState {
  filterType: FlagType | 'all';
  filterStatus: FlagStatus | 'all';
}
```

---

#### 2.4.9 SharesWorkspace

**Mục đích:** Quản lý share links (public preview, client review, team draft).

**Language impact:** ❌ Không bị ảnh hưởng

**Interface:**

```typescript
interface SharesWorkspaceProps {
  shareLinks: ShareLink[];
  onShareLinksUpdate: (links: ShareLink[]) => void;
}

interface SharesWorkspaceState {
  selectedLinkId: string | null;
  isCreatingNew: boolean;
}
```

---

#### 2.4.10 CollaboratorsWorkspace

**Mục đích:** Quản lý collaborators và permissions (languages, steps, spreads access).

**Language impact:** ❌ Không bị ảnh hưởng (nhưng hiển thị danh sách languages trong permissions)

**Interface:**

```typescript
interface CollaboratorsWorkspaceProps {
  collaborations: Collaboration[];
  spreadsCount: number;
  onCollaborationsUpdate: (collabs: Collaboration[]) => void;
}

interface CollaboratorsWorkspaceState {
  selectedCollabId: string | null;
  isInviting: boolean;
}
```

---

#### 2.4.11 SettingsWorkspace

**Mục đích:** Cấu hình book: general, creative, typography, layout, remix, export.

**Language impact:** ❌ Không bị ảnh hưởng (nhưng Remix section quản lý `book.remix.languages[]`)

**Interface:**

```typescript
interface SettingsWorkspaceProps {
  book: Book;
  onBookUpdate: (updates: Partial<Book>) => void;
}

interface SettingsWorkspaceState {
  activeSection: 'general' | 'creative' | 'typography' | 'layout' | 'remix' | 'export';
}
```

---

### 2.5 RightSidebar (AI Assistant)

**Mục đích:** Panel AI Assistant hỗ trợ người dùng. Contextual với workspace hiện tại.

**Interface:**

```typescript
interface RightSidebarProps {
  isOpen: boolean;
  bookId: string;
  currentStep: Step;
  activeWorkspace: WorkspaceType;
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

## 3. Technical Notes

### 3.1 Key Design Decisions

**No Intermediate MainWorkspace Component**
EditorPage render trực tiếp workspace dựa trên `activeWorkspace`. Lý do: MainWorkspace không có responsibility riêng ngoài routing, giảm props drilling, code đơn giản hơn.

**State Management**
EditorPage giữ toàn bộ state chính (book, snapshot, currentLanguage). Các workspace nhận data qua props và báo thay đổi qua callbacks. Đảm bảo single source of truth và dễ implement autosave.

**Language as UI State**
`currentLanguage` là UI state (view preference), không phải data state. Nó quyết định ngôn ngữ nào được hiển thị trong editor, nhưng không thay đổi data structure của textbox.

**Menu State is Local**
`isMenuOpen` là local state của EditorHeader, không cần lift lên EditorPage vì menu chỉ ảnh hưởng trong phạm vi EditorHeader.

**Static Language List**
Danh sách available languages lấy từ constant tĩnh (định nghĩa riêng), không phải từ `book.remix.languages[]`. Đơn giản hóa logic và không phụ thuộc vào book data.

**Workspace Isolation**
Mỗi workspace có local state riêng (selected item, active tab, filter). State này không cần sync lên EditorPage vì chỉ phục vụ UI của workspace đó.

**Conditional Rendering**
Render duy nhất một workspace tại một thời điểm. Unmount workspace cũ khi chuyển, nhưng EditorPage giữ data nên không mất state.

### 3.2 Initial Language
Khi load Editor, `currentLanguage` mặc định là `book.original_language` hoặc language đầu tiên trong `AVAILABLE_LANGUAGES`.

### 3.3 Khi nào cần refactor thêm MainWorkspace?

Cân nhắc tách MainWorkspace nếu xuất hiện nhu cầu:
- Transition animation giữa các workspace
- Shared toolbar/header riêng cho workspace area
- Error boundary riêng (crash workspace không crash toàn app)
- Lazy loading workspaces (code splitting với Suspense)
- Workspace state persistence (giữ state khi switch, không unmount)
