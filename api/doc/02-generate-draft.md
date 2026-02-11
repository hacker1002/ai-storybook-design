# generate-draft

## Description
**Writing Draft Agent** - Triển khai brief thành 2 bản draft đầy đủ với cốt truyện hoàn chỉnh, character bible, 6W, và dialogue mẫu. User chọn 1 draft để chuyển sang bước script.

**Entry point:** POST `/api/doc/generate-draft`

## DB Schema Dependencies

### Tables Referenced
- `prompt_templates`: Lấy WRITING_DRAFT_SKILL, GENERATE_DRAFT_SYSTEM prompts + model
- `eras`: Truy vấn era name, description (nếu chỉ truyền id)
- `locations`: Truy vấn location name, description (nếu chỉ truyền id)

## Parameters

```typescript
interface GenerateDraftRequest {
  // Required - Selected brief from previous step (markdown string)
  brief: string;                            // Markdown text of selected brief

  // Required - User refinement
  prompt: string;                           // Ghi chú bổ sung từ tác giả

  // Required - LLM Context
  llmContext: {
    // Story attributes (required)
    targetAudience: 1 | 2 | 3 | 4;
    targetCoreValue: number;
    formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
    contentGenre: number;

    // Additional context (optional)
    era?: { id: string; name: string; description: string };
    location?: { id: string; name: string; description: string };
  };
}
```

## Result

```typescript
interface GenerateDraftResult {
  success: boolean;
  data: string;                             // Markdown text containing 2 drafts
  meta?: {
    processingTime?: number;
    tokenUsage?: number;
  };
}
```

### AI Output Format (Markdown)

AI trả về markdown gồm 2 đoạn, mỗi đoạn có các trường:

| Field | Type | Description |
|-------|------|-------------|
| title | string | Tiêu đề draft |
| voice_style | string | Mô tả giọng văn |
| who | string | Nhân vật chính là ai |
| what | string | Chuyện gì xảy ra |
| when | string | Bối cảnh thời gian |
| where | string | Bối cảnh không gian |
| why | string | Tại sao nhân vật hành động |
| how | string | Nhân vật giải quyết bằng cách nào |
| characters | array | Danh sách nhân vật (name, role, appearance, personality, want, arc) |
| full_narrative | string | Cốt truyện đầy đủ + dialogue mẫu |

## Prompt Templates

| Type | Name | Description |
|------|------|-------------|
| System | `GENERATE_DRAFT_SYSTEM` | User request + output format |
| Skill | `WRITING_DRAFT_SKILL` | Role + knowledge (3-act structure, 6W, character bible, voice) |

**Skill Reference:** `@@Skill sử dụng: WRITING_DRAFT_SKILL`

## Flow

```
1. Validate input
   - brief: required string (markdown)
   - prompt: required string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre

2. Build prompts (Prompt Build Logic)
   a. Fetch system prompt: GENERATE_DRAFT_SYSTEM
   b. Extract skill names: parse "@@Skill sử dụng: ..." → ['WRITING_DRAFT_SKILL']
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
   - Replace {%request.brief%} with brief (markdown string)
   - Replace {%request.prompt%} with user input

5. Call LLM
   - model: from prompt_templates.model
   - thinking_level: high, temperature: 2, responseMimeType: text/plain

6. Return markdown response
   - Return raw markdown text from AI
   - Client parses markdown to extract 2 drafts

7. Return { success, data: string (markdown), meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Empty Response**: If AI returns empty/invalid markdown, retry once

## References

- [Prompt Template System](../README.md#prompt-template-system) - Variable syntax `{%request.brief%}`, `{%request.prompt%}`, skill reference pattern
- [Enum Mappings](../README.md#enum-mappings) - TARGET_AUDIENCE_MAP, TARGET_CORE_VALUE_MAP, FORMAT_GENRE_MAP, CONTENT_GENRE_MAP

## Notes
- API is stateless - does not save drafts to DB
- AI returns markdown, client parses to extract structured drafts
- Client stores selected draft locally for next step (generate-script)
- 2 drafts must differ in voice/tone and storytelling approach
- full_narrative should include sample dialogues showing character voice
