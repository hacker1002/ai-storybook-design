# translate-content

## Description
Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả.

## DB Schema Dependencies

### Story Table Fields
- `id` (UUID) - Primary key, story ID
- `title` (VARCHAR) - Tiêu đề truyện, được dùng làm context khi dịch
- `original_language` (VARCHAR) - Ngôn ngữ gốc (vi, en), dùng làm fallback cho sourceLanguage
- `target_audience` (SMALLINT) - Nhóm tuổi mục tiêu: preschool (2-5), primary (6-8), (9-10)
- `genre` (SMALLINT) - Thể loại: 1=fantasy, 2=scifi, 3=mystery, 4=romance, 5=horror

### Related Tables
- `snapshots` - Chứa characters[], props[], stages[] để lấy thông tin character name mappings nếu cần

## Parameters
```typescript
interface TranslateContentParams {
  storyId: string;                // ID của story trong DB
  content: string | Array<{ id: string; text: string }>;  // Nội dung cần dịch
  sourceLanguage?: string;        // Ngôn ngữ nguồn - nếu không truyền, lấy từ story.original_language
  targetLanguage: string;         // Ngôn ngữ đích (bắt buộc)
  contentType: "manuscript" | "textbox" | "visual_description" | "metadata";
  context?: {
    characterNames?: Record<string, string>;  // Name mappings (optional)
  };
  preserveFormatting?: boolean;   // Giữ nguyên markdown/formatting (default: true)
}
```

## Result
```typescript
interface TranslateContentResult {
  translations: Array<{
    id?: string;              // ID nếu input là array
    originalText: string;
    translatedText: string;
  }>;
  languagePair: string;          // e.g., "vi→en"
}
```

## Prompt

> **DB Template Names:**
> - System: `TRANSLATOR_SYSTEM`
> - User: `TRANSLATE_CONTENT_USER_TEMPLATE`

### System Prompt
```
You are a professional translator specializing in children's literature.
Translate content while:
- Maintaining the reading level appropriate for the target age group
- Preserving emotional tone and storytelling rhythm
- Adapting cultural references when necessary
- Keeping character names as specified in the name mappings (or unchanged if not specified)
- Preserving all markdown formatting and structure

Rules:
- Do NOT translate @key references
- Preserve line breaks and paragraph structure
- Keep onomatopoeia culturally appropriate for target language
- Maintain the same sentence length ratio as source
```

### User Prompt Template
```
Translate the following content from {%source_language%} to {%target_language%}:

## Context
- Story Title: {%title%}
- Target Age: {%target_audience%}
- Genre: {%genre%}

## Character Name Mappings
{ // from request context.characterNames (optional):
- original_name → translated_name
}
{%character_name_mappings%}

## Content Type
{%content_type%}

## Content to Translate
{ // single string or array of { id, text }:
}
{%content%}

## Requirements
- Preserve formatting: {%preserve_formatting%}
- Do NOT translate @key (e.g., @miu_cat, @forest_stage)
- Keep the same emotional tone
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
1. Validate input parameters (storyId, content, targetLanguage, contentType)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "TRANSLATOR_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "TRANSLATE_CONTENT_USER_TEMPLATE" → user prompt template
3. Lấy story info từ DB (original_language, target_audience, genre, title)
4. Nếu sourceLanguage không truyền, dùng story.original_language
5. Render user prompt template với variables:
   - source_language, target_language, title, target_audience, genre
   - character_name_mappings, content_type, content, preserve_formatting
6. Call LLM với system prompt và rendered user prompt
7. Return translations
```

