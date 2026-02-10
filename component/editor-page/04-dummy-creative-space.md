# DummyCreativeSpace: Component Design

**Screenshot:** `screenshots/manuscript-dummy-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            DummyCreativeSpace                                    │
│  ┌────────────────────────────┐  ┌────────────────────────────────────────────┐  │
│  │       DummySidebar         │  │       ManuscriptSpreadView (REUSE)         │  │
│  │  ┌──────────────────────┐  │  │  ┌──────────────────────────────────────┐  │  │
│  │  │ SidebarHeader        │  │  │  │ SpreadViewHeader                     │  │  │
│  │  │ "Dummies"      [+]   │  │  │  │ [⚏]              ─●───── + 100%      │  │  │
│  │  └──────────────────────┘  │  │  └──────────────────────────────────────┘  │  │
│  │  ┌──────────────────────┐  │  │  ┌──────────────────────────────────────┐  │  │
│  │  │ DummyList            │  │  │  │ SpreadEditorPanel                    │  │  │
│  │  │ ┌────────────────┐   │  │  │  │ ┌────────────────┬───────────────┐   │  │  │
│  │  │ │ DummyItem      │   │  │  │  │ │  Left Page     │  Right Page   │   │  │  │
│  │  │ │ (Prose) [∨]    │   │  │  │  │ │  [Image]       │   [Textbox]   │   │  │  │
│  │  │ │ PromptPanel    │   │  │  │  │ │      2         │       3       │   │  │  │
│  │  │ └────────────────┘   │  │  │  │ └────────────────┴───────────────┘   │  │  │
│  │  │ ┌────────────────┐   │  │  │  └──────────────────────────────────────┘  │  │
│  │  │ │ DummyItem      │   │  │  │  ┌──────────────────────────────────────┐  │  │
│  │  │ │ (Poetry) [>]   │   │  │  │  │ SpreadThumbnailList                  │  │  │
│  │  │ └────────────────┘   │  │  │  │ [0-1][2-3][4-5][6-7][8-9][+NEW]      │  │  │
│  │  └──────────────────────┘  │  │  └──────────────────────────────────────┘  │  │
│  └────────────────────────────┘  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                    ┌─────────────────────┐
                    │    SnapshotStore    │
                    │   (Zustand global)  │
                    └──────────┬──────────┘
                               │
                     ┌─────────┼─────────┐
                     │         │         │
                     ▼         ▼         ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                            DummyCreativeSpace                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Local State: activeDummyId                                                 │  │
│  │  Store: dummies = useDummies(), activeDummy = useDummyById(activeDummyId)   │  │
│  │  Actions: addDummy, updateDummy, deleteDummy, ... = useSnapshotActions()    │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                              │                          │
│         ▼                                              ▼                          │
│  ┌──────────────────────────┐              ┌──────────────────────────────────┐   │
│  │      DummySidebar        │              │    ManuscriptSpreadView (REUSE)  │   │
│  │                          │              │                                  │   │
│  │  Props:                  │              │  Props:                          │   │
│  │  • activeDummyId         │              │  • mode: 'dummy'                 │   │
│  │  • onDummySelect         │              │  • dummyId: activeDummyId        │   │
│  │  • onAddDummy            │              │                                  │   │
│  │                          │              │  (uses stores internally)        │   │
│  │  (uses store internally) │              │                                  │   │
│  └──────────────────────────┘              └──────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Dummy Types Comparison

| Feature | Prose Dummy | Poetry Dummy |
|---------|-------------|--------------|
| Layout template | Standard book layout | Poetry-specific layout |
| Spread structure | Same | Same |
| Generate prompt | Prose-optimized | Poetry-optimized |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Container cho Dummy layout editing. Left panel với dynamic dummy list (can add multiple), right panel với ManuscriptSpreadView (reuse).

**Key Difference from DocCreativeSpace:** Dummies are a dynamic list (can create multiple), not fixed tabs.

**Shared Types:**

```typescript
type DummyType = 'prose' | 'poetry';

interface ManuscriptDummy {
  id: string;
  title: string;
  type: DummyType;
  spreads: DummySpread[];
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface DummyCreativeSpaceProps {
  // No props - pure store consumer
}

interface DummyCreativeSpaceState {
  activeDummyId: string | null;  // Selected dummy ID (null if no dummies)
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors
dummies = useDummies();                      // ManuscriptDummy[]
activeDummy = useDummyById(activeDummyId);   // Selected dummy

// SnapshotStore Actions
{
  addDummy,              // Create new dummy (with type)
  updateDummy,           // Update dummy metadata
  deleteDummy,           // Delete dummy
  addDummySpread,
  updateDummySpread,
  deleteDummySpread,
  reorderDummySpreads
} = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
DummyCreativeSpace:
  // Store selectors
  dummies = useDummies()
  { addDummy } = useSnapshotActions()

  // Local state: select first dummy by default
  [activeDummyId, setActiveDummyId] = useState(() =>
    dummies.length > 0 ? dummies[0].id : null
  )

  // Update selection when dummies change
  useEffect(() => {
    IF activeDummyId === null AND dummies.length > 0:
      setActiveDummyId(dummies[0].id)
    ELSE IF activeDummyId NOT IN dummies.map(d => d.id):
      setActiveDummyId(dummies[0]?.id ?? null)
  }, [dummies])

  handleDummySelect(dummyId):
    setActiveDummyId(dummyId)

  handleAddDummy(type: DummyType):
    newDummy = addDummy({
      id: crypto.randomUUID(),
      type,
      title: generateDummyTitle(type, dummies),
      spreads: []
    })
    setActiveDummyId(newDummy.id)

  RENDER Container (flex row):

    // Left panel
    RENDER DummySidebar với:
      - activeDummyId
      - onDummySelect: handleDummySelect
      - onAddDummy: handleAddDummy

    // Right panel
    IF activeDummyId !== null:
      RENDER ManuscriptSpreadView với:
        - mode: 'dummy'
        - dummyId: activeDummyId
    ELSE:
      RENDER EmptyState "No dummies yet. Click + to create your first dummy."
```

### 2.4 Visual

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────┐  ┌──────────────────────────────────────────────────┐│
│  │ Dummies           [+]  │  │ [⚏]  Show full spread view      ─●────── + 100% ││
│  │ ┌────────────────────┐ │  ├──────────────────────────────────────────────────┤│
│  │ │ 📐 Prose Dummy   ∨ │ │  │                                                  ││
│  │ │   PROMPT           │ │  │    ┌────────┐  ┌────────┐  ┌────────┐            ││
│  │ │   [textarea]       │ │  │    │  0-1   │  │  2-3   │  │  4-5   │  ...       ││
│  │ │   [✨ Generate]    │ │  │    │ [img]  │  │ [img]  │  │ [img]  │            ││
│  │ └────────────────────┘ │  │    │   T    │  │   T    │  │   T    │            ││
│  │ 📐 Poetry Dummy     >  │  │    └────────┘  └────────┘  └────────┘            ││
│  │                        │  │                                                  ││
│  │                        │  │    ┌────────┐  ┌─────────────────┐               ││
│  │                        │  │    │  6-7   │  │       + NEW     │               ││
│  │                        │  │    │ [img]  │  │                 │               ││
│  │                        │  │    └────────┘  └─────────────────┘               ││
│  └────────────────────────┘  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State (no dummies):**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────┐  ┌──────────────────────────────────────────────────┐│
│  │ Dummies           [+]  │  │                                                  ││
│  │                        │  │                                                  ││
│  │  No dummies yet        │  │            📐 No dummies yet                     ││
│  │  Click + to create     │  │                                                  ││
│  │  your first dummy      │  │     Click + in the sidebar to create your        ││
│  │                        │  │     first dummy layout.                          ││
│  │                        │  │                                                  ││
│  └────────────────────────┘  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child).
> Child components tự thiết kế store integration trong file riêng.

### 3.1 DummySidebar

📄 **Doc:** [component/editor-page/04-01-dummy-sidebar.md](component/editor-page/04-01-dummy-sidebar.md)

**Mục đích:** Left sidebar với dynamic dummy list. Accordion-style với PromptPanel per dummy.

**Props & Callbacks:**

```typescript
interface DummySidebarProps {
  activeDummyId: string | null;
  onDummySelect: (dummyId: string) => void;
  onAddDummy: (type: DummyType) => void;
}
```

---

### 3.2 ManuscriptSpreadView

📄 **Doc:** [component/editor-page/04-02-manuscript-spread-view.md](component/editor-page/04-02-manuscript-spread-view.md)

**Mục đích:** Spread grid/editor view. **REUSE** - already designed, renumbered from `03-03`.

**Props for DummyCreativeSpace usage:**

```typescript
// Usage in DummyCreativeSpace
<ManuscriptSpreadView
  mode="dummy"
  dummyId={activeDummyId}  // Specific dummy ID
/>
```

**Interface (from existing design):**

```typescript
interface ManuscriptSpreadViewProps {
  mode: SpreadViewMode;        // 'dummy' | 'finalize'
  dummyId?: string;            // Required when mode='dummy'
}
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Dynamic Dummy List**
- User can create multiple dummies of same or different types
- Each dummy has: `id`, `title`, `type` ('prose' | 'poetry'), `spreads[]`
- Dummies are stored in `snapshot.manuscript.dummies[]`

**ManuscriptSpreadView Props Update**
Changed from `dummyType?: DummyType` to `dummyId?: string` to support multiple dummies of same type.

**Store Selector Update:**
```typescript
// Old: useDummySpreads(dummyType)
// New: useDummyById(dummyId)?.spreads
```

### 4.2 Generate Flow

| Action | Input | Output |
|--------|-------|--------|
| Create Dummy | Type selection | New dummy in `dummies[]` |
| Generate Spreads | Prompt + Script + Dummy type | Spread layouts → `dummy.spreads[]` |

### 4.3 Layout Constants

| Element | Value |
|---------|-------|
| Sidebar width | 280px (fixed) |
| Sidebar min-width | 240px |
| SpreadView min-width | 400px |

### 4.4 Dummy Title Generation

```typescript
function generateDummyTitle(type: DummyType, existingDummies: ManuscriptDummy[]): string {
  const countOfType = existingDummies.filter(d => d.type === type).length;
  const typeName = type === 'prose' ? 'Prose' : 'Poetry';
  return `${typeName} Dummy ${countOfType + 1}`;
}
// Examples: "Prose Dummy 1", "Poetry Dummy 1", "Prose Dummy 2"
```

### 4.5 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Sidebar | `complementary` | `aria-label="Dummy navigation"` |
| DummyList | `listbox` | `aria-label="Dummy layouts"` |
| DummyItem | `option` | `aria-selected` |
| SpreadView | `main` | `aria-label="Spread editor"` |

---
