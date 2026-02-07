# SnapshotStore: Store Design

---

## 1. Overview

**Mục đích:** Zustand store quản lý toàn bộ snapshot data (manuscript, spreads, characters, props, stages). Sử dụng slice pattern để modularize theo domain.

**Pattern:** Slice pattern + Immer middleware

**Structure:**
```
stores/
├── snapshot-store/
│   ├── index.ts                    # Combined store + exports
│   ├── types.ts                    # Shared types
│   ├── selectors.ts                # Memoized selectors
│   └── slices/
│       ├── manuscript-slice.ts
│       ├── spreads-slice.ts
│       ├── characters-slice.ts
│       ├── props-slice.ts
│       └── stages-slice.ts
```

---

## 2. Shared Types

```typescript
// types.ts

// Re-export from DB types
export type {
  Manuscript,
  ManuscriptDoc,
  ManuscriptDummy,
  DummySpread,
  Spread,
  Character,
  Prop,
  Stage,
  Geometry,
  Typography,
} from '@/types/db';

// Store-specific types
export interface SnapshotMeta {
  id: string | null;
  bookId: string | null;
  version: string | null;
  tag: string | null;
}

export interface SyncState {
  isDirty: boolean;
  lastSavedAt: Date | null;
  isSaving: boolean;
  error: string | null;
}
```

---

## 3. Slice Interfaces

### 3.1 ManuscriptSlice

```typescript
interface ManuscriptSlice {
  // State
  manuscript: Manuscript;

  // Actions
  setManuscript: (manuscript: Manuscript) => void;

  // Doc actions
  updateDoc: (docType: DocType, content: string) => void;

  // Dummy actions
  updateDummySpreads: (dummyType: DummyType, spreads: DummySpread[]) => void;
  addDummySpread: (dummyType: DummyType, spread: DummySpread) => void;
  updateDummySpread: (dummyType: DummyType, spreadId: string, updates: Partial<DummySpread>) => void;
  deleteDummySpread: (dummyType: DummyType, spreadId: string) => void;
  reorderDummySpreads: (dummyType: DummyType, fromIndex: number, toIndex: number) => void;
}

type DocType = 'brief' | 'draft' | 'script';
type DummyType = 'prose' | 'poetry';
```

### 3.2 SpreadsSlice

```typescript
interface SpreadsSlice {
  // State
  spreads: Spread[];

  // Actions
  setSpreads: (spreads: Spread[]) => void;

  // CRUD
  addSpread: (spread: Spread) => void;
  updateSpread: (spreadId: string, updates: Partial<Spread>) => void;
  deleteSpread: (spreadId: string) => void;
  reorderSpreads: (fromIndex: number, toIndex: number) => void;

  // Nested: Images
  addSpreadImage: (spreadId: string, image: SpreadImage) => void;
  updateSpreadImage: (spreadId: string, imageIndex: number, updates: Partial<SpreadImage>) => void;
  deleteSpreadImage: (spreadId: string, imageIndex: number) => void;

  // Nested: Textboxes
  addSpreadTextbox: (spreadId: string, textbox: SpreadTextbox) => void;
  updateSpreadTextbox: (spreadId: string, textboxId: string, updates: Partial<SpreadTextbox>) => void;
  deleteSpreadTextbox: (spreadId: string, textboxId: string) => void;

  // Nested: Objects
  addSpreadObject: (spreadId: string, object: SpreadObject) => void;
  updateSpreadObject: (spreadId: string, objectId: string, updates: Partial<SpreadObject>) => void;
  deleteSpreadObject: (spreadId: string, objectId: string) => void;

  // Nested: Animations
  addSpreadAnimation: (spreadId: string, animation: SpreadAnimation) => void;
  updateSpreadAnimation: (spreadId: string, animationIndex: number, updates: Partial<SpreadAnimation>) => void;
  deleteSpreadAnimation: (spreadId: string, animationIndex: number) => void;
}
```

### 3.3 CharactersSlice

```typescript
interface CharactersSlice {
  // State
  characters: Character[];

  // Actions
  setCharacters: (characters: Character[]) => void;

  // CRUD
  addCharacter: (character: Character) => void;
  updateCharacter: (key: string, updates: Partial<Character>) => void;
  deleteCharacter: (key: string) => void;
  reorderCharacters: (fromIndex: number, toIndex: number) => void;

  // Nested: Variants
  addCharacterVariant: (key: string, variant: CharacterVariant) => void;
  updateCharacterVariant: (key: string, variantKey: string, updates: Partial<CharacterVariant>) => void;
  deleteCharacterVariant: (key: string, variantKey: string) => void;

  // Nested: Voices
  addCharacterVoice: (key: string, voice: CharacterVoice) => void;
  updateCharacterVoice: (key: string, voiceKey: string, updates: Partial<CharacterVoice>) => void;
  deleteCharacterVoice: (key: string, voiceKey: string) => void;
}
```

### 3.4 PropsSlice

```typescript
interface PropsSlice {
  // State
  props: Prop[];

  // Actions
  setProps: (props: Prop[]) => void;

  // CRUD
  addProp: (prop: Prop) => void;
  updateProp: (key: string, updates: Partial<Prop>) => void;
  deleteProp: (key: string) => void;
  reorderProps: (fromIndex: number, toIndex: number) => void;

  // Nested: States
  addPropState: (key: string, state: PropState) => void;
  updatePropState: (key: string, stateKey: string, updates: Partial<PropState>) => void;
  deletePropState: (key: string, stateKey: string) => void;
}
```

### 3.5 StagesSlice

```typescript
interface StagesSlice {
  // State
  stages: Stage[];

  // Actions
  setStages: (stages: Stage[]) => void;

  // CRUD
  addStage: (stage: Stage) => void;
  updateStage: (key: string, updates: Partial<Stage>) => void;
  deleteStage: (key: string) => void;
  reorderStages: (fromIndex: number, toIndex: number) => void;

  // Nested: Settings
  addStageSetting: (key: string, setting: StageSetting) => void;
  updateStageSetting: (key: string, settingKey: string, updates: Partial<StageSetting>) => void;
  deleteStageSetting: (key: string, settingKey: string) => void;
}
```

### 3.6 MetaSlice

```typescript
interface MetaSlice {
  // State
  meta: SnapshotMeta;
  sync: SyncState;

  // Actions
  setMeta: (meta: SnapshotMeta) => void;
  markDirty: () => void;
  markClean: () => void;
  setSaving: (isSaving: boolean) => void;
  setSaveError: (error: string | null) => void;
}
```

---

## 4. Combined Store Interface

```typescript
// SnapshotStore = union of all slices
type SnapshotStore =
  & ManuscriptSlice
  & SpreadsSlice
  & CharactersSlice
  & PropsSlice
  & StagesSlice
  & MetaSlice
  & {
      // Top-level actions
      initSnapshot: (snapshot: Snapshot) => void;
      resetSnapshot: () => void;
    };
```

---

## 5. Selectors Interface

```typescript
// selectors.ts

// === Atomic Selectors (single value) ===

// Meta
export const useSnapshotId: () => string | null;
export const useIsDirty: () => boolean;
export const useIsSaving: () => boolean;

// Manuscript
export const useManuscript: () => Manuscript;
export const useDoc: (type: DocType) => ManuscriptDoc | undefined;
export const useDummy: (type: DummyType) => ManuscriptDummy | undefined;
export const useDummySpreads: (type: DummyType) => DummySpread[];
export const useDummySpreadIds: (type: DummyType) => string[];  // shallow compare
export const useDummySpreadById: (type: DummyType, id: string) => DummySpread | undefined;

// Spreads
export const useSpreads: () => Spread[];
export const useSpreadById: (id: string) => Spread | undefined;
export const useSpreadIds: () => string[];  // shallow compare
export const useSpreadCount: () => number;

// Characters
export const useCharacters: () => Character[];
export const useCharacterByKey: (key: string) => Character | undefined;
export const useCharacterKeys: () => string[];  // shallow compare

// Props
export const useProps: () => Prop[];
export const usePropByKey: (key: string) => Prop | undefined;
export const usePropKeys: () => string[];  // shallow compare

// Stages
export const useStages: () => Stage[];
export const useStageByKey: (key: string) => Stage | undefined;
export const useStageKeys: () => string[];  // shallow compare


// === Computed Selectors ===

// Cross-slice: Get all entity mentions for autocomplete
export const useEntityMentions: () => EntityMention[];

// Cross-slice: Get character/prop/stage used in a spread
export const useSpreadEntities: (spreadId: string) => {
  characters: Character[];
  props: Prop[];
  stages: Stage[];
};


// === Actions-only Hook (no re-render) ===

export const useSnapshotActions: () => SnapshotActions;

interface SnapshotActions {
  // Manuscript
  updateDoc: ManuscriptSlice['updateDoc'];
  updateDummySpreads: ManuscriptSlice['updateDummySpreads'];
  addDummySpread: ManuscriptSlice['addDummySpread'];
  updateDummySpread: ManuscriptSlice['updateDummySpread'];
  deleteDummySpread: ManuscriptSlice['deleteDummySpread'];
  reorderDummySpreads: ManuscriptSlice['reorderDummySpreads'];

  // Spreads
  addSpread: SpreadsSlice['addSpread'];
  updateSpread: SpreadsSlice['updateSpread'];
  deleteSpread: SpreadsSlice['deleteSpread'];
  reorderSpreads: SpreadsSlice['reorderSpreads'];
  // ... nested actions

  // Characters
  addCharacter: CharactersSlice['addCharacter'];
  updateCharacter: CharactersSlice['updateCharacter'];
  deleteCharacter: CharactersSlice['deleteCharacter'];
  reorderCharacters: CharactersSlice['reorderCharacters'];
  // ... nested actions

  // Props
  addProp: PropsSlice['addProp'];
  updateProp: PropsSlice['updateProp'];
  deleteProp: PropsSlice['deleteProp'];
  reorderProps: PropsSlice['reorderProps'];
  // ... nested actions

  // Stages
  addStage: StagesSlice['addStage'];
  updateStage: StagesSlice['updateStage'];
  deleteStage: StagesSlice['deleteStage'];
  reorderStages: StagesSlice['reorderStages'];
  // ... nested actions

  // Meta
  markDirty: MetaSlice['markDirty'];
}
```

---

## 6. Usage Patterns

### 6.1 Component Subscription

```typescript
// List component: subscribe IDs only
function SpreadThumbnailList() {
  const spreadIds = useSpreadIds();  // shallow compare
  return spreadIds.map(id => <SpreadThumbnail key={id} id={id} />);
}

// Item component: subscribe single item
function SpreadThumbnail({ id }: { id: string }) {
  const spread = useSpreadById(id);
  // Only re-renders when this specific spread changes
}
```

### 6.2 Actions Usage

```typescript
function SpreadEditor({ spreadId }: Props) {
  const spread = useSpreadById(spreadId);
  const { updateSpread, addSpreadTextbox } = useSnapshotActions();

  const handleImageMove = (imageIndex: number, geometry: Geometry) => {
    updateSpread(spreadId, {
      images: spread.images.map((img, i) =>
        i === imageIndex ? { ...img, geometry } : img
      )
    });
  };
}
```

### 6.3 Cross-slice Access

```typescript
// Inside a slice, can access other slices via get()
const createCharactersSlice: StateCreator<SnapshotStore, ...> = (set, get) => ({
  deleteCharacterAndCleanup: (key) => set((state) => {
    // Delete character
    state.characters = state.characters.filter(c => c.key !== key);

    // Cross-slice: cleanup references in spreads
    state.spreads.forEach(spread => {
      spread.objects = spread.objects.filter(obj => obj.name !== key);
    });
  }),
});
```

---

## 7. Technical Notes

### 7.1 Middleware Stack

```typescript
create<SnapshotStore>()(
  devtools(                    // Redux DevTools
    subscribeWithSelector(     // Granular subscriptions
      immer(                   // Immutable updates with mutation syntax
        (...args) => ({
          ...createManuscriptSlice(...args),
          ...createSpreadsSlice(...args),
          ...createCharactersSlice(...args),
          ...createPropsSlice(...args),
          ...createStagesSlice(...args),
          ...createMetaSlice(...args),
        })
      )
    ),
    { name: 'snapshot-store' }
  )
);
```

### 7.2 Backward Compatibility

Existing data without `id` field:

```typescript
initSnapshot: (snapshot) => set((state) => {
  // Ensure all spreads have IDs
  state.spreads = snapshot.spreads.map(s => ({
    ...s,
    id: s.id || generateSpreadId()
  }));

  // Ensure all dummy spreads have IDs
  state.manuscript.dummies.forEach(dummy => {
    dummy.spreads = dummy.spreads.map(s => ({
      ...s,
      id: s.id || generateSpreadId()
    }));
  });
});
```

### 7.3 Re-render Optimization

| Selector | Comparison | Re-render when |
|----------|------------|----------------|
| `useSpreadById(id)` | `Object.is` | Spread reference changes |
| `useSpreadIds()` | `shallow` | Array content changes |
| `useSpreadCount()` | `===` | Length changes |
| `useSnapshotActions()` | None | Never (stable reference) |

### 7.4 Autosave Integration

```typescript
// hooks/use-autosave.ts
export function useAutosave(debounceMs = 60_000) {
  const isDirty = useIsDirty();
  const { markClean, setSaving, setSaveError } = useSnapshotActions();

  useEffect(() => {
    if (!isDirty) return;

    const timeout = setTimeout(async () => {
      setSaving(true);
      try {
        const state = useSnapshotStore.getState();
        await saveSnapshotToServer(state);
        markClean();
      } catch (e) {
        setSaveError(e.message);
      } finally {
        setSaving(false);
      }
    }, debounceMs);

    return () => clearTimeout(timeout);
  }, [isDirty]);
}
```

---

## 8. Related Docs

- Database Schema: [DATABASE-SCHEMA.md](../../DATABASE-SCHEMA.md)
- EditorPage: [00-editor-page.md](../editor-page/00-editor-page.md)
- ManuscriptCreativeSpace: [03-manuscript-creative-space.md](../editor-page/03-manuscript-creative-space.md)
