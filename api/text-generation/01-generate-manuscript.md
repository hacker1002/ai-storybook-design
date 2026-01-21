# 1. generate-manuscript

## Description
**Orchestrator function** - Điều phối việc tạo manuscript hoàn chỉnh qua 3 bước tuần tự. Function này gọi lần lượt 3 step functions và quản lý flow tổng thể.

## DB Schema Dependencies

### Tables Referenced
- `story`: Lưu thông tin story được tạo (title, summary, target_core_value, step, artstyle_id, target_audience)
- `snapshot`: Lưu version đầu tiên của story
- `art_styles`: Truy vấn description để truyền vào các step functions

### Fields Used
- `story.id`: UUID của story
- `story.artstyle_id`: FK → art_styles
- `snapshot.id`: UUID của snapshot
- `snapshot.story_id`: FK → story

## Parameters
```typescript
interface GenerateManuscriptParams {
  storyIdea: string;              // "Một chú mèo con tên Miu lạc đường..."
  attributes: {
    storyType: string[];          // ["adventure", "drama"]
    audience: string;             // "kids-3-6" | "kids-7-12" | "teens"
    length: "short" | "medium" | "long";
    artstyleId: string;           // UUID của artstyle trong DB
  };
  language?: string;              // "vi" | "en" - nếu không truyền, dùng "vi"
  options?: {
    skipStep2?: boolean;          // Bỏ qua Art Direction (default: false)
    skipStep3?: boolean;          // Bỏ qua Quality Check (default: false)
  };
}
```

## Result
```typescript
interface GenerateManuscriptResult {
  storyId: string;                // UUID của story được tạo
  snapshotId: string;             // UUID của snapshot được tạo
  status: "completed" | "partial";
  stepsCompleted: {
    step1: boolean;               // Story Analysis
    step2: boolean;               // Art Direction
    step3: boolean;               // Quality Check
  };
  summary: {
    title: string;
    spreadCount: number;
    characterCount: number;
    propCount: number;
    stageCount: number;
    flagCount: number;            // Số lượng issues từ Step 3
  };
}
```

## Flow
```
1. Validate input parameters
2. Gọi generate-story-draft (truyền storyIdea, attributes, language)
   → Tạo Story + Snapshot trong DB
   → Lưu: docs[], characters[], props[], stages[], spreads[]
3. Nếu không skipStep2:
   Gọi generate-art-direction (truyền storyId, snapshotId)
   → Update: characters[].visual_description, props[], stages[], spreads[].images[]
4. Nếu không skipStep3:
   Gọi generate-quality-check (truyền storyId, snapshotId)
   → Lưu: flags[] (nếu có issues)
5. Return result
```

## Error Handling
- Nếu Step 1 fail → Return error, không tạo story
- Nếu Step 2 fail → Return partial result, vẫn giữ Step 1 data
- Nếu Step 3 fail → Return partial result, vẫn giữ Step 1 + Step 2 data
