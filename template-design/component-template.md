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
│  ┌─────────────┬─────────────────────────────────┬───────────┐  │
│  │ [Child2]    │        [Child3]                 │  [Child4] │  │
│  └─────────────┴─────────────────────────────────┴───────────┘  │
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
│ [content]     │     │ ⏳ Loading... │     │ ⚠️ Error msg  │
│               │     │               │     │ [Retry]       │
└───────────────┘     └───────────────┘     └───────────────┘

Selected:             Disabled:
┌───────────────┐     ┌───────────────┐
│ ● [content]   │     │ [content]     │
│   ▲ active    │     │   (dimmed)    │
└───────────────┘     └───────────────┘
```

---

## 3. Child Components Interface

> **Lưu ý:** Section này chỉ định nghĩa **props và callbacks** (contract giữa parent-child).
> State và logic chi tiết của mỗi child sẽ được thiết kế trong file riêng của component đó.

### 3.1 [ChildComponent1]

📄 **Doc:** [`component/{page-name}/{hierarchy}-{child-component1}.md`](./component/{page-name}/{hierarchy}-{child-component1}.md)
**(Important) vẫn thêm link doc tới child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Props & Callbacks:**

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

📄 **Doc:** [`component/{page-name}/{hierarchy}-{child-component2}.md`](./component/{page-name}/{hierarchy}-{child-component2}.md)
**(Important) vẫn thêm link doc tới child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Props & Callbacks:**

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

📄 **Doc:** [`component/{page-name}/{hierarchy}-{child-component3}.md`](./component/{page-name}/{hierarchy}-{child-component3}.md)
**(Important) vẫn thêm link doc tới child component kể cả chưa có doc**

**Mục đích:** [Mô tả nhiệm vụ - 1 câu]

**Special Impact:** ✅ **BỊ ẢNH HƯỞNG** — [Giải thích logic global ảnh hưởng tới component]

**Props & Callbacks:**

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
