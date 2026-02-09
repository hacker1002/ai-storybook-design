# EditorHeader: Component Design

📸 **Screenshot:** [screenshots/header.png](screenshots/header.png)

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                      EditorHeader                                        │
│  ┌────────┬────────────────┬─────────────────────────────┬──────────┬──────────┬───────┐ │
│  │MenuBtn │   [BookTitle]  │     [StepBreadcrumb]        │SaveStatus│LangSelect│NotifBtn││
│  │   ≡    │The Hidden Val..│ (Manuscript) > Illus > Ret  │ ✓ Saved  │🌐 EN (US)│  🔔   │ │
│  └────────┴────────────────┴─────────────────────────────┴──────────┴──────────┴───────┘ │
│                           ▲ inline render (no child component)                           │
│                                                                                          │
│  ┌────────────────────────────────┐                                                      │
│  │         MenuPopover            │  (when isMenuOpen = true)                            │
│  │  ┌──────────────────────────┐  │                                                      │
│  │  │  ✨ Points    750 / 1000 │  │                                                      │
│  │  │  [████████████░░░░░░░░]  │  │                                                      │
│  │  ├──────────────────────────┤  │                                                      │
│  │  │      ← Home              │  │                                                      │
│  │  ├──────────────────────────┤  │                                                      │
│  │  │   ⚙️ Editor Mode: Book    │  │  (display only, no submenu)                          │
│  │  └──────────────────────────┘  │                                                      │
│  └────────────────────────────────┘                                                      │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                      EditorHeader                                       │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  Props: bookTitle, saveStatus, notificationCount, userPoints, editorMode           │ │
│  │  LocalState: isMenuOpen, isEditingTitle, editTitleValue                            │ │
│  └────────────────────────────────────────────────────────────────────────────────────┘ │
│           │                │                  │                │                        │
│           ▼                ▼                  ▼                ▼                        │
│    ┌───────────┐    ┌───────────┐      ┌───────────────┐  ┌──────────────────────────┐  │
│    │ MenuBtn   │    │[BookTitle]│      │[Breadcrumb]   │  │     Actions Group        │  │
│    │           │    │ (inline)  │      │ (inline)      │  │ ┌─────────┬───────────┐  │  │
│    │ onClick:  │    │           │      │               │  │ │SaveStat │ LangSelect│  │  │
│    │ toggle    │    │ Render:   │      │ ⚡ Uses        │  │ ├─────────┼───────────┤  │  │
│    │ isMenuOpen│    │ IF editing│      │ EditorSetting │  │ │NotifBtn │           │  │  │
│    │           │    │  → input  │      │ Store direct  │  │ └─────────┴───────────┘  │  │
│    │           │    │ ELSE      │      │               │  │                          │  │
│    │           │    │  → text   │      │               │  │ ⚡ LangSelect uses        │  │
│    │           │    │           │      │               │  │ EditorSettingStore direct│  │
│    └───────────┘    └───────────┘      └───────────────┘  └──────────────────────────┘  │
│           │                                                                             │
│           ▼                                                                             │
│    ┌────────────┐                                                                       │
│    │MenuPopover │                                                                       │
│    │ Props:     │                                                                       │
│    │ •isOpen    │                                                                       │
│    │ •points    │                                                                       │
│    │ •editorMd  │                                                                       │
│    │ Callbacks: │                                                                       │
│    │ •onClose   │                                                                       │
│    │ •onHome    │                                                                       │
│    └────────────┘                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────┐
│  EditorSettingsStore   │
│  ┌──────────────────┐  │
│  │ currentLanguage  │──────► LanguageSelector (direct)
│  │ currentStep      │──────► StepBreadcrumb inline (direct)
│  └──────────────────┘  │
└────────────────────────┘
```

### 1.3 Step Breadcrumb Visual States (Inline)

> **Note:** Breadcrumb render inline trong EditorHeader, không tách child component.

**3 Steps:** `manuscript` → `illustration` → `retouch`

**2 trạng thái:**

| State | Visual | Behavior |
|-------|--------|----------|
| `active` | Pill/chip background với border | Step hiện tại |
| `default` | Plain text, muted color | Click → navigate |

```
Step: manuscript (active)
┌───────────────────────────────────────────────────────────────┐
│  (Manuscript)  >  Illustration  >  Retouch                    │
│   ▲ active         default         default                    │
└───────────────────────────────────────────────────────────────┘

Step: illustration (active)
┌───────────────────────────────────────────────────────────────┐
│  Manuscript  >  (Illustration)  >  Retouch                    │
│   default        ▲ active          default                    │
└───────────────────────────────────────────────────────────────┘

Step: retouch (active)
┌───────────────────────────────────────────────────────────────┐
│  Manuscript  >  Illustration  >  (Retouch)                    │
│   default        default         ▲ active                     │
└───────────────────────────────────────────────────────────────┘
```

**Visual Styling:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  State        │ Background       │ Text Color   │ Cursor            │
├─────────────────────────────────────────────────────────────────────┤
│  active       │ pill w/ border   │ primary      │ default           │
│  default      │ transparent      │ muted        │ pointer           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Top navigation bar. Hiển thị book info, step navigation, language selector, và quick actions (save, notifications). Chứa Menu popover hiển thị points, home link, và editor mode (display only).

**Shared Types:**

```typescript
type PipelineStep = 'manuscript' | 'illustration' | 'retouch';
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

const PIPELINE_STEPS: { key: PipelineStep; label: string }[] = [
  { key: 'manuscript', label: 'Manuscript' },
  { key: 'illustration', label: 'Illustration' },
  { key: 'retouch', label: 'Retouch' },
];
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface EditorHeaderProps {
  bookTitle: string;
  saveStatus: SaveStatus;
  notificationCount: number;
  userPoints: UserPoints;
  editorMode: EditorMode;              // Display only in menu

  onTitleEdit: (newTitle: string) => void;
  onSave: () => Promise<void>;
  onNotificationClick: () => void;
  onNavigateHome: () => void;
  onLanguageChange: (newLang: Language, prevLang: Language) => void;
  // ⚡ Callback để EditorPage handle Translation Check Flow
}

interface EditorHeaderState {
  isMenuOpen: boolean;
  isEditingTitle: boolean;
  editTitleValue: string;  // local value khi đang edit
}
```

**Store Integration:**

```typescript
// EditorHeader passes onLanguageChange to LanguageSelector
// LanguageSelector: setCurrentLanguage() → onLanguageChange()

// StepBreadcrumb inline: useCurrentStep(), setCurrentStep()
```

**Lưu ý:** Language/Step state managed by EditorSettingsStore. LanguageSelector cần callback `onLanguageChange` để notify EditorPage trigger Translation Check Flow.

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

    // Center section - StepBreadcrumb (inline, consumes store directly)
    RENDER div.flex.items-center.gap-2
      FOR each step in PIPELINE_STEPS:
        IF step.key === currentStep:
          RENDER span.pill với step.label (active style)
        ELSE:
          RENDER button với:
            - text: step.label
            - onClick: setCurrentStep(step.key)
        IF not last step:
          RENDER ChevronRight icon (separator)

    // Right section
    RENDER div.flex.items-center.gap-2
      RENDER SaveStatus với saveStatus, onSave
      RENDER LanguageSelector với onLanguageChange  // store + callback
      RENDER NotificationButton với notificationCount, onNotificationClick

  IF isMenuOpen:
    RENDER MenuPopover với props, callbacks
```

### 2.4 Visual

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│  ┌────┐  ┌─────────────────┐  ┌──────────────────────────────────┐  ┌───────┐ ┌───────┐ ┌───┐ │
│  │ ≡  │  │ The Hidden Val..│  │ (Manuscript) > Illus > Retouch   │  │✓ Saved│ │🌐 EN ▾│ │🔔 │ │
│  └────┘  └─────────────────┘  └──────────────────────────────────┘  └───────┘ └───────┘ └───┘ │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý quan trọng:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - **KHÔNG** thiết kế store integration tại đây — child component tự thiết kế store selectors/actions trong file riêng của nó
> - State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó

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

### 3.2 StepBreadcrumb (Inline - Removed as Child)

📄 **Doc:** *(inline, không cần file riêng)*

> Render inline trong EditorHeader vì logic đơn giản (chỉ map steps + highlight active).

**Mục đích:** Hiển thị step pipeline breadcrumb.

**Store Integration (inline trong EditorHeader):**
- `useCurrentStep()` - read current step
- `useEditorSettingsActions().setCurrentStep()` - update step

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

### 3.5 LanguageSelector (Inline Design)

📄 **Doc:** *(inline, không tách file riêng)*

**Mục đích:** Dropdown chọn ngôn ngữ hiển thị trong editor. Đặt trực tiếp trên header để dễ truy cập. Khi đổi ngôn ngữ, cần notify parent để trigger Translation Check Flow.

**Props & Callbacks:**

```typescript
interface LanguageSelectorProps {
  onLanguageChange: (newLang: Language, prevLang: Language) => void;
  // Callback ra EditorPage để trigger Translation Check Flow
  // EditorPage sẽ check xem content đã được translate chưa
}

interface LanguageSelectorState {
  isOpen: boolean;
}
```

**Constants:**

```typescript
const AVAILABLE_LANGUAGES: Language[] = [
  { code: 'en_US', name: 'English (US)' },
  { code: 'vi_VN', name: 'Tiếng Việt' },
  { code: 'ja_JP', name: '日本語' },
  { code: 'ko_KR', name: '한국어' },
  { code: 'zh_CN', name: '中文 (简体)' },
];
```

**Store Integration:**
- `useCurrentLanguage()` - read current language
- `useEditorSettingsActions().setCurrentLanguage()` - update store

**Render Logic (pseudo):**
```
LanguageSelector:
  currentLang = useCurrentLanguage()

  RENDER button.trigger
    - Globe icon + currentLang.name + ChevronDown
    - onClick: toggle isOpen

  IF isOpen:
    RENDER dropdown (portal, positioned below trigger)
      FOR each lang in AVAILABLE_LANGUAGES:
        RENDER option với:
          - Checkmark if lang.code === currentLang.code
          - lang.name
          - onClick:
              prevLang = currentLang
              setCurrentLanguage(lang)
              onLanguageChange(lang, prevLang)
              setIsOpen(false)
```

**Flow khi select language:**
```
User click language option
  → setCurrentLanguage(newLang) // update store
  → onLanguageChange(newLang, prevLang) // notify parent
    → EditorPage handles Translation Check Flow
```

**Translation Check Flow (handled by EditorPage):**
- Check if content exists for new language
- If not: prompt user to translate or show empty state
- If yes: load content for selected language

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

**Notes:**
- Languages từ static constant `AVAILABLE_LANGUAGES`
- Click outside → close dropdown
- Keyboard: Escape → close, Enter → select, ↑↓ → navigate

---

### 3.6 MenuPopover (Inline Design)

📄 **Doc:** *(inline, không tách file riêng)*

**Mục đích:** Popover menu chứa user points display, navigation home, và editor mode info (display only).

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

**Render Logic (pseudo):**
```
MenuPopover:
  IF NOT isOpen: return null

  RENDER Popover (portal, positioned below MenuButton)
    // Points Section
    RENDER div.points-section
      RENDER span "✨ Points"
      RENDER span "{current} / {total}"
      RENDER ProgressBar với value={current/total * 100}

    RENDER Divider

    // Home Navigation
    RENDER button.menu-item
      - ← icon + "Home"
      - onClick: onNavigateHome(), onClose()

    RENDER Divider

    // Editor Mode (display only)
    RENDER div.menu-item.disabled
      - ⚙️ icon + "Editor Mode: {editorMode}"
      - No onClick (display only)
```

**Visual:**

```
┌─────────────────────────────────┐
│  ✨ Points        750 / 1000    │
│  [████████████░░░░░░░░]  75%    │
├─────────────────────────────────┤
│  ← Home                         │
├─────────────────────────────────┤
│  ⚙️ Editor Mode: Book            │
└─────────────────────────────────┘
     ▲
     │ anchor to MenuButton
```

**Behavior:**
- Click outside → onClose()
- Click Home → navigate + close
- Editor Mode: read-only, shows current mode (Book/Asset)
- Escape key → onClose()

**PointsDisplay (inline trong MenuPopover):**

```typescript
// Không tách component, render trực tiếp trong MenuPopover
// Progress bar: width = (current / total) * 100%
// Color: primary khi < 80%, warning khi >= 80%, danger khi >= 95%
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Store-based Language/Step**
`currentLanguage` và `currentStep` được quản lý bởi EditorSettingsStore. LanguageSelector consume store trực tiếp. StepBreadcrumb render inline trong EditorHeader (không tách component riêng vì logic đơn giản).

**3-Step Pipeline**
Pipeline gồm 3 steps: `manuscript` → `illustration` → `retouch`. Không có completed/inactive states phức tạp - chỉ có active (pill style) và default (plain text, clickable).

**Menu State is Local**
`isMenuOpen` là local state của EditorHeader. Không cần lift lên EditorPage vì menu chỉ ảnh hưởng UI trong phạm vi EditorHeader.

**Language Selector on Header**
Language selector đặt trực tiếp trên header (không trong menu) vì là action thường xuyên sử dụng khi edit multi-language content. Giảm số click cần thiết.

**Language Change + Translation Check Flow**
Khi user đổi ngôn ngữ:
1. LanguageSelector gọi `setCurrentLanguage()` để update store
2. Sau đó callback `onLanguageChange(newLang, prevLang)` ra EditorPage
3. EditorPage handle Translation Check Flow:
   - Check content tồn tại cho ngôn ngữ mới không
   - Nếu chưa: prompt translate hoặc show empty state
   - Nếu có: load content cho ngôn ngữ đã chọn

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
| Tablet (768-1024px) | Truncate BookTitle, collapse LanguageSelector to icon only |
| Mobile (<768px) | Hide breadcrumb (show in Menu), LanguageSelector collapse to globe icon |

### 4.4 Animation

- MenuPopover: `animate-in fade-in-0 zoom-in-95` (150ms)
- LanguageSelector dropdown: `animate-in fade-in-0 zoom-in-95` (150ms)
- SaveStatus transition: `transition-colors duration-200`

---
