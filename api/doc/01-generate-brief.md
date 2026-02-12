# generate-brief

## Description
**Writing Brief Agent** - Tạo 3 brief tiềm năng từ ý tưởng truyện. Mỗi brief bao gồm: title, logline, plot_summary, main_character, story_question, theme_message, unique_angle.

**Entry point:** POST `/api/doc/generate-brief`

## DB Schema Dependencies

### Tables Referenced
- `prompt_templates`: Lấy WRITING_BRIEF_SKILL, GENERATE_BRIEF_SYSTEM prompts + model
- `eras`: Truy vấn era name, description (nếu chỉ truyền id)
- `locations`: Truy vấn location name, description (nếu chỉ truyền id)

## Parameters

```typescript
interface Attachment {
  filename: string;                          // Tên file gốc
  mimeType: string;                          // MIME type (image/png, application/pdf, etc.)
  base64Data: string;                        // Base64 encoded content (max 10MB per file)
}

interface GenerateBriefRequest {
  // Required - User input
  prompt: string;                           // Ý tưởng truyện từ tác giả

  // Optional - Current content (for regeneration/refinement)
  currentBrief?: string;                    // Brief hiện tại (nếu đang chỉnh sửa)

  // Optional - Attachments
  attachments?: Attachment[];               // File đính kèm (reference materials, images, etc.)

  // Required - LLM Context
  llmContext: {
    // Story attributes (required)
    targetAudience: 1 | 2 | 3 | 4;          // 1: kindergarten, 2: preschool, 3: primary, 4: middle grade
    targetCoreValue: number;                 // 1-21: Core values
    formatGenre: 1 | 2 | 3 | 4 | 5 | 6;     // Narrative, Lullaby, Concept, Non-fiction, Early Reader, Wordless
    contentGenre: number;                    // 1-11: Mystery, Fantasy, etc.

    // Additional context (optional)
    era?: {
      id: string;
      name: string;
      description: string;
    };
    location?: {
      id: string;
      name: string;
      description: string;
    };
  };
}
```

## Result

```typescript
interface GenerateBriefResult {
  success: boolean;
  data: string;                             // Markdown text containing 3 briefs
  meta?: {
    processingTime?: number;                // ms
    tokenUsage?: number;
  };
}
```

### AI Output Format (Markdown)

AI trả về markdown gồm 3 đoạn, mỗi đoạn có các trường:

| Field | Type | Description |
|-------|------|-------------|
| title | string | Tiêu đề hấp dẫn, ngắn gọn |
| logline | string | 1 câu khiến tò mò ngay lập tức |
| plot_summary | string | 3-5 câu: mở đầu → xung đột → cao trào → kết thúc |
| main_character | string | Đặc điểm nổi bật phù hợp plot |
| story_question | string | Câu hỏi cốt lõi |
| theme_message | string | Thông điệp, bài học |
| unique_angle | string | Điều gì khiến truyện đặc biệt |

## Prompt Templates

| Type | Name | Description |
|------|------|-------------|
| System | `GENERATE_BRIEF_SYSTEM` | User request + output format |
| Skill | `WRITING_BRIEF_SKILL` | Role + knowledge (story question, plot types, age requirements) |

**Skill Reference:** `@@Skill sử dụng: WRITING_BRIEF_SKILL`

## Flow

```
1. Validate input
   - prompt: required, non-empty string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre
   - currentBrief: optional string
   - attachments: optional array, each item must have filename, mimeType, base64Data (max 10MB)

2. Build prompts (Prompt Build Logic)
   a. Fetch system prompt: GENERATE_BRIEF_SYSTEM
   b. Extract skill names: parse "@@Skill sử dụng: ..." → ['WRITING_BRIEF_SKILL']
   c. Fetch skill prompts from DB
   d. Combine: skillPrompts + systemPrompt

3. Enrich with context
   - IF era.id (missing name/desc): Query eras → append to prompt
   - IF location.id (missing name/desc): Query locations → append to prompt
   - Append Story Attributes:
     • targetAudience → TARGET_AUDIENCE_MAP
     • targetCoreValue → TARGET_CORE_VALUE_MAP
     • formatGenre → FORMAT_GENRE_MAP
     • contentGenre → CONTENT_GENRE_MAP

4. Render variables
   - Replace {%request.prompt%} with user input
   - Replace {%request.current_brief%} with currentBrief (empty string if null)
   - Replace {%request.attachments%} with attachments metadata (filenames, types), keep attachment sequences
   - IF attachments: Include as inlineData parts in LLM request (base64 passed directly)

5. Call LLM
   - model: from prompt_templates.model
   - thinking_level: high, temperature: 2, responseMimeType: text/plain

6. Return markdown response
   - Return raw markdown text from AI
   - Client parses markdown to extract 3 briefs

7. Return { success, data: string (markdown), meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Empty Response**: If AI returns empty/invalid markdown, retry once

## References

- [Prompt Template System](../README.md#prompt-template-system) - Variable syntax `{%request.prompt%}`, skill reference pattern
- [Enum Mappings](../README.md#enum-mappings) - TARGET_AUDIENCE_MAP, TARGET_CORE_VALUE_MAP, FORMAT_GENRE_MAP, CONTENT_GENRE_MAP

## Notes
- API is stateless - does not save briefs to DB
- AI returns markdown, client parses to extract structured briefs
- Client stores selected brief locally for next step (generate-draft)
- 3 briefs must be distinctly different in approach, tone, character
