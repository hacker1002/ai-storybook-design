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
| `inactive` | Dim/muted, opacity 50% | Click → skip ahead | Step chưa đến, user có thể skip ahead |

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
│  inactive     │ transparent │ muted (50%) │ none   │ pointer       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

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

### 2.2 Interface

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

### 2.3 Render Logic (pseudo)

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
    RENDER MenuPopover với props, callbacks
```

### 2.4 Visual

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────┐  ┌────────────────┐  ┌────────────────────────┐  ┌──────┐ ┌────────┐ ┌────┐ │
│  │   ≡    │  │ The Hidden...  │  │ [I] > S > I > R        │  │✓ Saved│ │ 🌐 EN ▾ │ │ 🔔 │ │
│  └────────┘  └────────────────┘  └────────────────────────┘  └──────┘ └────────┘ └────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 MenuButton

📄 **Doc:** *(inline, không cần file riêng)*

**Mục đích:** Hamburger icon button mở Menu popover.

**Props & Callbacks:**

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

### 3.2 StepBreadcrumb

📄 **Doc:** [`01-01-step-breadcrumb.md`](./01-01-step-breadcrumb.md)

**Mục đích:** Breadcrumb navigation giữa 4 steps. Click step trước hoặc sau để chuyển step.

**Props & Callbacks:**

```typescript
interface StepBreadcrumbProps {
  currentStep: Step;
  onStepChange: (step: Step) => void;
}
```

---

### 3.3 SaveStatus

📄 **Doc:** *(inline, không cần file riêng)*

**Mục đích:** Indicator hiển thị trạng thái save.

**Props & Callbacks:**

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

### 3.4 NotificationButton

📄 **Doc:** *(inline, không cần file riêng)*

**Mục đích:** Bell icon với badge count, mở notification panel.

**Props & Callbacks:**

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

### 3.5 LanguageSelector

📄 **Doc:** [`01-02-language-selector.md`](./01-02-language-selector.md)

**Mục đích:** Dropdown chọn ngôn ngữ hiển thị trong editor. Đặt trực tiếp trên header để dễ truy cập.

**Props & Callbacks:**

```typescript
interface LanguageSelectorProps {
  currentLanguage: Language;
  onLanguageChange: (language: Language) => void;
}
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

### 3.6 MenuPopover

📄 **Doc:** [`01-03-menu-popover.md`](./01-03-menu-popover.md)

**Mục đích:** Popover menu chứa points, navigation home, và editor mode display.

**Props & Callbacks:**

```typescript
interface MenuPopoverProps {
  isOpen: boolean;
  userPoints: UserPoints;
  editorMode: EditorMode;
  onClose: () => void;
  onNavigateHome: () => void;
}
```

**Visual:**

```
┌─────────────────────────────────┐
│  ✨ Points        750 / 1000    │
│  [████████████░░░░░░░░]  75%    │
├─────────────────────────────────┤
│      ← Home                     │
├─────────────────────────────────┤
│   ⚙️ Editor Mode: Book           │
└─────────────────────────────────┘
```

---

### 3.7 PointsDisplay

📄 **Doc:** *(inline trong MenuPopover)*

**Mục đích:** Hiển thị user points với progress bar trong MenuPopover.

**Props & Callbacks:**

```typescript
interface PointsDisplayProps {
  current: number;
  total: number;
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

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

### 4.2 Accessibility

- MenuButton: `aria-expanded`, `aria-haspopup="menu"`
- MenuPopover: `role="menu"`, keyboard navigation (↑↓ arrows, Enter, Escape)
- StepBreadcrumb: `aria-current="step"` cho active step
- All interactive elements: visible focus states

### 4.3 Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| Desktop (>1024px) | Full layout như design |
| Tablet (768-1024px) | Truncate BookTitle, hide step labels (chỉ hiện icons), collapse LanguageSelector to icon only |
| Mobile (<768px) | Hide StepBreadcrumb (show in Menu), LanguageSelector collapse to globe icon |

### 4.4 Animation

- MenuPopover: `animate-in fade-in-0 zoom-in-95` (150ms)
- LanguageSelector dropdown: `animate-in fade-in-0 zoom-in-95` (150ms)
- SaveStatus transition: `transition-colors duration-200`
