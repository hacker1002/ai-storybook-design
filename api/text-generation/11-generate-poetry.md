# generate-poetry

## Description
**Word Smith Agent** - Chuyển đổi nội dung văn xuôi thành thơ cho spread hoặc toàn bộ truyện. Hỗ trợ nhiều thể loại thơ Việt Nam và quốc tế phù hợp với trẻ em.

## DB Schema Dependencies

### Tables Referenced
- `stories`: Truy vấn title, target_audience, original_language
- `snapshots`: Đọc docs[], spreads[] để lấy nội dung text cần chuyển thành thơ

### Fields Used
- `stories.id` (UUID) - Primary key
- `stories.title` (VARCHAR) - Tiêu đề truyện
- `stories.target_audience` (SMALLINT) - Nhóm tuổi: 0=preschool, 1=primary, 2=tweens
- `stories.original_language` (VARCHAR) - Ngôn ngữ gốc (vi, en)
- `snapshots.docs[]` (JSONB) - Manuscript context
- `snapshots.spreads[].manuscript` (TEXT) - Cốt truyện spread
- `snapshots.spreads[].textboxes[].language[].text` (TEXT) - Nội dung textbox

## Parameters
```typescript
interface GeneratePoetryParams {
  storyId: string;
  snapshotId: string;
  scope: 'spread' | 'story';
  spreadNumber?: number;           // Required when scope = 'spread'
  language?: string;               // Fallback: story.original_language
  poetryType: PoetryType;
  customInstructions?: string;
}

type PoetryType =
  // Vietnamese
  | 'luc_bat'              // Lục bát (6-8)
  | 'tu_tuyet'             // Tứ tuyệt (4 câu, 7 chữ)
  // International
  | 'haiku'                // 5-7-5 syllables
  | 'quatrain'             // 4-line, ABAB/AABB
  | 'couplet'              // 2-line rhyming pairs
  | 'free_verse'           // No fixed structure
  // Children-specific
  | 'nursery_rhyme'        // Simple rhyming verse
  | 'counting_verse'       // Number-based rhythm
  | 'action_verse';        // Interactive/movement
```

## Result
```typescript
interface GeneratePoetryResult {
  success: boolean;
  poetry: PoetryOutput;
  summary: PoetrySummary;
}

interface PoetryOutput {
  scope: 'spread' | 'story';
  spreadNumber?: number;
  poetryType: PoetryType;
  language: string;
  items: PoetryItem[];
}

interface PoetryItem {
  spreadNumber: number;
  sourceText: string;
  poem: string;
  lines: string[];
  rhymeScheme?: string;     // e.g., "ABAB", "AABB"
  syllableCounts?: number[];
}

interface PoetrySummary {
  totalItems: number;
  totalLines: number;
}
```

## Prompt

> **DB Template Names:**
> - System: `WORD_SMITH_SYSTEM`
> - User: `POETRY_GENERATION_USER_TEMPLATE`

### System Prompt
*(Reuse WORD_SMITH_SYSTEM)*

### User Prompt Template
```
Transform the following content into poetry for a children's picture book:

## STORY CONTEXT
- Title: {%title%}
- Target Audience: {%audience%}
- Language: {%language%}

## POETRY PARAMETERS
- Scope: {%scope%}
- Poetry Type: {%poetry_type%}
- Poetry Type Description: {%poetry_type_description%}

## SOURCE CONTENT
{%source_content%}

{%custom_instructions%}

---

## OUTPUT FORMAT
Return JSON:
{
  "items": [{
    "spreadNumber": 1,
    "sourceText": "Original prose text...",
    "poem": "Poem text with line breaks...",
    "lines": ["Line 1", "Line 2", ...],
    "rhymeScheme": "AABB",
    "syllableCounts": [6, 8, 6, 8]
  }],
  "summary": {
    "totalItems": 1,
    "totalLines": 4
  }
}

**Rules:**
- Maintain emotional tone of source text
- Age-appropriate vocabulary ALWAYS
- Poetry must make sense independently
- Include rhymeScheme and syllableCounts for structured types
- For Vietnamese poetry, ensure proper tonal patterns
```

## Poetry Type Descriptions
```typescript
const POETRY_TYPE_DESCRIPTIONS: Record<PoetryType, string> = {
  'luc_bat': 'Lục bát - câu 6 chữ xen kẽ câu 8 chữ, vần bằng',
  'tu_tuyet': 'Tứ tuyệt - 4 câu 7 chữ, Khai-Thừa-Chuyển-Hợp',
  'haiku': 'Haiku - 3 dòng 5-7-5 âm tiết, hình ảnh thiên nhiên',
  'quatrain': 'Quatrain - 4 dòng với rhyme scheme (ABAB/AABB)',
  'couplet': 'Couplet - các cặp 2 dòng gieo vần (AA BB CC)',
  'free_verse': 'Thơ tự do - không cấu trúc cố định',
  'nursery_rhyme': 'Đồng dao - đơn giản, lặp lại, dễ nhớ',
  'counting_verse': 'Thơ đếm số - kết hợp số và nhịp điệu',
  'action_verse': 'Thơ hành động - có gợi ý động tác'
};
```

## Flow
```
1. Validate input parameters (storyId, snapshotId, poetryType)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "WORD_SMITH_SYSTEM"
   - Query `prompt_templates` với name = "POETRY_GENERATION_USER_TEMPLATE"
3. Lấy story metadata (title, target_audience, original_language)
4. Determine language: params.language || story.original_language
5. Lấy source content based on scope:
   - scope = 'spread': Lấy spread[spreadNumber].manuscript + textboxes
   - scope = 'story': Lấy docs[manuscript] + tất cả spreads
6. Map target_audience enum to text
7. Render user prompt template với variables
8. Call LLM với system prompt và rendered user prompt
9. Parse JSON response
10. Return result với summary
```

## Error Handling
- Nếu story/snapshot không tồn tại → Return error
- Nếu scope = 'spread' nhưng spreadNumber không có → Return validation error
- Nếu không có text content trong scope → Return error
- Nếu poetryType không hợp lệ → Return validation error với danh sách types
- Nếu LLM response không parse được → Log error, return error

## Usage Examples

### Example 1: Tạo lục bát cho 1 spread
```typescript
const result = await generatePoetry({
  storyId: 'uuid-1',
  snapshotId: 'uuid-2',
  scope: 'spread',
  spreadNumber: 3,
  language: 'vi',
  poetryType: 'luc_bat'
});
```

### Example 2: Tạo haiku cho toàn bộ truyện
```typescript
const result = await generatePoetry({
  storyId: 'uuid-1',
  snapshotId: 'uuid-2',
  scope: 'story',
  language: 'en',
  poetryType: 'haiku'
});
```

### Example 3: Tạo đồng dao với custom instructions
```typescript
const result = await generatePoetry({
  storyId: 'uuid-1',
  snapshotId: 'uuid-2',
  scope: 'spread',
  spreadNumber: 1,
  poetryType: 'nursery_rhyme',
  customInstructions: 'Sử dụng onomatopoeia mèo kêu (meo meo)'
});
```

## Notes
- Function hoạt động độc lập, không nằm trong manuscript pipeline
- Có thể gọi sau khi story đã có nội dung (sau Step 3 - text refinement)
- Poetry output có thể pass cho `generate-translation` nhưng nên re-generate thay vì translate để preserve rhyme
- syllableCounts giúp xác định tempo cho text-to-speech (future feature)
