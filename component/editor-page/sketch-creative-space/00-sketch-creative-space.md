# SketchCreativeSpace: Component Design

**Screenshot:** `screenshots/manuscript-sketch-space.png`

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            SketchCreativeSpace                                   │
│  ┌────────────────────────────┐  ┌────────────────────────────────────────────┐  │
│  │       SketchSidebar        │  │  Conditional View by activeSketchType:     │  │
│  │  ┌──────────────────────┐  │  │                                            │  │
│  │  │ SidebarHeader        │  │  │  'characters' | 'props':                   │  │
│  │  │ "Sketch"        [⚏]  │  │  │  ┌──────────────────────────────────────┐  │  │
│  │  └──────────────────────┘  │  │  │  SketchViewer (NEW)                  │  │  │
│  │  ┌──────────────────────┐  │  │  │  ┌────────────────────────────────┐  │  │  │
│  │  │ SketchTypeList       │  │  │  │  │  SheetPreview (large view)     │  │  │  │
│  │  │ ┌────────────────┐   │  │  │  │  │  [Character/Prop Sheet Image]  │  │  │  │
│  │  │ │ ◎ Characters ∨ │   │  │  │  │  └────────────────────────────────┘  │  │  │
│  │  │ │  PromptPanel   │   │  │  │  │  ┌────────────────────────────────┐  │  │  │
│  │  │ └────────────────┘   │  │  │  │  │  SheetThumbnailList (filmstrip)│  │  │  │
│  │  │ ┌────────────────┐   │  │  │  │  │  [1][2][3][4][+NEW]            │  │  │  │
│  │  │ │ ◎ Props      > │   │  │  │  │  └────────────────────────────────┘  │  │  │
│  │  │ └────────────────┘   │  │  │  └──────────────────────────────────────┘  │  │
│  │  │ ┌────────────────┐   │  │  │                                            │  │
│  │  │ │ ▣ Spreads    > │   │  │  │  'spreads':                                │  │
│  │  │ └────────────────┘   │  │  │  ┌──────────────────────────────────────┐  │  │
│  │  └──────────────────────┘  │  │  │  ManuscriptSpreadView (REUSE)        │  │  │
│  └────────────────────────────┘  │  │  mode='finalize', editable           │  │  │
│                                  │  └──────────────────────────────────────┘  │  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
        ┌─────────────────────┐
        │    SnapshotStore    │
        │   (Zustand global)  │
        └──────────┬──────────┘
                   │
         ┌─────────┼─────────────────────────────────────┐
         │         │ (selectors)                         │
         ▼         ▼                                     ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                            SketchCreativeSpace                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  Local State: activeSketchType, selectedSheetIndex                          │  │
│  │  Store: sketch = useSketch()                                                │  │
│  │  Store: characterSheets = useCharacterSheets()  // string[] (URLs)          │  │
│  │  Store: propSheets = usePropSheets()            // string[] (URLs)          │  │
│  │  Store: spreads = useSpreads()                  // Spread[] (for Spreads)   │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                              │                          │
│         ▼                                              ▼                          │
│  ┌──────────────────────────┐              ┌──────────────────────────────────┐   │
│  │      SketchSidebar       │              │  SketchViewer (characters/props) │   │
│  │                          │              │  OR                              │   │
│  │  Props:                  │              │  ManuscriptSpreadView (spreads)  │   │
│  │  • activeSketchType      │              │                                  │   │
│  │  • onSketchTypeChange    │              │  Props (SketchViewer):           │   │
│  │                          │              │  • sketchType                    │   │
│  │  (uses store internally) │              │  • selectedIndex                 │   │
│  └──────────────────────────┘              │  • onIndexChange                 │   │
│                                            │                                  │   │
│                                            │  Props (ManuscriptSpreadView):   │   │
│                                            │  • mode: 'finalize'              │   │
│                                            └──────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Sketch Types Comparison

| Type | Data Source | View Component | Generate Action |
|------|-------------|----------------|-----------------|
| Characters | `sketch.character_sheets[]` (URLs) | SketchViewer | ✅ Has PromptPanel |
| Props | `sketch.prop_sheets[]` (URLs) | SketchViewer | ✅ Has PromptPanel |
| Spreads | `snapshot.spreads` (NOT sketch) | ManuscriptSpreadView | ❌ No PromptPanel |

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** Container cho sketch sheets viewing. Left panel với accordion tabs (Characters, Props, Spreads), right panel conditional: SketchViewer for Characters/Props (image sheets), ManuscriptSpreadView for Spreads.

**Key Difference from DummyCreativeSpace:**
- Characters/Props show **image sheets** (URLs), not spread layouts
- Spreads tab reuses ManuscriptSpreadView with `snapshot.spreads` data

**Shared Types:**

```typescript
type SketchType = 'characters' | 'props' | 'spreads';

interface ManuscriptSketch {
  character_sheets: string[];  // Image URLs
  prop_sheets: string[];       // Image URLs
}
```

### 2.2 Interface

**Props & Local State:**

```typescript
interface SketchCreativeSpaceProps {
  // No props - pure store consumer
}

interface SketchCreativeSpaceState {
  activeSketchType: SketchType;      // 'characters' | 'props' | 'spreads'
  selectedSheetIndex: number;        // For Characters/Props (0-based index)
}
```

**Store Integration:**

```typescript
// SnapshotStore Selectors
sketch = useSketch();
characterSheets = useCharacterSheets();  // string[] (URLs)
propSheets = usePropSheets();            // string[] (URLs)
spreads = useSpreads();                  // Spread[] from snapshot.spreads

// SnapshotStore Actions
{
  addCharacterSheet,
  removeCharacterSheet,
  reorderCharacterSheets,
  addPropSheet,
  removePropSheet,
  reorderPropSheets,
  // Spreads actions (same as finalize mode)
  addSpread,
  updateSpread,
  deleteSpread,
  reorderSpreads,
} = useSnapshotActions();
```

### 2.3 Render Logic (pseudo)

```
SketchCreativeSpace:
  // Store selectors
  characterSheets = useCharacterSheets()
  propSheets = usePropSheets()
  spreads = useSpreads()

  // Local state
  [activeSketchType, setActiveSketchType] = useState('characters')
  [selectedSheetIndex, setSelectedSheetIndex] = useState(0)

  // Reset index when switching types
  handleSketchTypeChange(type):
    setActiveSketchType(type)
    setSelectedSheetIndex(0)

  handleSheetIndexChange(index):
    setSelectedSheetIndex(index)

  RENDER Container (flex row):

    // Left panel
    RENDER SketchSidebar với:
      - activeSketchType
      - onSketchTypeChange: handleSketchTypeChange

    // Right panel - conditional by type
    IF activeSketchType === 'characters' OR activeSketchType === 'props':
      sheets = activeSketchType === 'characters' ? characterSheets : propSheets

      IF sheets.length === 0:
        RENDER EmptyState với:
          - "No character sheets yet" / "No prop sheets yet"
          - "Generate sheets from your dummy"
          - [✨ Generate Sheets] button
      ELSE:
        RENDER SketchViewer với:
          - sketchType: activeSketchType
          - selectedIndex: selectedSheetIndex
          - onIndexChange: handleSheetIndexChange

    ELSE (activeSketchType === 'spreads'):
      RENDER ManuscriptSpreadView với:
        - mode: 'finalize'
        // Uses snapshot.spreads, editable
```

### 2.4 Visual

**Characters/Props View:**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────┐  ┌──────────────────────────────────────────────────┐│
│  │ Sketch            [⚏]  │  │                                                  ││
│  │ ┌────────────────────┐ │  │      ┌─────────────────────────────────┐         ││
│  │ │ ◎ Characters     ∨ │ │  │      │    [Large Sheet Preview]        │         ││
│  │ │   PROMPT           │ │  │      │         ┌─────────────┐         │         ││
│  │ │   [textarea]       │ │  │      │         │   [image]   │         │         ││
│  │ │   [✨ Generate]    │ │  │      │         └─────────────┘         │         ││
│  │ └────────────────────┘ │  │      │                                 │         ││
│  │ ◎ Props             >  │  │      └─────────────────────────────────┘         ││
│  │ ▣ Spreads           >  │  │                                                  ││
│  │                        │  ├──────────────────────────────────────────────────┤│
│  │                        │  │  ╔═════╗ ┌─────┐ ┌─────┐ ┌─────┐ ┌───────┐       ││
│  │                        │  │  ║  1  ║ │  2  │ │  3  │ │  4  │ │ +NEW  │ ← film││
│  │                        │  │  ╚═════╝ └─────┘ └─────┘ └─────┘ └───────┘  strip││
│  └────────────────────────┘  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Spreads View (reuses ManuscriptSpreadView):**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────┐  ┌──────────────────────────────────────────────────┐│
│  │ Sketch            [⚏] │  │ [⚏]                            ─●───── + 100%    ││
│  │ ◎ Characters        >  │  ├──────────────────────────────────────────────────┤│
│  │ ◎ Props             >  │  │           ┌─────────────────────────────┐        ││
│  │ ┌────────────────────┐ │  │           │  [Spread Editor Panel]      │        ││
│  │ │ ▣ Spreads       ∨  │ │  │           │  Left Page  │  Right Page   │        ││
│  │ │                    │ │  │           │  [img] 0    │  [text] 1     │        ││
│  │ │  (no prompt panel) │ │  │           └─────────────────────────────┘        ││
│  │ │  Uses existing     │ │  │                                                  ││
│  │ │  snapshot.spreads  │ │  ├──────────────────────────────────────────────────┤│
│  │ └────────────────────┘ │  │  ╔═══╗ ┌───┐ ┌───┐ ┌───┐ ┌───────┐               ││
│  │                        │  │  ║0-1║ │2-3│ │4-5│ │6-7│ │ +NEW  │               ││
│  │                        │  │  ╚═══╝ └───┘ └───┘ └───┘ └───────┘               ││
│  └────────────────────────┘  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Empty State (Characters/Props):**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌────────────────────────┐  ┌──────────────────────────────────────────────────┐│
│  │ Sketch            [⚏]  │  │                                                  ││
│  │ ┌────────────────────┐ │  │                                                  ││
│  │ │ ◎ Characters     ∨ │ │  │         📄 No character sheets yet               ││
│  │ │   PROMPT           │ │  │                                                  ││
│  │ │   [textarea]       │ │  │         Generate sheets from your dummy          ││
│  │ │   [✨ Generate]    │ │  │                                                  ││
│  │ └────────────────────┘ │  │   ┌─────────────────────────────────────────┐    ││
│  │ ◎ Props             >  │  │   │     ✨ Generate Character Sheets        │    ││
│  │ ▣ Spreads           >  │  │   └─────────────────────────────────────────┘    ││
│  └────────────────────────┘  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child).
> Child components tự thiết kế store integration trong file riêng.

### 3.1 SketchSidebar

📄 **Doc:** [component/editor-page/sketch-creative-space/01-sketch-sidebar.md](component/editor-page/sketch-creative-space/01-sketch-sidebar.md)

**Mục đích:** Left sidebar với accordion tabs for Characters, Props, Spreads. PromptPanel for Characters/Props only.

**Props & Callbacks:**

```typescript
interface SketchSidebarProps {
  activeSketchType: SketchType;
  onSketchTypeChange: (type: SketchType) => void;
}
```

---

### 3.2 SketchViewer

📄 **Doc:** [component/editor-page/sketch-creative-space/02-sketch-viewer.md](component/editor-page/sketch-creative-space/02-sketch-viewer.md)

**Mục đích:** Image sheet viewer with large preview and horizontal filmstrip. Used only for Characters and Props tabs.

**Props & Callbacks:**

```typescript
interface SketchViewerProps {
  sketchType: 'characters' | 'props';  // NOT 'spreads'
  selectedIndex: number;
  onIndexChange: (index: number) => void;
}
```

---

### 3.3 ManuscriptSpreadView

📄 **Doc:** [component/editor-page/shared/manuscript-spread-view/00-manuscript-spread-view.md](component/editor-page/shared/manuscript-spread-view/00-manuscript-spread-view.md)

**Mục đích:** Spread editor view. **REUSE** - used for Spreads tab with `mode='finalize'`.

**Props for SketchCreativeSpace usage:**

```typescript
// Usage in SketchCreativeSpace when activeSketchType === 'spreads'
<ManuscriptSpreadView
  mode="finalize"    // Uses snapshot.spreads
  // Editable - same as DummyCreativeSpace finalize mode
/>
```

---

## 4. Technical Notes

### 4.1 Key Design Decisions

**Spreads Tab Uses snapshot.spreads**
The Spreads tab does NOT use sketch data. It reuses:
- `ManuscriptSpreadView` component
- `useSpreads()` selector (from `snapshot.spreads`)
- Same as finalize mode, editable

This provides visual reference of final spreads within SketchCreativeSpace.

**Accordion with PromptPanel**
Same pattern as DocSidebar:
- Only one tab expanded at a time
- PromptPanel for Characters/Props (generate action)
- Spreads tab has no PromptPanel (uses existing data)

**Sheet Generation Workflow**
1. User creates dummy spreads in DummyCreativeSpace
2. User navigates to SketchCreativeSpace
3. User expands Characters/Props tab
4. User enters prompt and clicks "Generate"
5. AI analyzes dummy spreads, generates reference sheets
6. Sheets stored in `sketch.character_sheets[]` / `sketch.prop_sheets[]`

### 4.2 Layout Constants

| Element | Value |
|---------|-------|
| Sidebar width | 280px (fixed) |
| Sheet preview max-height | 60vh |
| Filmstrip height | 100px |
| Thumbnail size | 80×80px |

### 4.3 No Language Impact
SketchCreativeSpace is NOT affected by `currentLanguage`. Sketch sheets are image-only (no multilingual text).

### 4.4 Accessibility

| Element | Role | ARIA |
|---------|------|------|
| Sidebar | `complementary` | `aria-label="Sketch navigation"` |
| SketchTypeList | `tablist` | `aria-orientation="vertical"` |
| SketchTypeItem | `tab` | `aria-selected`, `aria-expanded` |
| SketchViewer | `main` | `aria-label="Sheet viewer"` |
| SheetThumbnailList | `listbox` | `aria-label="Sheet thumbnails"` |

---
