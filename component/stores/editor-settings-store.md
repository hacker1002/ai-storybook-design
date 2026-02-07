# EditorSettingsStore: Store Design

---

## 1. Overview

**Mục đích:** Zustand store quản lý global UI settings cho Editor page. Tránh prop drilling cho `currentLanguage` và `currentStep`.

**Pattern:** Simple store (không cần slice pattern vì state đơn giản)

**Structure:**
```
stores/
├── snapshot-store/       # Existing - data state
│   └── ...
└── editor-settings-store.ts  # NEW - UI settings state
```

---

## 2. Types

```typescript
// types.ts hoặc inline

interface Language {
  name: string;       // "English", "Tiếng Việt", ...
  code: string;       // "en_US", "vi_VN", ...
}

type PipelineStep = 'idea' | 'sketch' | 'illustration' | 'retouch';
```

---

## 3. Store Interface

```typescript
interface EditorSettingsStore {
  // === State ===
  currentLanguage: Language;
  currentStep: PipelineStep;

  // === Actions ===
  setCurrentLanguage: (language: Language) => void;
  setCurrentStep: (step: PipelineStep) => void;

  // Bulk reset (on page unmount or book change)
  resetSettings: (language: Language, step: PipelineStep) => void;
}
```

---

## 4. Store Implementation

```typescript
// editor-settings-store.ts
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

const DEFAULT_LANGUAGE: Language = { name: 'English (US)', code: 'en_US' };
const DEFAULT_STEP: PipelineStep = 'idea';

export const useEditorSettingsStore = create<EditorSettingsStore>()(
  devtools(
    (set) => ({
      // State
      currentLanguage: DEFAULT_LANGUAGE,
      currentStep: DEFAULT_STEP,

      // Actions
      setCurrentLanguage: (language) => set({ currentLanguage: language }),
      setCurrentStep: (step) => set({ currentStep: step }),
      resetSettings: (language, step) => set({
        currentLanguage: language,
        currentStep: step
      }),
    }),
    { name: 'editor-settings-store' }
  )
);
```

---

## 5. Selectors

```typescript
// Atomic selectors (stable reference)
export const useCurrentLanguage = () =>
  useEditorSettingsStore((s) => s.currentLanguage);

export const useCurrentStep = () =>
  useEditorSettingsStore((s) => s.currentStep);

export const useLanguageCode = () =>
  useEditorSettingsStore((s) => s.currentLanguage.code);

// Actions-only hook (no re-render on state change)
export const useEditorSettingsActions = () =>
  useEditorSettingsStore((s) => ({
    setCurrentLanguage: s.setCurrentLanguage,
    setCurrentStep: s.setCurrentStep,
    resetSettings: s.resetSettings,
  }));
```

---

## 6. Usage Patterns

### 6.1 Initialize on EditorPage mount

```typescript
// EditorPage.tsx
function EditorPage({ bookId }: EditorPageProps) {
  const { resetSettings } = useEditorSettingsActions();
  const book = useBook(bookId);

  useEffect(() => {
    if (book) {
      // Init language from book.original_language
      const initialLanguage = {
        name: book.original_language_name,
        code: book.original_language,
      };
      resetSettings(initialLanguage, 'idea');
    }
  }, [book, resetSettings]);

  // ...
}
```

### 6.2 Consume in deep child components

```typescript
// SpreadTextbox.tsx (deeply nested)
function SpreadTextbox({ textbox }: Props) {
  const languageCode = useLanguageCode();

  // Direct access - no prop drilling
  const content = textbox[languageCode];

  return <div>{content?.text}</div>;
}
```

### 6.3 Header language selector

```typescript
// LanguageSelector.tsx
function LanguageSelector() {
  const currentLanguage = useCurrentLanguage();
  const { setCurrentLanguage } = useEditorSettingsActions();

  return (
    <Select
      value={currentLanguage.code}
      onChange={(code) => setCurrentLanguage(findLanguageByCode(code))}
    />
  );
}
```

### 6.4 Step-aware components

```typescript
// IconRail.tsx
function IconRail({ activeCreativeSpace, onCreativeSpaceChange }: Props) {
  const currentStep = useCurrentStep();

  // Enable/disable items based on step
  const isSpreadEnabled = currentStep !== 'idea';

  return (
    <nav>
      <IconItem icon="spreads" disabled={!isSpreadEnabled} />
      ...
    </nav>
  );
}
```

---

## 7. Migration Notes

### 7.1 Components to update

| Component | Change |
|-----------|--------|
| EditorPage | Remove `currentLanguage`, `currentStep` local state; init store on mount |
| EditorHeader | Remove `currentLanguage`, `currentStep` props; use selectors |
| IconRail | Remove `currentStep` prop; use `useCurrentStep()` |
| ManuscriptCreativeSpace | Remove `currentLanguage` prop; use `useCurrentLanguage()` |
| SpreadsCreativeSpace | Remove `currentLanguage` prop; use `useCurrentLanguage()` |
| AnimationsCreativeSpace | Remove `currentLanguage` prop; use `useCurrentLanguage()` |
| RightSidebar | Remove `currentLanguage`, `currentStep` props; use selectors |

### 7.2 Props to remove from EditorPage children

```typescript
// BEFORE
<ManuscriptCreativeSpace currentLanguage={currentLanguage} />
<SpreadsCreativeSpace currentStep={currentStep} currentLanguage={currentLanguage} />
<IconRail currentStep={currentStep} activeCreativeSpace={activeCreativeSpace} />

// AFTER
<ManuscriptCreativeSpace />
<SpreadsCreativeSpace />
<IconRail activeCreativeSpace={activeCreativeSpace} />
```

---

## 8. Technical Notes

### 8.1 Re-render Optimization

| Selector | Re-renders when |
|----------|-----------------|
| `useCurrentLanguage()` | Language object changes |
| `useCurrentStep()` | Step changes |
| `useLanguageCode()` | Language code string changes |
| `useEditorSettingsActions()` | Never (stable functions) |

### 8.2 DevTools

Store visible in Redux DevTools under name `editor-settings-store`.

### 8.3 Why not combine with SnapshotStore?

| Reason | Explanation |
|--------|-------------|
| Separation of concerns | Snapshot = data, EditorSettings = UI |
| Different lifecycles | Snapshot persists to DB, settings are session-only |
| Independent re-renders | Changing language shouldn't trigger snapshot selectors |

---

## 9. Related Docs

- Snapshot Store: [snapshot-store.md](./snapshot-store.md)
- EditorPage: [00-editor-page.md](../editor-page/00-editor-page.md)
