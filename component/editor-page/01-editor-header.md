# EditorHeader: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    EditorHeader                                      │
│  ┌────────┬────────────────┬────────────────────────┬──────────┬────────┬────────┐  │
│  │MenuBtn │   BookTitle    │     StepBreadcrumb     │SaveStatus│NotifBtn│AIToggle│  │
│  │   ≡    │The Hidden Val..│ M > S > I > R          │ ✓ Saved  │   🔔   │   💬   │  │
│  └────────┴────────────────┴────────────────────────┴──────────┴────────┴────────┘  │
│                                                                                      │
│  ┌────────────────────────────────┐                                                  │
│  │         MenuPopover            │  (when isMenuOpen = true)                        │
│  │  ┌──────────────────────────┐  │                                                  │
│  │  │      PointsDisplay       │  │  750 / 1000 [████████░░]                         │
│  │  ├──────────────────────────┤  │                                                  │
│  │  │      ← Home              │  │                                                  │
│  │  ├──────────────────────────┤  │                                                  │
│  │  │   🌐 Language        >   │──┼──► LanguageSubmenu                               │
│  │  ├──────────────────────────┤  │    ┌─────────────────┐                           │
│  │  │   ⚙️ Editor Mode     >   │──┼──► │ ✓ English (US)  │                           │
│  │  └──────────────────────────┘  │    │   Tiếng Việt    │                           │
│  └────────────────────────────────┘    │   日本語         │                           │
│                                        │   한국어         │                           │
│                                        │   中文 (简体)    │                           │
│                                        └─────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                    EditorHeader                                       │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: bookTitle, currentStep, currentLanguage, hasUnsavedChanges,            │  │
│  │         notificationCount, userPoints                                          │  │
│  │  LocalState: isMenuOpen, activeSubmenu, isEditingTitle, isSaving               │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│           │                │                  │                │                     │
│           ▼                ▼                  ▼                ▼                     │
│    ┌───────────┐    ┌───────────┐      ┌───────────┐    ┌───────────┐               │
│    │ MenuBtn   │    │ BookTitle │      │   Step    │    │  Actions  │               │
│    │           │    │           │      │ Breadcrumb│    │   Group   │               │
│    │ onClick:  │    │ Props:    │      │           │    │           │               │
│    │ toggle    │    │ •title    │      │ Props:    │    │ Props:    │               │
│    │ isMenuOpen│    │ •isEditing│      │ •current  │    │ •unsaved  │               │
│    │           │    │           │      │  Step     │    │ •notifCnt │               │
│    │           │    │ Callback: │      │           │    │ •sideOpen │               │
│    │           │    │ •onEdit   │      │ Callback: │    │           │               │
│    │           │    │           │      │ •onChange │    │ Callbacks:│               │
│    └───────────┘    └───────────┘      └───────────┘    │ •onSave   │               │
│           │                                             │ •onNotif  │               │
│           ▼                                             │ •onToggle │               │
│    ┌───────────┐                                        └───────────┘               │
│    │MenuPopover│                                                                     │
│    │ Props:    │                                                                     │
│    │ •isOpen   │                                                                     │
│    │ •points   │                                                                     │
│    │ •language │                                                                     │
│    │ •editorMd │                                                                     │
│    │ •submenu  │                                                                     │
│    │           │                                                                     │
│    │ Callbacks:│                                                                     │
│    │ •onClose  │                                                                     │
│    │ •onHome   │                                                                     │
│    │ •onLang   │                                                                     │
│    │ •onMode   │                                                                     │
│    └───────────┘                                                                     │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step Breadcrumb Visual States

```
Step: manuscript (active)
┌───────────────────────────────────────────────────────────────┐
│  [Manuscript]  >  Sketch  >  Illustration  >  Retouch         │
│   ▲ active        dim         dim             dim             │
│   (solid bg)      (clickable) (clickable)     (clickable)     │
└───────────────────────────────────────────────────────────────┘

Step: illustration (active)
┌───────────────────────────────────────────────────────────────┐
│  Manuscript  >  Sketch  >  [Illustration]  >  Retouch         │
│   clickable     clickable    ▲ active          clickable      │
└───────────────────────────────────────────────────────────────┘
```

---

## 2. Component Designs

### 2.1 EditorHeader (Root Component)

**Mục đích:** Top navigation bar. Hiển thị book info, step navigation, và quick actions (save, notifications, AI toggle). Chứa Menu popover để chọn language/editor mode.

**Shared Types:**

```typescript
type Step = 'manuscript' | 'sketch' | 'illustration' | 'retouch';
type EditorMode = 'edit' | 'read';
type SubmenuType = 'language' | 'editor_mode' | null;

interface Language {
  name: string;       // "English (US)", "Tiếng Việt"
  code: string;       // "en_US", "vi_VN"
}

interface UserPoints {
  current: number;    // 750
  total: number;      // 1000
}
```

**Interface:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  currentStep: Step;
  currentLanguage: Language;
  hasUnsavedChanges: boolean;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;
  isSidebarOpen: boolean;
  onLanguageChange: (language: Language) => void;
  onTitleEdit: (newTitle: string) => void;
  onStepChange: (step: Step) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onToggleSidebar: () => void;
  onEditorModeChange: (mode: EditorMode) => void;
  onNavigateHome: () => void;
}

interface EditorHeaderState {
  isMenuOpen: boolean;
  activeSubmenu: SubmenuType;
  isEditingTitle: boolean;
  isSaving: boolean;
}
```

**Render Logic (pseudo):**

```
EditorHeader:
  RENDER header container (h-14, border-b, flex items-center justify-between)

    // Left section
    RENDER div.flex.items-center.gap-3
      RENDER MenuButton với onClick → toggle isMenuOpen
      RENDER BookTitle với bookTitle, isEditingTitle, onTitleEdit

    // Center section
    RENDER StepBreadcrumb với currentStep, onStepChange

    // Right section
    RENDER div.flex.items-center.gap-2
      RENDER SaveStatus với hasUnsavedChanges, isSaving
      RENDER NotificationButton với notificationCount, onNotificationClick
      RENDER AISidebarToggle với isSidebarOpen, onToggleSidebar

  IF isMenuOpen:
    RENDER MenuPopover với props, activeSubmenu, callbacks
```

---

### 2.2 MenuButton

**Mục đích:** Hamburger icon button mở Menu popover.

**Interface:**

```typescript
interface MenuButtonProps {
  onClick: () => void;
}
```

**Visual:** Icon `Menu` (☰) từ Lucide, 24x24px.

---

### 2.3 BookTitle

**Mục đích:** Hiển thị và cho phép edit book title inline.

**Interface:**

```typescript
interface BookTitleProps {
  title: string;
  isEditing: boolean;
  onStartEdit: () => void;
  onEndEdit: (newTitle: string) => void;
  onCancel: () => void;
}

interface BookTitleState {
  editValue: string;
}
```

**Behavior:**

- Default: Hiển thị title dạng text, truncate nếu quá dài (max ~200px)
- Click: Chuyển sang input mode
- Enter/Blur: Save và exit edit mode
- Escape: Cancel edit, restore original

---

### 2.4 StepBreadcrumb

**Mục đích:** Breadcrumb navigation giữa 4 steps. Click để jump đến step bất kỳ.

**Interface:**

```typescript
interface StepBreadcrumbProps {
  currentStep: Step;
  onStepChange: (step: Step) => void;
}
```

**Configuration:**

```typescript
const STEPS: { id: Step; label: string }[] = [
  { id: 'manuscript', label: 'Manuscript' },
  { id: 'sketch', label: 'Sketch' },
  { id: 'illustration', label: 'Illustration' },
  { id: 'retouch', label: 'Retouch' },
];
```

**Visual States:**

| State | Style |
|-------|-------|
| Active | `bg-primary text-primary-foreground rounded-md px-3 py-1` |
| Inactive | `text-muted-foreground hover:text-foreground cursor-pointer` |
| Separator | `>` icon, `text-muted-foreground mx-2` |

---

### 2.5 SaveStatus

**Mục đích:** Indicator hiển thị trạng thái save.

**Interface:**

```typescript
interface SaveStatusProps {
  hasUnsavedChanges: boolean;
  isSaving: boolean;
  onSave: () => void;
}
```

**Visual States:**

| State | Display |
|-------|---------|
| Saved | `✓ Saved` (green check + text) |
| Unsaved | `● Unsaved` (yellow dot + text, clickable) |
| Saving | `○ Saving...` (spinner + text) |

---

### 2.6 NotificationButton

**Mục đích:** Bell icon với badge count, mở notification panel.

**Interface:**

```typescript
interface NotificationButtonProps {
  count: number;
  onClick: () => void;
}
```

**Visual:**

- Icon: `Bell` từ Lucide
- Badge: Hiển thị nếu count > 0, max display "9+"

---

### 2.7 AISidebarToggle

**Mục đích:** Toggle button mở/đóng AI Assistant sidebar.

**Interface:**

```typescript
interface AISidebarToggleProps {
  isOpen: boolean;
  onToggle: () => void;
}
```

**Visual:**

- Icon: `MessageCircle` từ Lucide
- Active state: `bg-primary text-primary-foreground`
- Inactive state: `text-muted-foreground hover:text-foreground`

---

### 2.8 MenuPopover

**Mục đích:** Popover menu chứa navigation, language selector, editor mode. Hỗ trợ nested submenu.

**Interface:**

```typescript
interface MenuPopoverProps {
  isOpen: boolean;
  userPoints: UserPoints;
  currentLanguage: Language;
  editorMode: EditorMode;
  activeSubmenu: SubmenuType;
  onClose: () => void;
  onNavigateHome: () => void;
  onLanguageChange: (language: Language) => void;
  onEditorModeChange: (mode: EditorMode) => void;
  onSubmenuChange: (submenu: SubmenuType) => void;
}
```

**Menu Items Configuration:**

```typescript
interface MenuItem {
  id: string;
  icon: string;
  label: string;
  type: 'action' | 'submenu';
  submenuId?: SubmenuType;
}

const MENU_ITEMS: MenuItem[] = [
  { id: 'home', icon: 'ArrowLeft', label: 'Home', type: 'action' },
  { id: 'language', icon: 'Globe', label: 'Language', type: 'submenu', submenuId: 'language' },
  { id: 'editor_mode', icon: 'Settings2', label: 'Editor Mode', type: 'submenu', submenuId: 'editor_mode' },
];

const AVAILABLE_LANGUAGES: Language[] = [
  { name: 'English (US)', code: 'en_US' },
  { name: 'Tiếng Việt', code: 'vi_VN' },
  { name: '日本語', code: 'ja_JP' },
  { name: '한국어', code: 'ko_KR' },
  { name: '中文 (简体)', code: 'zh_CN' },
];

const EDITOR_MODES: { id: EditorMode; label: string; description: string }[] = [
  { id: 'edit', label: 'Edit Mode', description: 'Full editing capabilities' },
  { id: 'read', label: 'Read Mode', description: 'View only, no changes' },
];
```

**Render Structure:**

```
MenuPopover:
  RENDER Popover container (w-64, shadow-lg, rounded-lg)

    // Points section
    RENDER PointsDisplay với userPoints

    RENDER Separator

    // Menu items
    FOR item IN MENU_ITEMS:
      IF item.type === 'action':
        RENDER MenuItem với icon, label, onClick
      ELSE IF item.type === 'submenu':
        RENDER SubmenuTrigger với icon, label, chevron

    // Submenu (positioned to the right)
    IF activeSubmenu === 'language':
      RENDER LanguageSubmenu với languages, current, onChange
    ELSE IF activeSubmenu === 'editor_mode':
      RENDER EditorModeSubmenu với modes, current, onChange
```

---

### 2.9 PointsDisplay

**Mục đích:** Hiển thị user points với progress bar trong MenuPopover.

**Interface:**

```typescript
interface PointsDisplayProps {
  current: number;
  total: number;
}
```

**Visual:**

```
┌─────────────────────────────────┐
│  ✨ Points        750 / 1000    │
│  [████████████░░░░░░░░]  75%    │
└─────────────────────────────────┘
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

**Menu State is Local**
`isMenuOpen` và `activeSubmenu` là local state của EditorHeader. Không cần lift lên EditorPage vì menu chỉ ảnh hưởng UI trong phạm vi EditorHeader.

**Submenu Positioning**
Submenu render bên phải của parent menu item khi hover/click. Sử dụng absolute positioning với offset để không overlap với main menu.

**Points from User Context**
`userPoints` lấy từ user session/context, không phải từ book data. EditorPage truyền xuống như prop.

**Click Outside to Close**
MenuPopover đóng khi click outside. Sử dụng portal để render popover ở root level, tránh z-index issues.

### 3.2 Accessibility

- MenuButton: `aria-expanded`, `aria-haspopup="menu"`
- MenuPopover: `role="menu"`, keyboard navigation (↑↓ arrows, Enter, Escape)
- StepBreadcrumb: `aria-current="step"` cho active step
- All interactive elements: visible focus states

### 3.3 Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| Desktop (>1024px) | Full layout như design |
| Tablet (768-1024px) | Truncate BookTitle, hide step labels (chỉ hiện icons) |
| Mobile (<768px) | Hide StepBreadcrumb, show in Menu instead |

### 3.4 Animation

- MenuPopover: `animate-in fade-in-0 zoom-in-95` (150ms)
- Submenu: `animate-in slide-in-from-left-2` (100ms)
- SaveStatus transition: `transition-colors duration-200`
