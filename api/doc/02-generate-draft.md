# generate-draft

## Description
**Writing Draft Agent** - Triển khai brief thành 2 bản draft đầy đủ với cốt truyện hoàn chỉnh, character bible, 6W, và dialogue mẫu. User chọn 1 draft để chuyển sang bước script.

**Entry point:** POST `/api/doc/generate-draft`

## DB Schema Dependencies

### Tables Referenced
- `prompt_templates`: Lấy GENERATE_DRAFT_SKILL, GENERATE_DRAFT_SYSTEM prompts + model
- `eras`: Truy vấn era name, description (nếu chỉ truyền id)
- `locations`: Truy vấn location name, description (nếu chỉ truyền id)

## Parameters

```typescript
interface GenerateDraftRequest {
  // Required - Selected brief from previous step
  brief: BriefItem;

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

// From generate-brief result
interface BriefItem {
  title: string;
  logline: string;
  plot_summary: string;
  main_character: string;
  story_question: string;
  theme_message: string;
  unique_angle: string;
}
```

## Result

```typescript
interface GenerateDraftResult {
  success: boolean;
  data: DraftItem[];                        // Array of 2 drafts
  meta?: {
    processingTime?: number;
    tokenUsage?: number;
  };
}

interface DraftItem {
  title: string;
  voice_style: string;                      // Mô tả giọng văn

  // 6W
  who: string;
  what: string;
  when: string;
  where: string;
  why: string;
  how: string;

  characters: CharacterProfile[];
  full_narrative: string;                   // Cốt truyện đầy đủ + dialogue mẫu
}

interface CharacterProfile {
  name: string;
  role: string;                             // main character, supporting, antagonist
  appearance: string;
  personality: string;
  want: string;
  arc: string;
}
```

## Prompt Templates

| Type | Name | Description |
|------|------|-------------|
| System | `GENERATE_DRAFT_SYSTEM` | User request + output format |
| Skill | `WRITING_DRAFT_SKILL` | Role + knowledge (3-act structure, 6W, character bible, voice) |

**Skill Reference:** `@@Skill sử dụng: WRITING_DRAFT_SKILL`

## Flow

```
1. Validate input
   - brief: required object with all BriefItem fields
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
   - Replace {%request.brief%} with JSON.stringify(brief)
   - Replace {%request.prompt%} with user input

5. Call LLM
   - model: from prompt_templates.model
   - thinking_level: high, temperature: 2, responseMimeType: text/plain

6. Parse & validate response
   - JSON array, length = 2, validate characters array structure

7. Return { success, data: DraftItem[], meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Parse Error**: If JSON invalid, retry once
- **Incomplete Response**: If < 2 drafts, return partial with warning

## Notes
- API is stateless - does not save drafts to DB
- Client stores selected draft locally for next step (generate-script)
- 2 drafts must differ in voice/tone and storytelling approach
- full_narrative should include sample dialogues showing character voice
