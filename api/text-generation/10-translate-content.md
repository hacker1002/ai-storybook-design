# generate-translation

## Description
**Translator Agent** - Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả.

## DB Schema Dependencies

### Tables Used
- `books`: id, title, original_language, target_audience, format_genre, content_genre
- `snapshots`: characters[], props[], stages[] (để lấy name mappings nếu cần)

### Fields Reference
- `books.id` (UUID) - Primary key
- `books.title` (VARCHAR) - Tiêu đề truyện, dùng làm context
- `books.original_language` (VARCHAR) - Ngôn ngữ gốc (vi, en), fallback cho sourceLanguage
- `books.target_audience` (SMALLINT) - Nhóm tuổi: 1=kindergarten (2-3), 2=preschool (4-5), 3=primary (6-8), 4=middle grade (9+)
- `books.format_genre` (SMALLINT) - 1: Narrative, 2: Lullaby, 3: Concept, 4: Non-fiction, 5: Early Reader, 6: Wordless
- `books.content_genre` (SMALLINT) - 1: Mystery, 2: Fantasy, 3: Realistic, 4: Historical, 5: Sci-Fi, 6: Folklore, 7: Humor, 8: Horror, 9: Biography, 10: Informational, 11: Memoir

## Parameters
```typescript
interface GenerateTranslationParams {
  bookId: string;                 // ID của book trong DB
  snapshotId?: string;            // ID snapshot (optional, để lấy character names)
  content: string | TranslationItem[];  // Nội dung cần dịch
  sourceLanguage?: string;        // Ngôn ngữ nguồn - fallback: book.original_language
  targetLanguage: string;         // Ngôn ngữ đích (required)
  contentType: ContentType;       // Loại nội dung
  characterNameMappings?: Record<string, string>;  // Name mappings (optional)
  preserveFormatting?: boolean;   // Giữ nguyên markdown/formatting (default: true)
}

interface TranslationItem {
  id: string;
  text: string;
}

type ContentType =
  | "manuscript"           // Cốt truyện
  | "textbox"              // Nội dung text trong spread
  | "visual_description"   // Mô tả hình ảnh
  | "metadata";            // Title, description, etc.
```

## Result
```typescript
interface GenerateTranslationResult {
  success: boolean;
  translations: TranslationOutput[];
  languagePair: string;    // e.g., "vi→en"
}

interface TranslationOutput {
  id?: string;             // ID nếu input là array
  originalText: string;
  translatedText: string;
}
```

## Prompt

> **DB Template Names:**
> - System: `TRANSLATOR_SYSTEM`
> - User: `TRANSLATOR_USER_TEMPLATE`

### System Prompt (TRANSLATOR_SYSTEM)
```
You are a professional translator specializing in children's literature.

Core Responsibilities:
- Maintain reading level appropriate for target age group
- Preserve emotional tone and storytelling rhythm
- Adapt cultural references when necessary
- Keep character names as specified in mappings (or unchanged if not specified)
- Preserve all markdown formatting and structure

Strict Rules:
- NEVER translate @key references (e.g., @miu_cat, @forest_1)
- Preserve line breaks and paragraph structure
- Keep onomatopoeia culturally appropriate for target language
- Maintain similar sentence length ratio as source
- Use vocabulary appropriate for the target audience age

Quality Standards:
- Natural, fluent translation (not word-for-word)
- Consistent terminology throughout
- Age-appropriate vocabulary and complexity
```

### User Prompt Template (TRANSLATOR_USER_TEMPLATE)
```
Translate the following content from {%source_language%} to {%target_language%}:

## Story Context
- Title: {%title%}
- Target Audience: {%target_audience%}
- Format Genre: {%format_genre%}
- Content Genre: {%content_genre%}

## Character Name Mappings
{%character_name_mappings%}

## Content Type
{%content_type%}

## Content to Translate
{%content%}

## Requirements
- Preserve formatting: {%preserve_formatting%}
- Do NOT translate @key references
- Maintain emotional tone
- Use vocabulary appropriate for {%target_audience%}

---

Respond in JSON format:
{
  "translations": [
    {
      "id": "..." (if applicable),
      "originalText": "...",
      "translatedText": "..."
    }
  ]
}
```

## Flow
```
1. Validate input parameters (bookId, content, targetLanguage, contentType)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "TRANSLATOR_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "TRANSLATOR_USER_TEMPLATE" → user prompt
3. Lấy book info từ DB:
   - SELECT title, original_language, target_audience, format_genre, content_genre
   - FROM books WHERE id = bookId
4. Determine sourceLanguage: params.sourceLanguage || book.original_language
5. Format character_name_mappings:
   - Nếu có params.characterNameMappings → format thành text
   - Nếu không → "No specific mappings (keep names unchanged)"
6. Format content:
   - Nếu string → wrap trong array
   - Nếu array → giữ nguyên
7. Map enum values to text:
   - target_audience: mapTargetAudienceToText(book.target_audience)
   - format_genre: mapFormatGenreToText(book.format_genre)
   - content_genre: mapContentGenreToText(book.content_genre)
8. Render user prompt template với variables
9. Call LLM với system prompt và rendered user prompt
10. Parse JSON response
11. Return result với languagePair
```

## Content Type Guidelines

| Content Type | Handling Notes |
|--------------|----------------|
| `manuscript` | Full story text, preserve narrative flow |
| `textbox` | Short phrases, maintain rhythm for read-aloud |
| `visual_description` | Technical descriptions, keep @key references |
| `metadata` | Titles, descriptions, keep brief and impactful |

## Error Handling
- Nếu book không tồn tại → Return error
- Nếu targetLanguage = sourceLanguage → Return original content unchanged
- Nếu content rỗng → Return empty translations array
- Nếu LLM không parse được → Log error, return partial results nếu có

## Usage Examples

### Example 1: Translate manuscript
```typescript
const result = await generateTranslation({
  bookId: 'uuid-1',
  content: 'Ngày xửa ngày xưa, có một chú mèo tên là @miu_cat...',
  targetLanguage: 'en',
  contentType: 'manuscript'
});
// Result: "Once upon a time, there was a cat named @miu_cat..."
```

### Example 2: Translate multiple textboxes
```typescript
const result = await generateTranslation({
  bookId: 'uuid-1',
  content: [
    { id: 'tb-1', text: 'Miu vui vẻ chạy nhảy.' },
    { id: 'tb-2', text: '@miu_cat thích chơi với @red_bow.' }
  ],
  targetLanguage: 'en',
  contentType: 'textbox',
  characterNameMappings: { 'Miu': 'Miu' }
});
```

### Example 3: Translate with custom name mappings
```typescript
const result = await generateTranslation({
  bookId: 'uuid-1',
  content: 'Bé An và Miu đang chơi trong vườn.',
  targetLanguage: 'en',
  contentType: 'textbox',
  characterNameMappings: {
    'Bé An': 'Little An',
    'Miu': 'Miu'
  }
});
// Result: "Little An and Miu are playing in the garden."
```

## Notes
- For poetry/verse content, consider using `generate-poetry` instead of translation (may not preserve rhyme schemes)
- @key references: see CLAUDE.md for format details
