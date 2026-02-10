# [ComponentName]: Component Design

---

## 1. Diagrams

### 1.1 Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        [ComponentName]                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      [ChildComponent1]                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌─────────────┬─────────────────────────────────┬───────────┐  │
│  │ [Child2]    │        [Child3]                 │  [Child4] │  │
│  └─────────────┴─────────────────────────────────┴───────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
┌─────────────┐                              ┌─────────────────────┐
│   API/DB    │                              │   Zustand Stores    │
└──────┬──────┘                              │  ┌───────────────┐  │
       │                                     │  │ useXxxStore   │  │
       │                                     │  │ useYyyStore   │  │
       ▼                                     │  └───────────────┘  │
┌──────────────────────────────────────────┐ └──────────┬──────────┘
│            [ComponentName]               │            │
│  ┌────────────────────────────────────┐  │◄───────────┘
│  │  Local State: [list variables]     │  │  (selectors, actions)
│  └────────────────────────────────────┘  │
│         │              │           │     │
│         ▼              ▼           ▼     │
│  ┌───────────┐  ┌───────────┐ ┌───────┐  │
│  │  [Child1] │  │  [Child2] │ │[Chil3]│  │
│  │ Props:    │  │ Props:    │ │ Props:│  │
│  │ •prop1    │  │ •prop1    │ │ •prop1│  │
│  │ Callbacks:│  │ Callbacks:│ │Callbk:│  │
│  │ •onAction │  │ •onAction │ │•onAct │  │
│  └───────────┘  └───────────┘ └───────┘  │
└──────────────────────────────────────────┘
```

**Lưu ý:**
- Mũi tên từ Store → Component: selectors để đọc data
- Mũi tên từ Component → Store: actions để update data

### 1.x Table/Graph Mapping/Summary for some special global logic

---

## 2. Root Component Design

### 2.1 Overview

**Mục đích:** [Mô tả ngắn gọn nhiệm vụ của component]

**Shared Types:**

```typescript
type [TypeName] = 'value1' | 'value2' | 'value3';

interface [SharedInterface] {
  property1: string;
  property2: number;
}
```

### 2.2 Interface

**Props & Local State:**

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

**Store Integration:**

```typescript
// State Selectors
spread = useSpreadById(spreadId);
currentPage = useCurrentPage();
isLoading = useSnapshotLoading();

// Actions
{ updateSpread, addSpreadTextbox, deleteSpread } = useSnapshotActions();
{ setCurrentPage } = useEditorActions();
```

**Lưu ý:** Chỉ liệt kê selectors và actions thực tế component sử dụng.

### 2.3 Render Logic (pseudo)

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

### 2.4 Visual

```
┌─────────────────────────────────────────┐
│ ┌─────────────────────────────────────┐ │
│ │         [ChildComponent1]           │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────┐ ┌───────────────┐ ┌───────┐ │
│ │ Child2  │ │    Child3     │ │Child4 │ │
│ └─────────┘ └───────────────┘ └───────┘ │
└─────────────────────────────────────────┘
```

**Visual States (nếu component có nhiều trạng thái UI):**

```
Default:              Loading:              Error:
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ [content]     │     │ ⏳ Loading... │     │ ⚠️ Error msg   │
│               │     │               │     │ [Retry]       │
└───────────────┘     └───────────────┘     └───────────────┘

Selected:             Disabled:
┌───────────────┐     ┌───────────────┐
│ ● [content]   │     │ [content]     │
│   ▲ active    │     │   (dimmed)    │
└───────────────┘     └───────────────┘
```

**Lưu ý:** nếu có screenshot mẫu thì thêm đường dẫn ảnh mẫu vào

---

## 3. Child Components Interface

> **Lưu ý quan trọng:**
> - Section này **CHỈ** định nghĩa **props và callbacks** (contract giữa parent-child)
> - **KHÔNG** thiết kế store integration tại đây — child component tự thiết kế store selectors/actions trong file riêng của nó
> - State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó

### 3.1 [ChildComponent1]

📄 **Doc:** [`{child-component1}.md`](./{child-component1}.md)
**(IMPORTANT) Add link child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Props & Callbacks:** *(chỉ parent-child interaction)*

```typescript
interface [ChildComponent1]Props {
  // Data từ parent
  data: DataType;
  isEnabled: boolean;

  // Callbacks về parent
  onAction: (param: ParamType) => void;
  onStatusChange: (status: StatusType) => void;
}
```

**Visual (optional - nếu cần clarify layout):**

```
┌─────────────────────┐
│  [Simple sketch]    │
└─────────────────────┘
```

---

### 3.2 [ChildComponent2]

📄 **Doc:** [`{child-component2}.md`](./{child-component2}.md)
**(IMPORTANT) Add link child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Props & Callbacks:** *(chỉ parent-child interaction)*

```typescript
interface [ChildComponent2]Props {
  items: Item[];
  selectedId: string | null;
  onItemSelect: (id: string) => void;
  onItemUpdate: (item: Item) => void;
}
```

---

### 3.3 [ChildComponent3]

📄 **Doc:** [`{child-component3}.md`](./{child-component3}.md)
**(IMPORTANT) Add link child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Special Impact:** ✅ **BỊ ẢNH HƯỞNG** — [Giải thích logic global ảnh hưởng tới component]

**Props & Callbacks:** *(chỉ parent-child interaction)*

```typescript
interface [ChildComponent3]Props {
  data: DataType;
  currentLanguage: Language;  // ⚡ language-aware
  onDataUpdate: (data: DataType) => void;
}
```

**Data Structure (nếu cần clarify format):**

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

## 4. Technical Notes

### 4.1 Key Design Decisions

**[Decision Title 1]**
[Mô tả quyết định]. Lý do: [giải thích tại sao].

**[Decision Title 2]**
[Mô tả quyết định]. Lý do: [giải thích tại sao].

### 4.2 [Note Title]

[Ghi chú bổ sung nếu cần]
