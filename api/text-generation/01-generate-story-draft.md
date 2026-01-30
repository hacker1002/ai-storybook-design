# generate-story-draft

## Description
**Story Teller Agent** - Phân tích ý tưởng và tạo khung truyện ban đầu bao gồm: docs[], characters[], props[], stages[], spreads[]. Function được gọi từ background job (Step 1).

## DB Schema Dependencies

### Tables Referenced
- `stories`: Đọc story record đã tạo ở Phase 1
- `snapshots`: Tạo snapshot đầu tiên
- `eras`: Truy vấn era description
- `locations`: Truy vấn location description và list cho stages
- `asset_categories`: Truy vấn list cho characters/props

### Fields Used
- `stories.id`, `stories.target_audience`, `stories.target_core_value`, `stories.format_genre`, `stories.content_genre`, `stories.writing_style`, `stories.era_id`, `stories.location_id`, `stories.original_language`
- `background_jobs.params` - Đọc storyIdea, storyIdeaExplanation
- `snapshots.docs[]`, `snapshots.characters[]`, `snapshots.props[]`, `snapshots.stages[]`, `snapshots.spreads[]`

## Parameters
```typescript
// Story đã được tạo từ bước trước, API nhận storyId + storyParams
interface GenerateStoryDraftInput {
  storyId: string;                      // Story đã tồn tại từ generate-manuscript
  storyParams: {
    storyIdea: string;
    storyIdeaExplanation: string;
  };
}

// Các settings khác được đọc từ DB bảng stories:
// - target_audience, target_core_value, format_genre, content_genre
// - writing_style, era_id, location_id, original_language
```

### Target Audience Mapping
| Value | Name | Age Range | Reading Level | Spreads | Words/Spread |
|-------|------|-----------|---------------|---------|--------------|
| 1 | Kindergarten | 2-3 tuổi | Từ rất đơn giản, câu 3-5 từ | 8-10 | 10-20 |
| 2 | Preschool | 4-5 tuổi | Từ đơn giản, câu ngắn | 10-12 | 20-40 |
| 3 | Primary | 6-8 tuổi | Câu phức, từ vựng đa dạng | 12-16 | 40-60 |
| 4 | Middle Grade | 9+ tuổi | Nội dung phức tạp, đa chủ đề | 16-20 | 50-80 |

### Format Genre Mapping
| Value | Name (EN) | Name (VI) |
|-------|-----------|-----------|
| 1 | Narrative Picture Books | Sách tranh kể chuyện |
| 2 | Lullaby/Bedtime Books | Sách ru ngủ |
| 3 | Concept Books | Sách khái niệm |
| 4 | Non-fiction Picture Books | Sách tranh phi hư cấu |
| 5 | Early Reader | Sách đọc sớm |
| 6 | Wordless Picture Books | Sách tranh không chữ |

### Content Genre Mapping
| Value | Name (EN) | Name (VI) |
|-------|-----------|-----------|
| 1 | Mystery | Bí ẩn |
| 2 | Fantasy | Kỳ ảo |
| 3 | Realistic Fiction | Hiện thực |
| 4 | Historical Fiction | Lịch sử hư cấu |
| 5 | Science Fiction | Khoa học viễn tưởng |
| 6 | Folklore/Fairy Tales | Dân gian/Cổ tích |
| 7 | Humor | Hài hước |
| 8 | Horror/Scary | Kinh dị |
| 9 | Biography | Tiểu sử |
| 10 | Informational | Thông tin |
| 11 | Memoir | Hồi ký |

### Writing Style Mapping
| Value | Name (EN) | Name (VI) | Description |
|-------|-----------|-----------|-------------|
| 1 | Narrative | Văn xuôi | Kể chuyện truyền thống |
| 2 | Rhyming | Thơ/Vần điệu | Có vần, nhịp điệu |
| 3 | Humorous Fiction | Hài hước | Phong cách hài, vui nhộn |

## Result
```typescript
interface GenerateStoryDraftResult {
  success: boolean;
  snapshotId: string;
  data: LLMOutput;
}

interface LLMOutput {
  metadata: {
    title: string;
    summary: string;
  };
  docs: DocItem[];
  characters: CharacterItem[];
  props: PropItem[];
  stages: StageItem[];
  spreads: SpreadItem[];
}

interface DocItem {
  type: 0 | 1 | 2;                      // 0=system r/w, 1=user r/w, 2=user ro
  title: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  content: string;
}

interface CharacterItem {
  order: number;
  key: string;
  name: string;
  basic_info: {
    description: string;
    gender: string;
    age: string;
    category_id: string;
    role: string;
  };
  personality: {
    core_essence: string;
    flaws: string;
    emotions: string;
    reactions: string;
    desires: string;
    likes: string;
    fears: string;
    contradictions: string;
  };
  variants: Array<{
    name: string;
    key: string;
    type: 0 | 1;                        // 0=default, 1=variant
    appearance: {
      height: number;
      hair: string;
      eyes: string;
      face: string;
      build: string;
    };
  }>;
}

interface PropItem {
  order: number;
  key: string;
  name: string;
  category_id: string;
  type: "narrative" | "anchor";
  states: Array<{
    name: string;
    key: string;
    type: 0 | 1;                        // 0=default, 1=state
  }>;
}

interface StageItem {
  order: number;
  key: string;
  name: string;
  location_id: string;
  settings: Array<{
    name: string;
    key: string;
    type: 0 | 1;                        // 0=default, 1=setting
  }>;
}

interface SpreadItem {
  type: 0 | 1;                          // 1=dps, 0=non-dps
  number: number;
  left_page: { number: number; type: string };
  right_page: { number: number; type: string };
  manuscript: string;
  textboxes: Array<{
    order: number;
    language: Array<{ text: string }>;
  }>;
}
```

## Prompt

> **DB Template Names:**
> - System: `STORY_TELLER_SYSTEM`
> - User: `STORY_DRAFT_USER_TEMPLATE`

### System Prompt
```
You are an expert Story Teller specializing in children's picture books. Your role is to deeply analyze story ideas and create comprehensive story frameworks.

Your expertise includes:
- Child psychology and age-appropriate storytelling
- Character development with multi-dimensional personalities
- Story structure (three-act, hero's journey, etc.)
- Literary devices and symbolic imagery
- Moral lessons that resonate without being preachy

You output structured JSON that can be directly used by the system.
```

### User Prompt Template
```
Analyze the following story idea and create a complete story framework:

## STORY IDEA
{%story_idea%}

## STORY IDEA EXPLANATION
{%story_idea_explanation%}

## STORY SETTINGS

### General Settings
- Target Audience: {%target_audience%}
- Target Core Value: {%target_core_value%}

### Creative Settings
- Format Genre: {%format_genre%}
- Content Genre: {%content_genre%}
- Writing Style: {%writing_style%}
- Era: {%era_name%} - {%era_description%}
- Location (id: {%location_id%}): {%location_name%} - {%location_description%}

### Format
- Language: {%language%}
- Number of Spreads: {%spreads%}
- Words per Spread: {%words_per_spread%}

## RESOURCES TABLE

### asset_categories
{%categories_text%}

---

## OUTPUT FORMAT

Return a JSON object with the following structure:

{
  "metadata": { "title": "...", "summary": "..." },
  "docs": [...],
  "characters": [...],
  "props": [...],
  "stages": [...],
  "spreads": [...]
}

---

## DETAILED REQUIREMENTS

### 1. metadata
- **title**: Tiêu đề truyện
- **summary**: Tóm tắt 2-3 câu về nội dung và bài học

### 2. docs[] - 4 documents
{
  "type": 0,  // 0=system r/w, 1=user r/w, 2=user ro
  "title": "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson",
  "content": "..." // Markdown content
}

**Document 1: manuscript** - Cốt truyện hoàn chỉnh
**Document 2: story_structure** - Three-act, conflicts, pacing
**Document 3: artistic_imagery** - Metaphors, symbols, visual motifs
**Document 4: moral_lesson** - Main theme, emotional journey

### 3. characters[]
{
  "order": 1,
  "key": "miu_cat",
  "name": "Miu",
  "basic_info": {
    "description": "...",
    "gender": "male",
    "age": "3 tuổi",
    "category_id": "uuid-of-category",
    "role": "main character"
  },
  "personality": {
    "core_essence": "...",
    "flaws": "...",
    "emotions": "...",
    "reactions": "...",
    "desires": "...",
    "likes": "...",
    "fears": "...",
    "contradictions": "..."
  },
  "variants": [{
    "name": "Default",
    "key": "default",
    "type": 0,  // 0=default, 1=variant
    "appearance": {
      "height": 30,
      "hair": "...",
      "eyes": "...",
      "face": "...",
      "build": "..."
    }
  }]
}

### 4. props[]
{
  "order": 1,
  "key": "red_bow",
  "name": "Chiếc nơ đỏ",
  "category_id": "uuid-of-category",
  "type": "narrative | anchor",
  "states": [{
    "name": "Default",
    "key": "default",
    "type": 0  // 0=default, 1=state
  }]
}

### 5. stages[]
{
  "order": 1,
  "key": "forest_1",
  "name": "Khu rừng bí ẩn",
  "location_id": "uuid-of-location",
  "settings": [{
    "name": "Default",
    "key": "default",
    "type": 0  // 0=default, 1=setting
  }]
}

### 6. spreads[]
{
  "type": 1,  // 1=dps (double page spread), 0=non-dps
  "number": 1,
  "left_page": { "number": 1, "type": "front_matter | story | back_matter" },
  "right_page": { "number": 2, "type": "front_matter | story | back_matter" },
  "manuscript": "trang 1: <cốt truyện trang 1>, trang 2: <cốt truyện trang 2>",
  "textboxes": [{
    "order": 1,
    "language": [{ "text": "Nội dung văn bản..." }]
  }]
}

---

## IMPORTANT RULES
1. All @key must be unique, lowercase, using underscores
2. Text content must match target audience reading level
3. Include emotional progression across spreads
4. Clear beginning, middle, end with meaningful moral
5. KHÔNG tạo visual_description ở step này
6. Output phải là valid JSON
7. Characters, props, stages PHẢI có `order` field (bắt đầu từ 1)
8. Character `appearance` PHẢI nằm trong `variants[]`, không ở root level
9. Props PHẢI có `states[]` với ít nhất 1 default state
10. Stages PHẢI có `settings[]` với ít nhất 1 default setting
11. Spreads PHẢI có `type` field (1=dps, 0=non-dps)
```

## Flow
```
1. Validate storyId và storyParams

2. Đọc Story record từ DB (storyId)
   - Lấy: target_audience, target_core_value, format_genre, content_genre
   - Lấy: writing_style, era_id, location_id, original_language

3. Lấy prompt templates từ DB:
   - STORY_TELLER_SYSTEM → system prompt + model
   - STORY_DRAFT_USER_TEMPLATE → user prompt template

4. Lấy reference data từ DB:
   - eras (id = story.era_id) → era_name, era_description
   - locations (id = story.location_id) → location_name, location_description

5. Lấy resource lists từ DB:
   - asset_categories → categories_text
   - locations → locations_text

6. Map input values:
   - targetAudience → target_audience_text, spreads, words_per_spread (theo mapping table)
   - targetCoreValue → target_core_value_text (1-21 mapping)
   - formatGenre → format_genre_text
   - contentGenre → content_genre_text
   - writingStyle → writing_style_text

7. Render user prompt template

8. Call LLM

9. Parse JSON response

10. Tạo Snapshot record với:
    - docs[], characters[], props[], stages[], spreads[]
    - story_id = storyId

11. Update Story record:
    - title, description từ LLM output
    - current_version = snapshot.id

12. Return { snapshotId, data }
```

## Error Handling
- LLM response không parse được → Retry 1 lần
- Vẫn fail → Return error, không tạo snapshot
- Missing required fields → Return validation error
