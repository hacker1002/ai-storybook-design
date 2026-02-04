# [ComponentName]: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        [ComponentName]                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      [ChildComponent1]                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌─────────────┬─────────────────────────────┬───────────────┐  │
│  │ [Child2]    │        [Child3]             │   [Child4]    │  │
│  └─────────────┴─────────────────────────────┴───────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
                                    ┌─────────────┐
                                    │   API/DB    │
                                    └──────┬──────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                          [ComponentName]                          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  State: [list state variables]                              │  │
│  └────────────────────────────────────────────────────────────┘  │
│         │              │                              │          │
│         ▼              ▼                              ▼          │
│  ┌───────────┐  ┌───────────┐                  ┌───────────┐    │
│  │  [Child1] │  │  [Child2] │                  │  [Child3] │    │
│  │           │  │           │                  │           │    │
│  │ Props:    │  │ Props:    │                  │ Props:    │    │
│  │ •prop1    │  │ •prop1    │                  │ •prop1    │    │
│  │ •prop2    │  │ •prop2    │                  │ •prop2    │    │
│  │           │  │           │                  │           │    │
│  │ Callbacks:│  │ Callbacks:│                  │ Callbacks:│    │
│  │ •onAction │  │ •onAction │                  │ •onAction │    │
│  └───────────┘  └───────────┘                  └───────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 1.x Table/Graph Mapping/Summary for some special global logic

---

## 2. Component Designs

### 2.1 [ComponentName] (Root Component)

**Mục đích:** [Mô tả ngắn gọn nhiệm vụ của component]

**Shared Types:**

```typescript
type [TypeName] = 'value1' | 'value2' | 'value3';

interface [SharedInterface] {
  property1: string;
  property2: number;
}
```

**Interface:**

```typescript
interface [ComponentName]Props {
  propName: PropType;
}

interface [ComponentName]State {
  // Data
  dataItem: DataType | null;

  // UI State
  selectedId: string | null;
  isLoading: boolean;
}

interface [ComponentName]Callbacks {
  onAction: (param: ParamType) => void;
  onUpdate: (data: DataType) => void;
}
```

**Render Logic (pseudo):**

```
[ComponentName]:
  RENDER [ChildComponent1] với prop1, prop2, callbacks
  RENDER [ChildComponent2] với prop3, prop4

  SWITCH activeType:
    'type1' → RENDER [Variant1] với props
    'type2' → RENDER [Variant2] với props

  IF showOptional:
    RENDER [OptionalChild] với props
```

---

### 2.2 [ChildComponent1]

**Mục đích:** [Mô tả nhiệm vụ]

**Interface:**

```typescript
interface [ChildComponent1]Props {
  data: DataType;
  onAction: (param: ParamType) => void;
}

interface [ChildComponent1]State {
  localState: StateType;
}
```

---

### 2.3 [ChildComponent2]

**Mục đích:** [Mô tả nhiệm vụ]

**Interface:**

```typescript
interface [ChildComponent2]Props {
  items: Item[];
  selectedId: string | null;
  onItemSelect: (id: string) => void;
  onItemUpdate: (item: Item) => void;
}
```

**Configuration:**

```typescript
const [CONFIG_ITEMS]: [ConfigItem][] = [
  { id: 'item1', icon: 'Icon1', label: 'Label 1' },
  { id: 'item2', icon: 'Icon2', label: 'Label 2' },
];

function [helperFunction](param: ParamType): ReturnType {
  return /* logic */;
}
```

---

### 2.4 [ChildComponent3]

**Mục đích:** [Mô tả nhiệm vụ]

**Some special impact:** ✅ **BỊ ẢNH HƯỞNG** — [Giải thích 1 số logic global ảnh hưởng tới component]

**Interface:**

```typescript
interface [ChildComponent3]Props {
  data: DataType;
  currentLanguage: Language;  // ⚡ language-aware
  onDataUpdate: (data: DataType) => void;
}

interface [ChildComponent3]State {
  selectedId: string | null;
  zoom: number;
}
```

**Data Structure:**

```json
{
  "items": [
    {
      "id": "item_001",
      "content": [
        { "code": "en_US", "text": "English text" },
        { "code": "vi_VN", "text": "Tiếng Việt" }
      ]
    }
  ]
}
```

---

## 3. Technical Notes

### 3.1 Key Design Decisions

**[Decision Title 1]**
[Mô tả quyết định]. Lý do: [giải thích tại sao].

**[Decision Title 2]**
[Mô tả quyết định]. Lý do: [giải thích tại sao].

### 3.2 [Note Title]

[Ghi chú bổ sung nếu cần]

### 3.3 Khi nào cần refactor?

Cân nhắc refactor nếu xuất hiện nhu cầu:
- [Điều kiện 1]
- [Điều kiện 2]
- [Điều kiện 3]
