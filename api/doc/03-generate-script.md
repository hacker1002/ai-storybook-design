# generate-script

## Description
**Writing Script Agent** - Chuyển draft thành manuscript script hoàn chỉnh: phân trang theo chuẩn 32 trang, text trau chuốt cho mỗi trang/spread, ghi chú minh họa cho illustrator, và page turn hooks.

**Entry point:** POST `/api/doc/generate-script`

## DB Schema Dependencies

### Tables Referenced
- `prompt_templates`: Lấy WRITING_SCRIPT_SKILL, GENERATE_SCRIPT_SYSTEM prompts + model
- `eras`: Truy vấn era name, description (nếu chỉ truyền id)
- `locations`: Truy vấn location name, description (nếu chỉ truyền id)

## Parameters

```typescript
interface GenerateScriptRequest {
  // Required - Selected draft from previous step
  draft: DraftItem;

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

// From generate-draft result
interface DraftItem {
  title: string;
  voice_style: string;
  who: string;
  what: string;
  when: string;
  where: string;
  why: string;
  how: string;
  characters: CharacterProfile[];
  full_narrative: string;
}
```

## Result

```typescript
interface GenerateScriptResult {
  success: boolean;
  data: ScriptItem;                         // Single script object
  meta?: {
    processingTime?: number;
    tokenUsage?: number;
  };
}

interface ScriptItem {
  title: string;
  dedication: string | null;

  pages: PageItem[];

  word_count: number;                       // Tổng số từ text
  reading_time_minutes: number;
}

interface PageItem {
  page_number: string;                      // "1", "2-3", "30-31"
  text: string;                             // Lời văn chính xác trên trang
  illustration_notes: string;               // Ghi chú cho illustrator
  page_turn_hook: string | null;            // Yếu tố tạo suspense
}
```

## Prompt Templates

| Type | Name | Description |
|------|------|-------------|
| System | `GENERATE_SCRIPT_SYSTEM` | User request + output format |
| Skill | `WRITING_SCRIPT_SKILL` | Role + knowledge (32-page structure, page turns, illustration notes) |
| Skill | `PACING_SKILL` | Pacing, rhythm, word count control |

**Skill Reference:** `@@Skill sử dụng: WRITING_SCRIPT_SKILL, PACING_SKILL`

## Flow

```
1. Validate input
   - draft: required object with all DraftItem fields
   - prompt: required string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre

2. Build prompts (Prompt Build Logic)
   a. Fetch system prompt: GENERATE_SCRIPT_SYSTEM
   b. Extract skill names: parse "@@Skill sử dụng: ..." → ['WRITING_SCRIPT_SKILL', 'PACING_SKILL']
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
   - Replace {%request.draft%} with JSON.stringify(draft)
   - Replace {%request.prompt%} with user input

5. Call LLM
   - model: from prompt_templates.model
   - thinking_level: high, temperature: 2, responseMimeType: text/plain

6. Parse & validate response
   - JSON object, required fields: title, pages, word_count, reading_time_minutes
   - Validate pages array structure

7. Return { success, data: ScriptItem, meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Parse Error**: If JSON invalid, retry once
- **Incomplete Pages**: If missing required page structure, return error

## References

- [Prompt Template System](../README.md#prompt-template-system) - Variable syntax `{%request.draft%}`, `{%request.prompt%}`, skill reference pattern
- [Enum Mappings](../README.md#enum-mappings) - TARGET_AUDIENCE_MAP, TARGET_CORE_VALUE_MAP, FORMAT_GENRE_MAP, CONTENT_GENRE_MAP

## Notes
- API is stateless - does not save script to DB
- This is the final step in the brief → draft → script flow
- Output script can be saved to snapshot.docs[] by client
- Script follows standard 32-page picture book structure
- word_count should match target audience expectations
