# EditorHeader: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                      EditorHeader                                         │
│  ┌────────┬────────────────┬────────────────────────┬──────────┬────────────┬─────────┐  │
│  │MenuBtn │   [BookTitle]  │     StepBreadcrumb     │SaveStatus│ LangSelect │ NotifBtn│  │
│  │   ≡    │The Hidden Val..│ [I] > S > I > R        │ ✓ Saved  │ English(US)│   🔔    │  │
│  └────────┴────────────────┴────────────────────────┴──────────┴────────────┴─────────┘  │
│                           ▲ inline render (no child component)                            │
│                                                                                           │
│  ┌────────────────────────────────┐                                                       │
│  │         MenuPopover            │  (when isMenuOpen = true)                             │
│  │  ┌──────────────────────────┐  │                                                       │
│  │  │  ✨ Points    750 / 1000 │  │                                                       │
│  │  │  [████████████░░░░░░░░]  │  │                                                       │
│  │  ├──────────────────────────┤  │                                                       │
│  │  │      ← Home              │  │                                                       │
│  │  ├──────────────────────────┤  │                                                       │
│  │  │   ⚙️ Editor Mode: Book    │  │  (display only, no submenu)                           │
│  │  └──────────────────────────┘  │                                                       │
│  └────────────────────────────────┘                                                       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                                     EditorHeader                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Props: bookTitle, currentStep, currentLanguage, saveStatus,                    │  │
│  │         notificationCount, userPoints, editorMode                               │  │
│  │  LocalState: isMenuOpen, isEditingTitle, editTitleValue                         │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│           │                │                  │                │                      │
│           ▼                ▼                  ▼                ▼                      │
│    ┌───────────┐    ┌───────────┐      ┌───────────┐    ┌─────────────────────────┐  │
│    │ MenuBtn   │    │[BookTitle]│      │   Step    │    │     Actions Group       │  │
│    │           │    │ (inline)  │      │ Breadcrumb│    │ ┌─────────┬───────────┐ │  │
│    │ onClick:  │    │           │      │           │    │ │SaveStat │ LangSelect│ │  │
│    │ toggle    │    │ Render:   │      │ Props:    │    │ ├─────────┼───────────┤ │  │
│    │ isMenuOpen│    │ IF editing│      │ •current  │    │ │NotifBtn │           │ │  │
│    │           │    │  → input  │      │  Step     │    │ └─────────┴───────────┘ │  │
│    │           │    │ ELSE      │      │           │    │                         │  │
│    │           │    │  → text   │      │ Callback: │    │ Callbacks: onSave,      │  │
│    │           │    │           │      │ •onChange │    │ onLangChange, onNotif   │  │
│    └───────────┘    └───────────┘      └───────────┘    └─────────────────────────┘  │
│           │                                                                           │
│           ▼                                                                           │
│    ┌───────────┐                                                                      │
│    │MenuPopover│                                                                      │
│    │ Props:    │                                                                      │
│    │ •isOpen   │                                                                      │
│    │ •points   │                                                                      │
│    │ •editorMd │                                                                      │
│    │           │                                                                      │
│    │ Callbacks:│                                                                      │
│    │ •onClose  │                                                                      │
│    │ •onHome   │                                                                      │
│    └───────────┘                                                                      │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Step Breadcrumb Visual States

**3 trạng thái:**

| State | Visual | Behavior | Description |
|-------|--------|----------|-------------|
| `active` | Solid bg, highlight | Không click | Step hiện tại đang làm việc |
| `completed` | Normal text, có check | Click → quay lại | Step đã hoàn thành, có thể quay lại |
| `inactive` | Dim/muted, opacity 50% | Click → đi tiếp | Step chưa mở khóa |

```
Step: idea (active) - Step đầu tiên
┌───────────────────────────────────────────────────────────────┐
│  [Idea]  >  Sketch  >  Illustration  >  Retouch               │
│   ▲ active   dim         dim             dim                  │
└───────────────────────────────────────────────────────────────┘

Step: sketch (active) - Có thể quay lại idea
┌───────────────────────────────────────────────────────────────┐
│  ✓ Idea  >  [Sketch]  >  Illustration  >  Retouch             │
│  ▲ completed   active       dim            dim                │
└───────────────────────────────────────────────────────────────┘

Step: illustration (active) - Có thể quay lại idea hoặc sketch
┌───────────────────────────────────────────────────────────────┐
│  ✓ Idea  >  ✓ Sketch  >  [Illustration]  >  Retouch           │
│  completed   completed      ▲ active          dim             │
└───────────────────────────────────────────────────────────────┘

Step: retouch (active) - Có thể quay lại tất cả steps trước
┌───────────────────────────────────────────────────────────────┐
│  ✓ Idea  >  ✓ Sketch  >  ✓ Illustration  >  [Retouch]         │
│  completed   completed      completed        ▲ active         │
└───────────────────────────────────────────────────────────────┘
```

**Visual Styling:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  State        │ Background  │ Text Color  │ Icon   │ Cursor        │
├─────────────────────────────────────────────────────────────────────┤
│  active       │ primary     │ white       │ none   │ default       │
│  completed    │ transparent │ primary     │ ✓      │ pointer       │
│  inactive     │ transparent │ muted (50%) │ none   │ not-allowed   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Designs

### 2.1 EditorHeader (Root Component)

**Mục đích:** Top navigation bar. Hiển thị book info, step navigation, language selector, và quick actions (save, notifications). Chứa Menu popover hiển thị points, home link, và editor mode (display only).

**Shared Types:**

```typescript
type Step = 'idea' | 'sketch' | 'illustration' | 'retouch';
type EditorMode = 'book' | 'asset';  // Maps to books.type: 1=book, 0=asset
type SaveStatus = 'unsaved' | 'saving' | 'saved';

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
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;              // Display only in menu
  onLanguageChange: (language: Language) => void;
  onTitleEdit: (newTitle: string) => void;
  onStepChange: (step: Step) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
}

interface EditorHeaderState {
  isMenuOpen: boolean;
  isEditingTitle: boolean;
  editTitleValue: string;  // local value khi đang edit
}
```

**Render Logic (pseudo):**

```
EditorHeader:
  RENDER header container (h-14, border-b, flex items-center justify-between)

    // Left section
    RENDER div.flex.items-center.gap-3
      RENDER MenuButton với onClick → toggle isMenuOpen

      // BookTitle (inline, không tách component)
      IF isEditingTitle:
        RENDER input với:
          - value: editTitleValue
          - onChange: setEditTitleValue
          - onKeyDown: Enter → onTitleEdit(editTitleValue), Escape → cancel
          - onBlur: onTitleEdit(editTitleValue)
          - autoFocus
      ELSE:
        RENDER span với:
          - text: bookTitle (truncate max ~200px)
          - onClick: setIsEditingTitle(true), setEditTitleValue(bookTitle)
          - cursor: text

    // Center section
    RENDER StepBreadcrumb với currentStep, onStepChange

    // Right section
    RENDER div.flex.items-center.gap-2
      RENDER SaveStatus với saveStatus
      RENDER LanguageSelector với currentLanguage, onLanguageChange
      RENDER NotificationButton với notificationCount, onNotificationClick

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

**Visual:**

```
┌─────┐
│  ≡  │
└─────┘
```

---

### 2.3 StepBreadcrumb

**Mục đích:** Breadcrumb navigation giữa 4 steps. Click step trước hoặc sau để chuyển step.

**Interface:**

```typescript
type StepState = 'active' | 'completed' | 'inactive';

interface StepBreadcrumbProps {
  currentStep: Step;
  onStepChange: (step: Step) => void;
}
```

**Configuration:**

```typescript
const STEPS: { id: Step; label: string }[] = [
  { id: 'idea', label: 'Idea' },
  { id: 'sketch', label: 'Sketch' },
  { id: 'illustration', label: 'Illustration' },
  { id: 'retouch', label: 'Retouch' },
];

const STEP_ORDER: Record<Step, number> = {
  idea: 0,
  sketch: 1,
  illustration: 2,
  retouch: 3,
};

// Xác định state của mỗi step
function getStepState(step: Step, currentStep: Step): StepState {
  const stepIndex = STEP_ORDER[step];
  const currentIndex = STEP_ORDER[currentStep];

  if (stepIndex === currentIndex) return 'active';
  if (stepIndex < currentIndex) return 'completed';
  return 'inactive';
}
```

**Render Logic (pseudo):**

```
StepBreadcrumb:
  FOR step IN STEPS:
    stepState = getStepState(step.id, currentStep)

    RENDER step item với:
      IF stepState === 'active':
        - cursor: default
        - không có onClick
      ELSE IF stepState === 'completed':
        - text: primary color
        - cursor: pointer
        - onClick: onStepChange(step.id)
      ELSE (inactive):
        - text: muted (opacity 50%)
        - cursor: pointer
        - onClick: onStepChange(step.id)

    IF không phải step cuối:
      RENDER separator ">"
```

---

### 2.4 SaveStatus

**Mục đích:** Indicator hiển thị trạng thái save.

**Interface:**

```typescript
interface SaveStatusProps {
  saveStatus: SaveStatus;
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

### 2.5 NotificationButton

**Mục đích:** Bell icon với badge count, mở notification panel.

**Interface:**

```typescript
interface NotificationButtonProps {
  count: number;
  onClick: () => void;
}
```

**Visual:**

```
No notifications:     Has notifications:
┌─────┐               ┌─────┐
│ 🔔  │               │ 🔔³ │
└─────┘               └─────┘
                         ▲ badge (max "9+")
```

---

### 2.6 LanguageSelector

**Mục đích:** Dropdown chọn ngôn ngữ hiển thị trong editor. Đặt trực tiếp trên header để dễ truy cập.

**Interface:**

```typescript
interface LanguageSelectorProps {
  currentLanguage: Language;
  onLanguageChange: (language: Language) => void;
}

interface LanguageSelectorState {
  isOpen: boolean;
}

const AVAILABLE_LANGUAGES: Language[] = [
  { name: 'English (US)', code: 'en_US' },
  { name: 'Tiếng Việt', code: 'vi_VN' },
  { name: '日本語', code: 'ja_JP' },
  { name: '한국어', code: 'ko_KR' },
  { name: '中文 (简体)', code: 'zh_CN' },
];
```

**Visual:**

```
Closed:                       Open:
┌─────────────────────┐       ┌─────────────────────┐
│ 🌐 English (US)  ▾  │       │ 🌐 English (US)  ▴  │
└─────────────────────┘       └─────────────────────┘
                              ┌─────────────────────┐
                              │ ✓ English (US)      │
                              │   Tiếng Việt        │
                              │   日本語             │
                              │   한국어             │
                              │   中文 (简体)        │
                              └─────────────────────┘
```

---

### 2.7 MenuPopover

**Mục đích:** Popover menu chứa points, navigation home, và editor mode display (không thay đổi được).

**Interface:**

```typescript
interface MenuPopoverProps {
  isOpen: boolean;
  userPoints: UserPoints;
  editorMode: EditorMode;
  onClose: () => void;
  onNavigateHome: () => void;
}
```

**Menu Items Configuration:**

```typescript
interface MenuItem {
  id: string;
  icon: string;
  label: string;
  type: 'action' | 'display';
  value?: string;               // For display items
}

const MENU_ITEMS: MenuItem[] = [
  { id: 'home', icon: 'ArrowLeft', label: 'Home', type: 'action' },
  { id: 'editor_mode', icon: 'Layers', label: 'Editor Mode', type: 'display' },
];

const EDITOR_MODE_LABELS: Record<EditorMode, string> = {
  book: 'Book',
  asset: 'Asset',
};
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
      ELSE IF item.type === 'display':
        RENDER DisplayItem với icon, label, value (e.g., "Editor Mode: Book")
```

---

### 2.8 PointsDisplay

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
`isMenuOpen` là local state của EditorHeader. Không cần lift lên EditorPage vì menu chỉ ảnh hưởng UI trong phạm vi EditorHeader.

**Language Selector on Header**
Language selector đặt trực tiếp trên header (không trong menu) vì là action thường xuyên sử dụng khi edit multi-language content. Giảm số click cần thiết.

**Editor Mode Display Only**
Editor mode được hiển thị trong menu nhưng không thay đổi được từ UI. Mode được xác định bởi permissions hoặc share link context.

**Points from User Context**
`userPoints` lấy từ user session/context, không phải từ book data. EditorPage truyền xuống như prop.

**Click Outside to Close**
MenuPopover và LanguageSelector đóng khi click outside. Sử dụng portal để render popover ở root level, tránh z-index issues.

### 3.2 Accessibility

- MenuButton: `aria-expanded`, `aria-haspopup="menu"`
- MenuPopover: `role="menu"`, keyboard navigation (↑↓ arrows, Enter, Escape)
- StepBreadcrumb: `aria-current="step"` cho active step
- All interactive elements: visible focus states

### 3.3 Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| Desktop (>1024px) | Full layout như design |
| Tablet (768-1024px) | Truncate BookTitle, hide step labels (chỉ hiện icons), collapse LanguageSelector to icon only |
| Mobile (<768px) | Hide StepBreadcrumb (show in Menu), LanguageSelector collapse to globe icon |

### 3.4 Animation

- MenuPopover: `animate-in fade-in-0 zoom-in-95` (150ms)
- LanguageSelector dropdown: `animate-in fade-in-0 zoom-in-95` (150ms)
- SaveStatus transition: `transition-colors duration-200`
