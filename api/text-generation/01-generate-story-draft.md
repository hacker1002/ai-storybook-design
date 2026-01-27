# generate-story-draft

## Description
**Story Teller Agent** - Phân tích ý tưởng và tạo khung truyện ban đầu bao gồm: character bible, prop bible, stage bible, phân tích nghệ thuật, và phân chia nội dung từng spread. Function này cũng tạo Story và Snapshot record trong DB.

## DB Schema Dependencies

### Tables Referenced
- `story`: Tạo record mới với metadata (title, summary, target_core_value, step, dimension, target_audience, genre, writing_style, era_id, location_id, artstyle_id, original_language)
- `snapshot`: Tạo snapshot đầu tiên chứa docs[], characters[], props[], stages[], spreads[]
- `art_styles`: Truy vấn description để đưa vào prompt
- `eras`: Truy vấn era description cho context
- `locations`: Truy vấn location description cho context và để LLM chọn location_id cho stages
- `asset_categories`: Truy vấn danh sách categories để LLM chọn category_id phù hợp cho characters/props

### Fields Used
- `story.id`, `story.title`, `story.description`, `story.step`, `story.dimension`, `story.target_audience`, `story.target_core_value`, `story.genre`, `story.writing_style`, `story.era_id`, `story.location_id`, `story.artstyle_id`, `story.original_language`
- `snapshot.id`, `snapshot.story_id`, `snapshot.docs[]`, `snapshot.characters[]`, `snapshot.props[]`, `snapshot.stages[]`, `snapshot.spreads[]`
- `art_styles.id`, `art_styles.name`, `art_styles.description`
- `eras.id`, `eras.name`, `eras.description`
- `locations.id`, `locations.name`, `locations.description`, `locations.nation`, `locations.city`
- `asset_categories.id`, `asset_categories.name`, `asset_categories.type`, `asset_categories.description`

## Parameters
```typescript
interface GenerateStoryAnalysisParams {
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
}
```

### Dimension Mapping
| Value | Size | Spreads | Words/Spread |
|-------|------|---------|--------------|
| 1 | Square (20x20cm) | 12 | 30-50 |
| 2 | A4 Landscape (29.7x21cm) | 16 | 40-60 |
| 3 | A4 Portrait (21x29.7cm) | 20 | 50-80 |

### Target Audience Mapping
| Value | Name | Age Range | Reading Level |
|-------|------|-----------|---------------|
| 1 | Preschool | 2-5 tuổi | Từ đơn giản, câu ngắn |
| 2 | Primary | 6-8 tuổi | Câu phức, từ vựng đa dạng |
| 3 | Tweens | 9-10 tuổi | Nội dung phức tạp, đa chủ đề |

### Genre Mapping
| Value | Name (EN) | Name (VI) |
|-------|-----------|-----------|
| 1 | Fantasy | Kỳ ảo |
| 2 | Sci-Fi | Khoa học viễn tưởng |
| 3 | Mystery | Bí ẩn |
| 4 | Romance | Lãng mạn |
| 5 | Horror | Kinh dị |

### Writing Style Mapping
| Value | Name (EN) | Name (VI) | Description |
|-------|-----------|-----------|-------------|
| 1 | Narrative | Văn xuôi | Kể chuyện truyền thống |
| 2 | Rhyming | Thơ/Vần điệu | Có vần, nhịp điệu |
| 3 | Humorous Fiction | Hài hước | Phong cách hài, vui nhộn |

## Result
```typescript
interface GenerateStoryAnalysisResult {
  success: boolean;
  storyId: string;                // UUID của story được tạo
  snapshotId: string;             // UUID của snapshot được tạo
  data: LLMOutput;                // Output từ LLM (đã parse)
}

// Output JSON từ LLM
interface LLMOutput {
  metadata: {
    title: string;
    summary: string;
  };
  docs: DocItem[];                // 4 documents: manuscript, structure, imagery, lesson
  characters: CharacterItem[];
  props: PropItem[];
  stages: StageItem[];
  spreads: SpreadItem[];
}

interface DocItem {
  type: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  title: string;
  content: string;                // Markdown content
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

## STORY SETTINGS

### General Settings
- Dimension: {%dimension%}              // Định dạng sách
- Target Audience: {%target_audience%}  // Đối tượng độc giả
- Target Core Value: {%target_core_value%}  // Giá trị cốt lõi (Đạo đức, Trí tuệ, Nghị lực)

### Creative Settings
- Genre: {%genre%}                      // Thể loại truyện
- Writing Style: {%writing_style%}      // Phong cách viết
- Era: {%era_name%} - {%era_description%}      // Thời đại
- Location: {%location_name%} - {%location_description%}  // Bối cảnh địa lý chính
- Art Style: {%art_style_name%} - {%art_style_description%}  // Phong cách minh họa

### Format
- Language: {%language%}
- Number of Spreads: {%spreads%}
- Words per Spread: {%words_per_spread%}

## RESOURCES TABLE

### asset_categories
{ // list from DB as below:
- uuid: id, name: "Abc", type: "human", description: ""
- ...
}
{%categories_text%}

### locations (for stages)
{ // list from DB as below:
- uuid: id, name: "Abc", description: "", nation: "", city: ""
- ...
}
{%locations_text%}

---

## OUTPUT FORMAT

Return a JSON object with the following structure:

{
  "metadata": {
    "title": "...",
    "summary": "..."
  },
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
Mỗi document là một object với cấu trúc:
{
  "type": "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson",
  "title": "...",
  "content": "..." // Markdown content
}

**Document 1: manuscript** - Cốt truyện hoàn chỉnh
- Nội dung truyện đầy đủ từ đầu đến cuối
- Viết theo dạng văn xuôi, dễ đọc
- Bao gồm lời thoại của nhân vật
- Phù hợp với reading level của target audience
- Đây là bản gốc để tham khảo khi viết text cho từng spread

**Document 2: story_structure** - Cấu trúc truyện
- Three-act structure breakdown
- Conflict types (Internal/External)
- Pacing và rhythm
- Key plot points

**Document 3: artistic_imagery** - Hình tượng nghệ thuật
- Metaphors và symbols
- Visual motifs
- Color symbolism
- Recurring imagery

**Document 4: moral_lesson** - Bài học đạo đức
- Main theme và sub-themes
- How lesson is conveyed (subtle, not preachy)
- Emotional journey của độc giả
- Age-appropriate messaging

### 3. characters[] - Danh sách nhân vật
Mỗi character là một object:
{
  "key": "miu_cat",
  "name": "Miu",
  "basic_info": {
    "description": "Mô tả ngắn về nhân vật",
    "gender": "male",
    "age": "3 tuổi (tương đương trẻ 5 tuổi)",
    "category_id": "uuid-of-category",  // FK → asset_categories
    "role": "main character"
  },
  "personality": {
    "core_essence": "Bản chất cốt lõi",
    "flaws": "Điểm yếu",
    "emotions": "Cách biểu đạt cảm xúc",
    "reactions": "Phản ứng hành vi đặc trưng",
    "desires": "Mong muốn sâu xa",
    "likes": "Sở thích",
    "fears": "Nỗi sợ",
    "contradictions": "Mâu thuẫn nội tâm"
  },
  "appearance": {
    "height": 30,
    "hair": "Mô tả lông/tóc",
    "eyes": "Mô tả mắt",
    "face": "Mô tả khuôn mặt",
    "build": "Thể hình (bao gồm trang phục nếu có)"
  }
}

### 4. props[] - Danh sách đạo cụ
{
  "key": "red_bow",
  "name": "Chiếc nơ đỏ",
  "category_id": "uuid-of-category",  // FK → asset_categories
  "type": "narrative | anchor"
}

**Giải thích:**
- **type**:
  - `narrative`: Đồ vật dẫn chuyện, tương tác với character
  - `anchor`: Đồ vật nằm trong stages, tạo sự nhất quán

### 5. stages[] - Danh sách bối cảnh
{
  "key": "forest_1",
  "name": "Khu rừng bí ẩn",
  "location_id": "uuid-of-location"  // FK → locations (địa danh cụ thể)
}

### 6. spreads[] - Danh sách các trang
{
  "number": 1,
  "left_page": { "number": 1, "type": "front_matter | story | back_matter" },
  "right_page": { "number": 2, "type": "story" },
  "manuscript": "trang 1: <cốt truyện trang 1>, trang 2: <cốt truyện trang 2>",
  "textboxes": [{
    "order": 1,
    "language": [{ "text": "Nội dung văn bản hiển thị trên trang..." }]
  }]
}

**Giải thích:**
- **number**: Số thứ tự của spread (bắt đầu từ 1)
- **left_page/right_page**: Thông tin trang trái/phải
  - `number`: Số trang
  - `type`: Loại trang (front_matter, story, back_matter)
- **manuscript**: Cốt truyện tương ứng cho spread này
  - Nếu spread có 2 trang: "trang 1: <nội dung>, trang 2: <nội dung>"
  - Nếu spread chỉ có 1 trang: không cần chia
  - Đây là reference cho Art Director khi tạo images[]
- **textboxes[]**: Nội dung text hiển thị trên trang
  - `order`: Thứ tự đọc của textbox
  - `language[]`: Mảng chứa text theo ngôn ngữ

---

## IMPORTANT RULES
1. All @key must be unique, lowercase, using underscores
2. Text content must match target audience reading level
3. Include emotional progression across spreads
4. Clear beginning, middle, end with meaningful moral
5. KHÔNG tạo visual_description ở step này (sẽ được tạo ở generate-art-direction)
6. Output phải là valid JSON
```

## Flow
```
1. Validate input parameters

2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "STORY_TELLER_SYSTEM" → system prompt + model
   - Query `prompt_templates` với name = "STORY_DRAFT_USER_TEMPLATE" → user prompt template

3. Lấy reference data từ DB:
   - Query `art_styles` với id = artstyleId → art_style_name, art_style_description
   - Query `eras` với id = eraId → era_name, era_description
   - Query `locations` với id = locationId → location_name, location_description

4. Lấy resource lists từ DB:
   - Query `asset_categories` → categories_text (cho LLM chọn category_id)
   - Query `locations` → locations_text (cho LLM chọn location_id cho stages)

5. Map input values to display text:
   - dimension → dimension_text (theo Dimension Mapping table)
   - targetAudience → target_audience_text (theo Target Audience Mapping table)
   - genre → genre_text (theo Genre Mapping table)
   - writingStyle → writing_style_text (theo Writing Style Mapping table)
   - Calculate spreads, words_per_spread dựa trên dimension

6. Render user prompt template với variables:
   - story_idea, dimension, target_audience, target_core_value
   - genre, writing_style, era_name, era_description
   - location_name, location_description, art_style_name, art_style_description
   - language, spreads, words_per_spread, categories_text, locations_text

7. Call LLM với:
   - system prompt content
   - rendered user prompt
   - model từ prompt_templates (dynamic, không hardcode)

8. Parse JSON response

9. Tạo Story record trong DB với:
   - step = 1 (manuscript)
   - dimension, target_audience, target_core_value
   - genre, writing_style, era_id, location_id, artstyle_id
   - original_language

10. Tạo Snapshot record ban đầu

11. Lưu vào snapshot:
    - docs[] (4 documents: manuscript, story_structure, artistic_imagery, moral_lesson)
    - characters[] (basic info với category_id → FK asset_categories, personality, appearance - chưa có visual_description)
    - props[] (name, key, category_id → FK asset_categories, type - chưa có visual_description)
    - stages[] (name, key, location_id → FK locations - chưa có visual_description)
    - spreads[] (number, left_page, right_page, manuscript, textboxes - chưa có images[])

12. Update story metadata (title, summary từ LLM output)

13. Return { storyId, snapshotId, data }
```
