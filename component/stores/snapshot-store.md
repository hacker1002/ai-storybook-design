# SnapshotStore: Store Design

---

## 1. Overview

**Mục đích:** Zustand store quản lý toàn bộ snapshot data (docs, dummies, sketch, spreads, characters, props, stages). Sử dụng slice pattern để modularize theo domain.

**Pattern:** Slice pattern + Immer middleware

**Structure:**
```
stores/
├── snapshot-store/
│   ├── index.ts                    # Combined store + exports
│   ├── types.ts                    # Shared types
│   ├── selectors.ts                # Memoized selectors
│   └── slices/
│       ├── docs-slice.ts           # brief, draft, script
│       ├── dummies-slice.ts        # dummy spreads
│       ├── sketch-slice.ts         # centralized sketch data
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
  ManuscriptDoc,
  ManuscriptDummy,
  DummySpread,
  Sketch,
  Spread,
  SpreadImage,
  SpreadTextbox,
  SpreadObject,
  SpreadAnimation,
  Character,
  CharacterVariant,
  CharacterVoice,
  Prop,
  PropState,
  Stage,
  StageSetting,
  Geometry,
  Typography,
  CropSheet,
  Crop,
  Sound,
  Background,
  ImageReference,
  Fill,
  Outline,
  Audio,
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

export type DocType = 'brief' | 'draft' | 'script' | 'other';
export type DummyType = 'prose' | 'poetry';
```

---

## 3. Slice Interfaces

### 3.1 DocsSlice

Manages manuscript documents (brief, draft, script, other).

```typescript
interface DocsSlice {
  // State
  docs: ManuscriptDoc[];

  // Actions
  setDocs: (docs: ManuscriptDoc[]) => void;
  addDoc: (doc: ManuscriptDoc) => void;                    // Add new doc (typically 'other' type)
  updateDoc: (index: number, updates: Partial<ManuscriptDoc>) => void;
  updateDocTitle: (index: number, title: string) => void;  // For 'other' type
  deleteDoc: (index: number) => void;                      // Only for 'other' type
  getDoc: (docType: DocType) => ManuscriptDoc | undefined;
}
```

### 3.2 DummiesSlice

Manages dummy versions (multiple versions per book, each with spreads).

```typescript
interface DummiesSlice {
  // State
  dummies: ManuscriptDummy[];

  // Actions
  setDummies: (dummies: ManuscriptDummy[]) => void;

  // Dummy-level CRUD (ID-based)
  addDummy: (dummy: ManuscriptDummy) => void;
  updateDummy: (dummyId: string, updates: Partial<ManuscriptDummy>) => void;
  deleteDummy: (dummyId: string) => void;
  getDummy: (dummyId: string) => ManuscriptDummy | undefined;

  // Spread-level (ID-based)
  addDummySpread: (dummyId: string, spread: DummySpread) => void;
  updateDummySpread: (dummyId: string, spreadId: string, updates: Partial<DummySpread>) => void;
  deleteDummySpread: (dummyId: string, spreadId: string) => void;
  reorderDummySpreads: (dummyId: string, fromIndex: number, toIndex: number) => void;

  // Bulk update
  updateDummySpreads: (dummyId: string, spreads: DummySpread[]) => void;
}
```

### 3.3 SketchSlice

Manages centralized sketch data (character sheets, prop sheets).

```typescript
interface SketchSlice {
  // State
  sketch: Sketch;

  // Actions
  setSketch: (sketch: Sketch) => void;

  // Linked dummy
  setSketchDummyId: (dummyId: string) => void;

  // Character sheets
  addCharacterSheet: (sheetUrl: string) => void;
  removeCharacterSheet: (index: number) => void;
  reorderCharacterSheets: (fromIndex: number, toIndex: number) => void;

  // Prop sheets
  addPropSheet: (sheetUrl: string) => void;
  removePropSheet: (index: number) => void;
  reorderPropSheets: (fromIndex: number, toIndex: number) => void;

  // Clear
  clearSketch: () => void;
}

// Sketch type (from DB)
interface Sketch {
  dummy_id: string | null;
  character_sheets: string[];
  prop_sheets: string[];
}
```

### 3.4 SpreadsSlice

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

  // Nested: Images (now with ID)
  addSpreadImage: (spreadId: string, image: SpreadImage) => void;
  updateSpreadImage: (spreadId: string, imageId: string, updates: Partial<SpreadImage>) => void;
  deleteSpreadImage: (spreadId: string, imageId: string) => void;

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

### 3.5 CharactersSlice

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

  // Nested: Variants (sketches[] removed - now in SketchSlice)
  addCharacterVariant: (key: string, variant: CharacterVariant) => void;
  updateCharacterVariant: (key: string, variantKey: string, updates: Partial<CharacterVariant>) => void;
  deleteCharacterVariant: (key: string, variantKey: string) => void;

  // Nested: Voices
  addCharacterVoice: (key: string, voice: CharacterVoice) => void;
  updateCharacterVoice: (key: string, voiceKey: string, updates: Partial<CharacterVoice>) => void;
  deleteCharacterVoice: (key: string, voiceKey: string) => void;

  // Nested: Crop Sheets
  addCharacterCropSheet: (key: string, cropSheet: CropSheet) => void;
  updateCharacterCropSheet: (key: string, cropSheetIndex: number, updates: Partial<CropSheet>) => void;
  deleteCharacterCropSheet: (key: string, cropSheetIndex: number) => void;
}
```

### 3.6 PropsSlice

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

  // Nested: States (sketches[] removed - now in SketchSlice)
  addPropState: (key: string, state: PropState) => void;
  updatePropState: (key: string, stateKey: string, updates: Partial<PropState>) => void;
  deletePropState: (key: string, stateKey: string) => void;

  // Nested: Crop Sheets
  addPropCropSheet: (key: string, cropSheet: CropSheet) => void;
  updatePropCropSheet: (key: string, cropSheetIndex: number, updates: Partial<CropSheet>) => void;
  deletePropCropSheet: (key: string, cropSheetIndex: number) => void;

  // Nested: Sounds
  addPropSound: (key: string, sound: Sound) => void;
  updatePropSound: (key: string, soundKey: string, updates: Partial<Sound>) => void;
  deletePropSound: (key: string, soundKey: string) => void;
}
```

### 3.7 StagesSlice

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

  // Nested: Settings (sketches[] removed - now in SketchSlice)
  addStageSetting: (key: string, setting: StageSetting) => void;
  updateStageSetting: (key: string, settingKey: string, updates: Partial<StageSetting>) => void;
  deleteStageSetting: (key: string, settingKey: string) => void;

  // Nested: Sounds
  addStageSound: (key: string, sound: Sound) => void;
  updateStageSound: (key: string, soundKey: string, updates: Partial<Sound>) => void;
  deleteStageSound: (key: string, soundKey: string) => void;
}
```

### 3.8 MetaSlice

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
  & DocsSlice
  & DummiesSlice
  & SketchSlice
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

// Docs
export const useDocs: () => ManuscriptDoc[];
export const useDocByIndex: (index: number) => ManuscriptDoc | undefined;
export const useDocByType: (type: DocType) => ManuscriptDoc | undefined;

// Dummies (ID-based)
export const useDummies: () => ManuscriptDummy[];
export const useDummyIds: () => string[];  // shallow compare
export const useDummyById: (dummyId: string) => ManuscriptDummy | undefined;
export const useDummySpreads: (dummyId: string) => DummySpread[];
export const useDummySpreadIds: (dummyId: string) => string[];  // shallow compare
export const useDummySpreadById: (dummyId: string, spreadId: string) => DummySpread | undefined;

// Sketch
export const useSketch: () => Sketch;
export const useSketchDummyId: () => string | null;
export const useCharacterSheets: () => string[];
export const usePropSheets: () => string[];

// Spreads
export const useSpreads: () => Spread[];
export const useSpreadById: (id: string) => Spread | undefined;
export const useSpreadIds: () => string[];  // shallow compare
export const useSpreadCount: () => number;
export const useSpreadImageById: (spreadId: string, imageId: string) => SpreadImage | undefined;

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

// Cross-slice: Get objects linked to specific image
export const useObjectsByImageId: (spreadId: string, imageId: string) => SpreadObject[];


// === Actions-only Hook (no re-render) ===

export const useSnapshotActions: () => SnapshotActions;

interface SnapshotActions {
  // Docs
  addDoc: DocsSlice['addDoc'];
  updateDoc: DocsSlice['updateDoc'];
  updateDocTitle: DocsSlice['updateDocTitle'];
  deleteDoc: DocsSlice['deleteDoc'];

  // Dummies (ID-based)
  addDummy: DummiesSlice['addDummy'];
  updateDummy: DummiesSlice['updateDummy'];
  deleteDummy: DummiesSlice['deleteDummy'];
  addDummySpread: DummiesSlice['addDummySpread'];
  updateDummySpread: DummiesSlice['updateDummySpread'];
  deleteDummySpread: DummiesSlice['deleteDummySpread'];
  reorderDummySpreads: DummiesSlice['reorderDummySpreads'];
  updateDummySpreads: DummiesSlice['updateDummySpreads'];

  // Sketch
  setSketchDummyId: SketchSlice['setSketchDummyId'];
  addCharacterSheet: SketchSlice['addCharacterSheet'];
  removeCharacterSheet: SketchSlice['removeCharacterSheet'];
  addPropSheet: SketchSlice['addPropSheet'];
  removePropSheet: SketchSlice['removePropSheet'];
  clearSketch: SketchSlice['clearSketch'];

  // Spreads
  addSpread: SpreadsSlice['addSpread'];
  updateSpread: SpreadsSlice['updateSpread'];
  deleteSpread: SpreadsSlice['deleteSpread'];
  reorderSpreads: SpreadsSlice['reorderSpreads'];
  addSpreadImage: SpreadsSlice['addSpreadImage'];
  updateSpreadImage: SpreadsSlice['updateSpreadImage'];
  deleteSpreadImage: SpreadsSlice['deleteSpreadImage'];
  addSpreadTextbox: SpreadsSlice['addSpreadTextbox'];
  updateSpreadTextbox: SpreadsSlice['updateSpreadTextbox'];
  deleteSpreadTextbox: SpreadsSlice['deleteSpreadTextbox'];
  addSpreadObject: SpreadsSlice['addSpreadObject'];
  updateSpreadObject: SpreadsSlice['updateSpreadObject'];
  deleteSpreadObject: SpreadsSlice['deleteSpreadObject'];
  addSpreadAnimation: SpreadsSlice['addSpreadAnimation'];
  updateSpreadAnimation: SpreadsSlice['updateSpreadAnimation'];
  deleteSpreadAnimation: SpreadsSlice['deleteSpreadAnimation'];

  // Characters
  addCharacter: CharactersSlice['addCharacter'];
  updateCharacter: CharactersSlice['updateCharacter'];
  deleteCharacter: CharactersSlice['deleteCharacter'];
  reorderCharacters: CharactersSlice['reorderCharacters'];
  addCharacterVariant: CharactersSlice['addCharacterVariant'];
  updateCharacterVariant: CharactersSlice['updateCharacterVariant'];
  deleteCharacterVariant: CharactersSlice['deleteCharacterVariant'];
  addCharacterVoice: CharactersSlice['addCharacterVoice'];
  updateCharacterVoice: CharactersSlice['updateCharacterVoice'];
  deleteCharacterVoice: CharactersSlice['deleteCharacterVoice'];
  addCharacterCropSheet: CharactersSlice['addCharacterCropSheet'];
  updateCharacterCropSheet: CharactersSlice['updateCharacterCropSheet'];
  deleteCharacterCropSheet: CharactersSlice['deleteCharacterCropSheet'];

  // Props
  addProp: PropsSlice['addProp'];
  updateProp: PropsSlice['updateProp'];
  deleteProp: PropsSlice['deleteProp'];
  reorderProps: PropsSlice['reorderProps'];
  addPropState: PropsSlice['addPropState'];
  updatePropState: PropsSlice['updatePropState'];
  deletePropState: PropsSlice['deletePropState'];
  addPropCropSheet: PropsSlice['addPropCropSheet'];
  updatePropCropSheet: PropsSlice['updatePropCropSheet'];
  deletePropCropSheet: PropsSlice['deletePropCropSheet'];
  addPropSound: PropsSlice['addPropSound'];
  updatePropSound: PropsSlice['updatePropSound'];
  deletePropSound: PropsSlice['deletePropSound'];

  // Stages
  addStage: StagesSlice['addStage'];
  updateStage: StagesSlice['updateStage'];
  deleteStage: StagesSlice['deleteStage'];
  reorderStages: StagesSlice['reorderStages'];
  addStageSetting: StagesSlice['addStageSetting'];
  updateStageSetting: StagesSlice['updateStageSetting'];
  deleteStageSetting: StagesSlice['deleteStageSetting'];
  addStageSound: StagesSlice['addStageSound'];
  updateStageSound: StagesSlice['updateStageSound'];
  deleteStageSound: StagesSlice['deleteStageSound'];

  // Meta
  markDirty: MetaSlice['markDirty'];

  // Top-level
  initSnapshot: SnapshotStore['initSnapshot'];
  resetSnapshot: SnapshotStore['resetSnapshot'];
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

// Image component: subscribe by ID (new)
function SpreadImageEditor({ spreadId, imageId }: Props) {
  const image = useSpreadImageById(spreadId, imageId);
  const { updateSpreadImage } = useSnapshotActions();
  // Only re-renders when this specific image changes
}
```

### 6.2 Actions Usage

```typescript
function SpreadEditor({ spreadId }: Props) {
  const spread = useSpreadById(spreadId);
  const { updateSpreadImage, addSpreadObject } = useSnapshotActions();

  const handleImageMove = (imageId: string, geometry: Geometry) => {
    updateSpreadImage(spreadId, imageId, { geometry });
  };

  const handleExtractObject = (imageId: string, objectData: Partial<SpreadObject>) => {
    addSpreadObject(spreadId, {
      id: crypto.randomUUID(),
      original_image_id: imageId,  // Link to source image
      ...objectData,
    });
  };
}
```

### 6.3 Dummy Workflow (ID-based)

```typescript
// List dummies
function DummyList() {
  const dummyIds = useDummyIds();  // shallow compare
  return dummyIds.map(id => <DummyItem key={id} dummyId={id} />);
}

// Single dummy item
function DummyItem({ dummyId }: { dummyId: string }) {
  const dummy = useDummyById(dummyId);
  const { updateDummy, deleteDummy } = useSnapshotActions();

  const handleRename = (title: string) => {
    updateDummy(dummyId, { title });
  };

  const handleDelete = () => {
    deleteDummy(dummyId);
  };
}

// Create new dummy
function CreateDummyButton() {
  const { addDummy } = useSnapshotActions();

  const handleCreate = () => {
    addDummy({
      id: crypto.randomUUID(),
      title: 'New Dummy',
      type: 'prose',
      spreads: [],
    });
  };
}
```

### 6.5 Sketch Workflow

```typescript
function SketchPanel() {
  const sketch = useSketch();
  const characterSheets = useCharacterSheets();
  const { addCharacterSheet, setSketchDummyId } = useSnapshotActions();

  const handleGenerateSheets = async (dummyId: string) => {
    setSketchDummyId(dummyId);
    const sheets = await generateCharacterSheets(dummyId);
    sheets.forEach(url => addCharacterSheet(url));
  };
}
```

### 6.6 Cross-slice Access

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
          ...createDocsSlice(...args),
          ...createDummiesSlice(...args),
          ...createSketchSlice(...args),
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

Existing data without `id` field or old structure:

```typescript
initSnapshot: (snapshot) => set((state) => {
  // Ensure all spreads have IDs
  state.spreads = snapshot.spreads.map(s => ({
    ...s,
    id: s.id || crypto.randomUUID(),
    images: s.images?.map(img => ({
      ...img,
      id: img.id || crypto.randomUUID(),  // New: images now have IDs
    })) || [],
  }));

  // Initialize docs (new structure)
  state.docs = snapshot.docs || [];

  // Initialize dummies (ID-based structure)
  state.dummies = (snapshot.dummies || []).map((dummy, index) => ({
    ...dummy,
    id: dummy.id || crypto.randomUUID(),
    title: dummy.title || `Dummy ${index + 1}`,
    spreads: dummy.spreads?.map(s => ({
      ...s,
      id: s.id || crypto.randomUUID(),
    })) || [],
  }));

  // Initialize sketch (new centralized structure)
  state.sketch = snapshot.sketch || {
    dummy_id: null,
    character_sheets: [],
    prop_sheets: [],
  };
});
```

### 7.3 Re-render Optimization

| Selector | Comparison | Re-render when |
|----------|------------|----------------|
| `useDummyById(id)` | `Object.is` | Dummy reference changes |
| `useDummyIds()` | `shallow` | Array content changes |
| `useSpreadById(id)` | `Object.is` | Spread reference changes |
| `useSpreadIds()` | `shallow` | Array content changes |
| `useSpreadCount()` | `===` | Length changes |
| `useSpreadImageById(sid, iid)` | `Object.is` | Image reference changes |
| `useCharacterSheets()` | `shallow` | Array content changes |
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

### 7.5 Data Migration Notes

Legacy snapshots có thể thiếu fields mới. `initSnapshot` cần handle:

- **IDs**: Auto-generate `id` cho spreads, images, dummy spreads nếu missing
- **Dummy ID/Title**: Auto-generate `id` và `title` cho dummies nếu missing (dummies giờ được identify bằng ID, không phải type)
- **docs/dummies**: Fallback `[]` nếu không có
- **sketch**: Fallback `{ dummy_id: null, character_sheets: [], prop_sheets: [] }`

> **Note:** Nếu cần migrate từ old `manuscript` structure hoặc scattered `sketches[]` trong variants/states, implement riêng migration util.

---

## 8. Related Docs

- Database Schema: [DATABASE-SCHEMA.md](../../DATABASE-SCHEMA.md)
- EditorPage: [00-editor-page.md](../editor-page/00-editor-page.md)
