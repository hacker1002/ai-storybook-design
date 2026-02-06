# ManuscriptFinalizationView: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ManuscriptFinalizationView                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  GridHeader                                                        │  │
│  │  [─]  4 / row  [+]                               [🌐 Translate]   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  SpreadGrid (reused from ManuscriptDummyView)                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│  │
│  │  │ Spread      │  │ Spread      │  │ Spread      │  │ Spread      ││  │
│  │  │ Thumbnail   │  │ Thumbnail   │  │ Thumbnail   │  │ Thumbnail   ││  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘│  │
│  │  ┌─────────────┐  ┌─────────────────┐                              │  │
│  │  │ Spread      │  │       +         │  ← NewSpreadButton           │  │
│  │  └─────────────┘  └─────────────────┘                              │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  TranslateDialog (conditional)                                     │  │
│  │  "Select target language for translation"                          │  │
│  │  [Language Dropdown] [Cancel] [Translate]                          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    ManuscriptFinalizationView                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Props: dummy, currentLanguage, availableLanguages         │  │
│  │  State: columnsPerRow, selectedSpreadIndex,                │  │
│  │         isTranslateDialogOpen, selectedTargetLanguage      │  │
│  │  Callbacks: onSpreadSelect, onSpreadAdd, onSpreadUpdate,   │  │
│  │             onTranslate                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│         │                    │                     │             │
│         ▼                    ▼                     ▼             │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐      │
│  │ GridHeader  │      │ SpreadGrid  │      │ Translate   │      │
│  │ + Translate │      │ (same as    │      │ Dialog      │      │
│  │   Button    │      │  DummyView) │      │ (modal)     │      │
│  └─────────────┘      └─────────────┘      └─────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 Translate Flow

```
┌──────────────────┐
│ Click Translate  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Open Dialog      │
│ (select language)│
└────────┬─────────┘
         │
    Select Language
         │
         ▼
┌──────────────────┐
│ Confirm Dialog   │
│ "Translate all   │
│  textboxes to    │
│  {language}?"    │
└────────┬─────────┘
         │
      Confirm
         │
         ▼
┌──────────────────┐
│ onTranslate(     │
│  targetLanguage) │
│                  │
│ → API call       │
│ → Add new lang   │
│   entries to     │
│   textboxes      │
└──────────────────┘
```

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** View cho Finalization step. Hiển thị spread grid từ selected dummy source với thêm Translate functionality để dịch textboxes sang ngôn ngữ khác.

**Language impact:** ✅ **BỊ ẢNH HƯỞNG** — Textbox text hiển thị và translate theo `currentLanguage.code`

**Khác biệt với ManuscriptDummyView:**
- Có thêm "Translate" button ở header
- Có TranslateDialog để chọn target language
- Dùng cho finalization (output → snapshot.spreads[])

### 2.2 Interface

```typescript
interface ManuscriptFinalizationViewProps {
  dummy: ManuscriptDummy | null;
  currentLanguage: Language;
  availableLanguages: Language[];
  onSpreadSelect: (spreadIndex: number) => void;
  onSpreadAdd: () => void;
  onSpreadUpdate: (spreadIndex: number, spread: DummySpread) => void;
  onTranslate: (targetLanguage: Language) => Promise<void>;
}

interface ManuscriptFinalizationViewState {
  columnsPerRow: number;
  selectedSpreadIndex: number | null;
  isTranslateDialogOpen: boolean;
  selectedTargetLanguage: Language | null;
  isTranslating: boolean;
}
```

### 2.3 Render Logic (pseudo)

```
ManuscriptFinalizationView:
  IF dummy === null:
    RENDER EmptyState "No dummy selected for finalization"
    RETURN

  targetLanguageOptions = availableLanguages
    .filter(lang => lang.code !== currentLanguage.code)

  RENDER GridHeader:
    - columnsPerRow controls (same as DummyView)
    - TranslateButton:
        - onClick: () => setIsTranslateDialogOpen(true)
        - disabled: isTranslating

  RENDER SpreadGrid:
    - (same as ManuscriptDummyView)

  IF isTranslateDialogOpen:
    RENDER TranslateDialog:
      - targetLanguageOptions
      - selectedTargetLanguage
      - onLanguageSelect
      - onCancel: () => setIsTranslateDialogOpen(false)
      - onConfirm: handleTranslate()

  handleTranslate():
    setIsTranslating(true)
    await onTranslate(selectedTargetLanguage)
    setIsTranslating(false)
    setIsTranslateDialogOpen(false)
```

### 2.4 Visual

**Normal State:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  ─   4 / row   +                                    🌐 Translate    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ 0   │   1   │  │ 2   │   3   │  │ 4   │   5   │  │ 6   │   7   │ │
│  │     │       │  │     │       │  │     │       │  │     │       │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
│    Page 1-2         Page 3-4         Page 5-6         Page 7-8      │
│                                                                     │
│  ┌─────────────┐  ┌─────────────────┐                               │
│  │ 8   │   9   │  │       +         │                               │
│  └─────────────┘  └─────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

**Translate Dialog Open:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  ─   4 / row   +                                    🌐 Translate    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                                                                │ │
│  │     ┌─────────────────────────────────────────┐                │ │
│  │     │  🌐 Translate Content                   │                │ │
│  │     │                                         │                │ │
│  │     │  Select target language:                │                │ │
│  │     │  ┌─────────────────────────────────┐   │                │ │
│  │     │  │ Vietnamese (vi_VN)           ∨ │   │                │ │
│  │     │  └─────────────────────────────────┘   │                │ │
│  │     │                                         │                │ │
│  │     │  This will translate all textboxes     │                │ │
│  │     │  from English to Vietnamese.           │                │ │
│  │     │                                         │                │ │
│  │     │  ┌─────────┐  ┌───────────────────┐    │                │ │
│  │     │  │ Cancel  │  │  🌐 Translate     │    │                │ │
│  │     │  └─────────┘  └───────────────────┘    │                │ │
│  │     └─────────────────────────────────────────┘                │ │
│  │                                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               ▲
                         Modal overlay
```

**Translating State:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  ─   4 / row   +                                 ⏳ Translating...   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  (grid content dimmed/disabled during translation)                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** SpreadGrid và SpreadThumbnail được reuse từ ManuscriptDummyView.
> Chỉ có TranslateDialog là element mới (modal dialog với form elements).

### 3.1 Shared Components (reused)

| Component | Source | Notes |
|-----------|--------|-------|
| SpreadGrid | ManuscriptDummyView | Same grid layout |
| SpreadThumbnail | 03-03-01 | Same thumbnail rendering |
| NewSpreadButton | ManuscriptDummyView | Same add button |

### 3.2 TranslateDialog (inline modal, không tách component)

**Mục đích:** Modal dialog để chọn target language và confirm translation.

**Elements:**

| Element | Type | Notes |
|---------|------|-------|
| Dialog overlay | `<div>` | Semi-transparent backdrop |
| Dialog container | `<div>` | Centered modal box |
| Title | `<h3>` | "Translate Content" |
| Language select | `<select>` | Dropdown với available languages |
| Description | `<p>` | Explains what will happen |
| Cancel button | `<button>` | Secondary, closes dialog |
| Translate button | `<button>` | Primary, triggers translation |

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Reuse SpreadGrid from DummyView**
FinalizationView shares 90% logic với DummyView. Lý do: DRY principle, consistent UX.

**TranslateDialog as inline modal**
Không tách TranslateDialog thành component riêng vì chỉ dùng ở đây và logic đơn giản (select + confirm). Lý do: YAGNI.

**Filter Current Language**
Target language options không bao gồm currentLanguage. Lý do: Không có ý nghĩa translate sang cùng ngôn ngữ.

**Translation adds new entries**
Translation không replace existing text, mà adds new language entry vào textbox. Lý do: Preserve original, support multi-language.

### 4.2 Translate API Call

```typescript
// onTranslate callback triggers API:
// POST /api/translate-manuscript
// Body: {
//   bookId: string,
//   dummyType: DummyType,
//   sourceLanguage: string,
//   targetLanguage: string
// }
// Response: Updated textboxes with new language entries
```

### 4.3 Output Destination

```
Finalization output:
- Generate Art Direction → visual_descriptions added to images
- Translated content → new language entries in textboxes
- Final spreads → copied to snapshot.spreads[]

This is the transition point:
manuscript.dummies[] → snapshot.spreads[]
```

### 4.4 Accessibility

```
TranslateDialog:
  role="dialog"
  aria-modal="true"
  aria-labelledby="translate-dialog-title"

  Focus trap: Tab cycles within dialog
  Escape key: closes dialog

Language select:
  aria-label="Select target language"
```
