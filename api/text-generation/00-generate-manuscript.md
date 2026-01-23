# generate-manuscript

## Description
**Orchestrator function** - Điều phối việc tạo manuscript hoàn chỉnh qua 5 bước tuần tự. Function này gọi lần lượt các step functions và quản lý flow tổng thể.

## DB Schema Dependencies

### Tables Referenced
- `story`: Lưu thông tin story (title, summary, target_core_value, step, dimension, target_audience, genre, writing_style, era_id, location_id, artstyle_id)
- `snapshot`: Lưu version đầu tiên của story
- `art_styles`: Truy vấn description để truyền vào các step functions
- `eras`: Truy vấn era description cho context
- `locations`: Truy vấn location description cho context

### Fields Used
- `story.id`: UUID của story
- `story.dimension`: SMALLINT (1: Square, 2: A4 Landscape, 3: A4 Portrait)
- `story.target_audience`: SMALLINT (1: preschool, 2: primary, 3: tweens)
- `story.target_core_value`: VARCHAR(255)
- `story.genre`: SMALLINT (1-5)
- `story.writing_style`: SMALLINT (1-3)
- `story.era_id`: FK → eras
- `story.location_id`: FK → locations
- `story.artstyle_id`: FK → art_styles
- `snapshot.id`: UUID của snapshot
- `snapshot.story_id`: FK → story

## Parameters
```typescript
interface GenerateManuscriptParams {
  storyIdea: string;              // "Một chú mèo con tên Miu lạc đường..."
  attributes: {
    // General settings
    dimension: 1 | 2 | 3;         // 1: Square (20x20cm), 2: A4 Landscape (29.7x21cm), 3: A4 Portrait (21x29.7cm)
    targetAudience: 1 | 2 | 3;    // 1: preschool (2-5), 2: primary (6-8), 3: tweens (9-10)
    targetCoreValue: string;      // Đạo đức, Trí tuệ, Nghị lực (varchar 255)

    // Creative settings
    genre: 1 | 2 | 3 | 4 | 5;     // 1: fantasy, 2: scifi, 3: mystery, 4: romance, 5: horror
    writingStyle: 1 | 2 | 3;      // 1: Narrative, 2: Rhyming/Rhyme, 3: Humorous Fiction
    eraId: string;                // UUID → eras table
    locationId: string;           // UUID → locations table
    artstyleId: string;           // UUID → art_styles table
  };
  language?: string;              // "vi" | "en" - nếu không truyền, dùng "vi"
  options?: {
    skipArtDirection?: boolean;   // Bỏ qua Art Direction (Step 2,3,4) (default: false)
    skipQualityCheck?: boolean;   // Bỏ qua Quality Check (Step 5) (default: false)
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
    step1_storyDraft: boolean;           // Story Draft (Story Teller)
    step2_spreadVisualPlan: boolean;     // Spread Visual Plan (Art Director)
    step3_textRefinement: boolean;       // Text Refinement (Word Smith)
    step4_spreadComposition: boolean;    // Spread Composition (Art Director)
    step5_qualityCheck: boolean;         // Quality Check
  };
  summary: {
    title: string;
    spreadCount: number;
    characterCount: number;
    propCount: number;
    stageCount: number;
    flagCount: number;            // Số lượng issues từ Step 5
  };
}
```

## Flow
```
1. Validate input parameters

2. Step 1: Story Draft
   Gọi generate-story-draft (truyền storyIdea, attributes, language)
   → Tạo Story + Snapshot trong DB
   → Lưu: docs[], characters[], props[], stages[], spreads[] (với textboxes[])

3. Nếu không skipArtDirection:

   Step 2: Spread Visual Plan (Art Director - Phase 1)
   Gọi generate-spread-visual-plan (truyền storyId, snapshotId)
   → Update: characters[].visual_description, props[], stages[]
   → Update: spreads[].images[] (basic, chưa có geometry)

   Step 3: Text Refinement (Word Smith)
   Gọi generate-text-refinement (truyền storyId, snapshotId)
   → Update: spreads[].textboxes[].language[].text (refined)

   Step 4: Spread Composition (Art Director - Phase 2)
   Gọi generate-spread-composition (truyền storyId, snapshotId)
   → Update: spreads[].images[].geometry
   → Update: spreads[].images[].visual_description (thêm text zone notes)
   → Update: spreads[].images[].text_zone, composition_notes
   → Update: spreads[].textboxes[].language[].geometry

4. Nếu không skipQualityCheck:
   Step 5: Quality Check
   Gọi generate-quality-check (truyền storyId, snapshotId)
   → Lưu: flags[] (nếu có issues)

5. Return result
```

## Step Dependencies
```
Step 1 (Story Draft)
    ↓
Step 2 (Visual Plan) ─── phụ thuộc Step 1 (cần characters, props, stages, spreads)
    ↓
Step 3 (Text Refinement) ─── có thể chạy song song với Step 2, nhưng sequential cho đơn giản
    ↓
Step 4 (Composition) ─── phụ thuộc cả Step 2 (cần images[]) và Step 3 (cần refined text)
    ↓
Step 5 (Quality Check) ─── phụ thuộc tất cả steps trước
```

## Error Handling
- Nếu Step 1 fail → Return error, không tạo story
- Nếu Step 2 fail → Return partial result, vẫn giữ Step 1 data
- Nếu Step 3 fail → Return partial result, vẫn giữ Step 1 + 2 data
- Nếu Step 4 fail → Return partial result, vẫn giữ Step 1 + 2 + 3 data
- Nếu Step 5 fail → Return partial result, vẫn giữ tất cả data trước đó
