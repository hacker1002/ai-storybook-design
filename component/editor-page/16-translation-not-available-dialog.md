# TranslationNotAvailableDialog: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     TranslationNotAvailableDialog                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                         DialogOverlay                               │  │
│  │  ┌───────────────────────────────────────────────────────────────┐  │  │
│  │  │                      DialogContent                            │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  DialogHeader                                           │  │  │  │
│  │  │  │  ┌───────────────────────────────────────┐ ┌─────────┐  │  │  │  │
│  │  │  │  │ Title: "Translation Not Available"    │ │    X    │  │  │  │  │
│  │  │  │  └───────────────────────────────────────┘ └─────────┘  │  │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘  │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  DialogBody                                             │  │  │  │
│  │  │  │  "The translation for **{language}** is not available   │  │  │  │
│  │  │  │   yet. Would you like to translate your content to      │  │  │  │
│  │  │  │   this language?"                                       │  │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘  │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  DialogFooter                                           │  │  │  │
│  │  │  │  ┌─────────────┐  ┌─────────────────────────────────┐   │  │  │  │
│  │  │  │  │   Cancel    │  │  ✨ Translate                   │   │  │  │  │
│  │  │  │  │  (outline)  │  │     (primary)                   │   │  │  │  │
│  │  │  │  └─────────────┘  └─────────────────────────────────┘   │  │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              EditorPage                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  State: translationDialogState: { isOpen, targetLanguage } | null    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                     │
│    handleLanguageChange(lang)        │                                     │
│           │                          │                                     │
│           ▼                          │                                     │
│    setCurrentLanguage(lang)          │                                     │
│           │                          │                                     │
│           ▼                          │                                     │
│    checkTranslationAvailable()       │                                     │
│           │                          │                                     │
│     ┌─────┴─────┐                    │                                     │
│  available    missing                │                                     │
│     │            │                   │                                     │
│     ▼            ▼                   ▼                                     │
│   (done)    setTranslationDialogState({ isOpen: true, targetLanguage })    │
│                                      │                                     │
│                                      ▼                                     │
│                    ┌──────────────────────────────────────────┐            │
│                    │   TranslationNotAvailableDialog          │            │
│                    │                                          │            │
│                    │   onCancel → setTranslationDialogState(null)          │
│                    │   onTranslate → call API, then close     │            │
│                    └──────────────────────────────────────────┘            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Visual Reference

📸 **Screenshot:** [editor-translate-popup.png](screenshots/editor-translate-popup.png)

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Dialog xác nhận khi user chọn language mà chưa có translation. Hỏi user có muốn translate content sang language đó không.

**Trigger:** Khi user chọn language trong LanguageSelector và `spreads[].textboxes[]` chưa có key `[language_code]`.

### 2.2 Interface

```typescript
interface TranslationNotAvailableDialogProps {
  isOpen: boolean;
  targetLanguage: Language;
  sourceLanguage: Language;  // Current original/source language
  onCancel: () => void;
  onTranslate: () => void;  // Trigger translation API call
}
```

### 2.3 Translation Check Logic (EditorPage)

```
handleLanguageChange(language):
  1. Always update currentLanguage (allow viewing empty state)
  2. Check if any textbox in spreads has key [language.code]
  3. If no translation found AND not original language:
     → Show TranslationNotAvailableDialog with targetLanguage, sourceLanguage
```

### 2.4 Render Logic (pseudo)

```
TranslationNotAvailableDialog:
  IF NOT isOpen:
    RETURN null

  RENDER Dialog với overlay
    RENDER DialogHeader
      RENDER title: "Translation Not Available"
      RENDER close button (X) → onCancel

    RENDER DialogBody
      RENDER text với targetLanguage.name in bold:
        "The translation for **{targetLanguage.name}** is not available yet.
         Would you like to translate your content to this language?"

    RENDER DialogFooter
      RENDER Button variant="outline" onClick=onCancel: "Cancel"
      RENDER Button variant="primary" onClick=onTranslate:
        Icon sparkle + "Translate"
```

### 2.5 Visual

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

## 3. EditorPage State Update

### 3.1 New State

```typescript
interface EditorPageState {
  // ... existing state ...

  // NEW: Translation dialog state
  translationDialogState: {
    isOpen: boolean;
    targetLanguage: Language;
    sourceLanguage: Language;
  } | null;
}
```

### 3.2 New Callbacks

```typescript
interface EditorPageCallbacks {
  // ... existing callbacks ...

  // NEW: Handle translation request
  onTranslateContent: (
    sourceLanguage: Language,
    targetLanguage: Language
  ) => Promise<void>;
}
```

---

## 4. Translation API (định nghĩa sau)

> **Note:** API endpoint cho translation sẽ được define riêng.
> Dialog chỉ gọi `onTranslate` callback, EditorPage xử lý API call.

```typescript
// Placeholder cho API call trong EditorPage
async function handleTranslateContent(
  sourceLanguage: Language,
  targetLanguage: Language
): Promise<void> {
  // 1. Show loading state
  // 2. Call translation API (TBD)
  // 3. Update spreads với translated textboxes
  // 4. Close dialog
  // 5. Show success toast
}
```

---

## 5. Technical Notes

### 5.1 Key Design Decisions

**Logic in EditorPage, Not LanguageSelector**
- EditorPage owns `spreads` data needed for translation check
- LanguageSelector stays simple (just emits selection)
- Single source of truth for translation state

**Always Update currentLanguage**
- User can view empty textboxes in new language even without translation
- Translation is optional enhancement, not blocking

**Check at Spread Level**
- Only check `spreads[].textboxes[]`, not `manuscript.dummies[]`
- Manuscript has separate finalization workflow

**Dialog vs Toast**
- Use Dialog (not toast) because translation is significant action
- User should consciously decide before triggering potentially expensive API call

### 5.2 Edge Cases

| Case | Behavior |
|------|----------|
| Select original language | No dialog (always has content) |
| All spreads empty (no textboxes) | No dialog (nothing to translate) |
| Some textboxes have translation, some don't | No dialog (partial OK) |
| User cancels | Just close dialog, language already changed |
| Translation API fails | Show error toast, dialog already closed |

### 5.3 Animation

- Dialog: `animate-in fade-in-0 zoom-in-95` (150ms)
- Overlay: `animate-in fade-in-0` (150ms)
- Button loading state: spinner icon replacing sparkle

### 5.4 Accessibility

- Dialog: `role="alertdialog"`, `aria-modal="true"`
- Focus trap within dialog
- Close on Escape key
- `aria-describedby` for body text
