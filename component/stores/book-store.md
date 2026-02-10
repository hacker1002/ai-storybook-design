# BookStore: Store Design

---

## 1. Overview

**Mục đích:** Zustand store quản lý book metadata và settings. Tách biệt với SnapshotStore (content data).

**Pattern:** Simple store với Immer middleware

**Structure:**
```
stores/
├── snapshot-store/           # Content data
├── editor-settings-store.ts  # UI settings
└── book-store.ts             # Book metadata (NEW)
```

---

## 2. Types

```typescript
// Re-export from DB types
export type {
  BookMeta,        // id, title, description, ownerId, step, type, originalLanguage, currentVersion
  BookSettings,    // bookType, dimension, targetAudience, targetCoreValue, formatGenre, contentGenre, writingStyle
  BookReferences,  // eraId, locationId, artstyleId
  BookCover,
  BookTypography,
  BookTemplateLayout,
  BookBackgroundMusic,
  BookRemix,
  BookPrintExport,
} from '@/types/db';

// Store-specific
interface BookSyncState {
  isDirty: boolean;
  isSaving: boolean;
  lastSavedAt: Date | null;
  error: string | null;
}
```

> **Note:** Xem chi tiết types trong [DATABASE-SCHEMA.md](DATABASE-SCHEMA.md)

---

## 3. Store Interface

```typescript
interface BookStore {
  // === State ===
  meta: BookMeta | null;
  settings: BookSettings;
  references: BookReferences;
  cover: BookCover;
  typography: BookTypography;
  templateLayout: BookTemplateLayout;
  backgroundMusic: BookBackgroundMusic;
  remix: BookRemix;
  printExport: BookPrintExport;
  sync: BookSyncState;

  // === Actions ===
  // Generic updates (Partial)
  updateMeta: (updates: Partial<BookMeta>) => void;
  updateSettings: (updates: Partial<BookSettings>) => void;
  updateReferences: (updates: Partial<BookReferences>) => void;
  updateCover: (updates: Partial<BookCover>) => void;
  updateTypography: (updates: Partial<BookTypography>) => void;
  updateTemplateLayout: (updates: Partial<BookTemplateLayout>) => void;
  updateBackgroundMusic: (updates: Partial<BookBackgroundMusic>) => void;
  updateRemix: (updates: Partial<BookRemix>) => void;
  updatePrintExport: (updates: Partial<BookPrintExport>) => void;

  // Remix array operations
  addRemixLanguage: (language: RemixLanguage) => void;
  removeRemixLanguage: (code: string) => void;
  addRemixAccessResource: (resource: RemixAccessResource) => void;
  removeRemixAccessResource: (key: string) => void;

  // Sync
  markDirty: () => void;
  markClean: () => void;
  setSaving: (isSaving: boolean) => void;
  setSaveError: (error: string | null) => void;

  // Lifecycle
  initBook: (book: Book) => void;
  resetBook: () => void;
}
```

---

## 4. Selectors

```typescript
// === State Selectors ===
export const useBookMeta: () => BookMeta | null;
export const useBookSettings: () => BookSettings;
export const useBookReferences: () => BookReferences;
export const useBookCover: () => BookCover;
export const useBookTypography: () => BookTypography;
export const useBookTemplateLayout: () => BookTemplateLayout;
export const useBackgroundMusic: () => BookBackgroundMusic;
export const useBookRemix: () => BookRemix;
export const useBookPrintExport: () => BookPrintExport;
export const useBookSync: () => BookSyncState;

// === Computed Selectors ===
export const useIsSourceBook: () => boolean;        // meta.type === 0
export const useHasRequiredSettings: () => boolean; // all required fields filled

// === Actions Hook (no re-render) ===
export const useBookActions: () => BookActions;
```

---

## 5. Usage Patterns

### Initialize & Access

```typescript
function EditorPage({ bookId }: Props) {
  const { initBook } = useBookActions();
  const { data: book } = useQuery(['book', bookId], () => fetchBook(bookId));

  useEffect(() => {
    if (book) initBook(book);
  }, [book, initBook]);
}

function BookHeader() {
  const meta = useBookMeta();
  const { updateMeta } = useBookActions();

  return <EditableTitle value={meta?.title} onChange={(title) => updateMeta({ title })} />;
}
```

### Settings Form

```typescript
function BookSettingsForm() {
  const settings = useBookSettings();
  const { updateSettings } = useBookActions();

  return (
    <Select
      value={settings.targetAudience}
      onChange={(v) => updateSettings({ targetAudience: v })}
    />
  );
}
```

---

## 6. Technical Notes

### Middleware

```typescript
export const useBookStore = create<BookStore>()(
  devtools(immer((set) => ({ /* ... */ })), { name: 'book-store' })
);
```

### Separation from SnapshotStore

| BookStore | SnapshotStore |
|-----------|---------------|
| Book metadata | Snapshot content |
| 1:1 with books table | 1:N snapshots per book |
| Updated infrequently | Updated frequently |
| Shallow state | Deeply nested arrays |

### Autosave

```typescript
export function useBookAutosave(debounceMs = 5_000) {
  const { isDirty } = useBookSync();
  const { markClean, setSaving } = useBookActions();

  useEffect(() => {
    if (!isDirty) return;
    const timeout = setTimeout(() => saveBook(), debounceMs);
    return () => clearTimeout(timeout);
  }, [isDirty]);
}
```

---

## 7. Related Docs

- Database Schema: [DATABASE-SCHEMA.md](DATABASE-SCHEMA.md)
- EditorPage: [README.md](../editor-page/README.md)
