# story-brainstorming-draft

## Description
**Phase 2: Brainstorming** - Generate và refine 4 docs (manuscript, story_structure, artistic_imagery, moral_lesson).

- **First call:** Nhận params + storyIdea từ Clarification → AI tạo 4 docs, tự chọn writingStyle, eraId, locationId
- **Subsequent calls:** User và AI brainstorm, AI update docs theo feedback

**Related Feature:** `app/ai-assistant/story-idea-brainstorming.md`

## DB Schema Dependencies

### Tables Used
| Table | Purpose |
|-------|---------|
| `ai_conversations` | Load/verify session với `step = "brainstorming"` |
| `ai_messages` | Load history, save messages |
| `eras` | AI fetch để auto-select eraId |
| `locations` | AI fetch để auto-select locationId |
| `prompt_templates` | Lấy system prompt và user template |

### Fields Used
- `ai_conversations.id`, `ai_conversations.step`
- `ai_messages.conversation_id`, `ai_messages.role`, `ai_messages.content`
- `eras.id`, `eras.name`, `eras.description`
- `locations.id`, `locations.name`, `locations.description`, `locations.nation`, `locations.city`
- `prompt_templates.name`, `prompt_templates.content`, `prompt_templates.model`

---

## Parameters

```typescript
interface BrainstormingDraftParams {
  conversationId: string;
  userMessage: string;  // First call: JSON params, subsequent: chat message
}
```

### First Call userMessage Format
```json
{
  "targetAudience": 1,
  "targetCoreValue": "Tình bạn",
  "formatGenre": 1,
  "contentGenre": 2,
  "storyIdea": "Chú mèo Miu học cách chia sẻ đồ chơi"
}
```

### Subsequent Calls
Regular chat message string, e.g., "Thêm nhân vật thỏ vào câu chuyện"

---

## Result

```typescript
interface BrainstormingDraftResult {
  conversationId: string;
  message: string;                    // AI response
  docs: StoryDoc[];                   // 4 docs (manuscript, story_structure, artistic_imagery, moral_lesson)
  params: StoryParams;                // All params (input + auto-selected)
  turnNumber: number;
}

interface StoryDoc {
  type: 0 | 1 | 2;                    // 0: system, 1: user edit, 2: user read-only
  title: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  content: string;
}

interface StoryParams {
  // From Clarification (input)
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  storyIdea: string | null;
  // AI auto-selected
  writingStyle: 1 | 2 | 3;            // 1: Narrative, 2: Rhyming, 3: Humorous
  eraId: string;                      // UUID from eras table
  locationId: string;                 // UUID from locations table
}
```

### Docs Structure
| title | type | Mô tả |
|-------|------|-------|
| `manuscript` | 0 | Draft truyện (<500 từ) |
| `story_structure` | 0 | Cấu trúc 3 hồi, conflicts, pacing |
| `artistic_imagery` | 0 | Metaphors, symbols, hình tượng nghệ thuật |
| `moral_lesson` | 0 | Bài học đạo đức, themes, emotional journey |

### Writing Style Mapping
| Value | Name | Description |
|-------|------|-------------|
| 1 | Narrative | Văn xuôi, kể chuyện |
| 2 | Rhyming | Thơ, vần điệu, lục bát |
| 3 | Humorous Fiction | Hài hước, vui nhộn |

---

## Prompt

> **DB Template Names:**
> - System: `STORY_DRAFT_GENERATOR_SYSTEM`
> - User (first call): `STORY_DRAFT_INIT_USER_TEMPLATE`
> - User (chat): `STORY_DRAFT_CHAT_USER_TEMPLATE`

### System Prompt
```
You are a Story Consultant helping develop children's picture book ideas.

Your role:
- First call: Generate 4 comprehensive docs (manuscript <500 words, story_structure, artistic_imagery, moral_lesson)
- Auto-select: writingStyle, eraId, locationId based on story context
- Subsequent calls: Help refine docs based on user feedback
- Answer questions and update docs when requested

You understand Vietnamese and English. Match the user's language.
Return structured JSON with your response and updated docs.
```

### User Prompt Template (First Call)
```
## STORY PARAMETERS
{%params_json%}

## AVAILABLE OPTIONS

### Writing Styles
- 1: Narrative (văn xuôi, kể chuyện)
- 2: Rhyming (thơ, vần điệu)
- 3: Humorous Fiction (hài hước)

### Eras
{%available_eras%}

### Locations
{%available_locations%}

---

## YOUR TASK

Generate 4 docs based on the story parameters:

1. **manuscript** (<500 từ): Draft truyện hoàn chỉnh
   - Phù hợp với targetAudience và formatGenre
   - Thể hiện targetCoreValue tự nhiên
   - Có mở đầu, phát triển, kết thúc

2. **story_structure**: Cấu trúc truyện
   - Three-act structure
   - Main conflict và resolution
   - Pacing notes

3. **artistic_imagery**: Hình tượng nghệ thuật
   - Key metaphors và symbols
   - Visual motifs
   - Color palette suggestions

4. **moral_lesson**: Bài học đạo đức
   - Core theme analysis
   - Emotional journey
   - Age-appropriate messaging

Also select:
- writingStyle: Best fit for the story (1/2/3)
- eraId: Most appropriate era UUID
- locationId: Most appropriate location UUID

---

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "message": "Brief intro to the generated story",
  "docs": [
    { "type": 0, "title": "manuscript", "content": "..." },
    { "type": 0, "title": "story_structure", "content": "..." },
    { "type": 0, "title": "artistic_imagery", "content": "..." },
    { "type": 0, "title": "moral_lesson", "content": "..." }
  ],
  "autoSelectedParams": {
    "writingStyle": 1 | 2 | 3,
    "eraId": "uuid-string",
    "locationId": "uuid-string"
  }
}
```

### User Prompt Template (Chat)
```
## CONVERSATION HISTORY
{%conversation_history%}

## CURRENT DOCS
{%current_docs%}

## CURRENT PARAMS
{%current_params%}

## USER MESSAGE
{%user_message%}

---

## YOUR TASK

1. Respond to user's message naturally
2. Update docs if user requests changes
3. Update autoSelectedParams if user requests (e.g., "đổi sang phong cách thơ vần")

---

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "message": "Your response to the user",
  "docs": [
    { "type": 0, "title": "manuscript", "content": "..." },
    { "type": 0, "title": "story_structure", "content": "..." },
    { "type": 0, "title": "artistic_imagery", "content": "..." },
    { "type": 0, "title": "moral_lesson", "content": "..." }
  ],
  "autoSelectedParams": {
    "writingStyle": 1 | 2 | 3,
    "eraId": "uuid-string",
    "locationId": "uuid-string"
  }
}
```

---

## Flow

```
1. Validate input:
   - conversationId không được rỗng
   - userMessage không được rỗng

2. Verify conversation:
   - SELECT ai_conversations WHERE id = conversationId
   - Check step = "brainstorming"

3. Load conversation history:
   - SELECT ai_messages WHERE conversation_id ORDER BY created_at
   - Determine if first call (no assistant messages with docs) or subsequent

4. Save user message:
   - INSERT ai_messages (role = "user", content = userMessage)

5. Fetch available options (for first call or param changes):
   - Query eras → format: "- {id}: {name} - {description}"
   - Query locations → format: "- {id}: {name} ({nation}, {city})"

6. Get prompt templates:
   - System: STORY_DRAFT_GENERATOR_SYSTEM
   - User: STORY_DRAFT_INIT_USER_TEMPLATE (first) or STORY_DRAFT_CHAT_USER_TEMPLATE (chat)

7. Build context:
   IF first call:
     - params_json = userMessage
     - available_eras, available_locations
   ELSE:
     - conversation_history from messages
     - current_docs from last assistant message
     - current_params from last assistant message
     - user_message

8. Render user prompt template

9. Call LLM:
   - system: STORY_DRAFT_GENERATOR_SYSTEM content
   - user: rendered template
   - model: từ prompt_templates

10. Parse JSON response

11. Merge params:
    - Input params (from first call) + autoSelectedParams

12. Save assistant message:
    - INSERT ai_messages (role = "assistant", content = JSON.stringify(response))

13. Return result:
    - conversationId
    - message: response.message
    - docs: response.docs
    - params: merged params
    - turnNumber: count user messages
```

---

## Error Handling

| Error | Code | Message |
|-------|------|---------|
| Empty conversationId | 400 | "Conversation ID is required" |
| Empty userMessage | 400 | "User message is required" |
| Conversation not found | 404 | "Conversation not found" |
| Wrong step | 400 | "Conversation is not in brainstorming step" |
| Invalid first call params | 400 | "Invalid parameters format" |
| LLM call failed | 500 | "AI service unavailable" |
| Invalid LLM response | 500 | "Failed to parse AI response" |

---

## Example

### First Call Request
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "{\"targetAudience\":1,\"targetCoreValue\":\"Tình bạn\",\"formatGenre\":1,\"contentGenre\":2,\"storyIdea\":\"Chú mèo Miu học cách chia sẻ đồ chơi\"}"
}
```

### First Call Response
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Mình đã tạo bản draft đầu tiên cho truyện về chú mèo Miu! Truyện kể về hành trình Miu học cách chia sẻ đồ chơi với bạn thỏ. Bạn xem qua các docs và cho mình biết muốn điều chỉnh gì nhé!",
  "docs": [
    {
      "type": 0,
      "title": "manuscript",
      "content": "Ngày xưa, trong khu vườn xinh đẹp, có chú mèo con tên Miu...(~450 từ)..."
    },
    {
      "type": 0,
      "title": "story_structure",
      "content": "**Act 1 - Setup:**\n- Giới thiệu Miu và bộ đồ chơi...\n**Act 2 - Conflict:**\n- Thỏ Bông đến chơi...\n**Act 3 - Resolution:**\n- Miu nhận ra niềm vui chia sẻ..."
    },
    {
      "type": 0,
      "title": "artistic_imagery",
      "content": "**Metaphors:**\n- Đồ chơi = tình bạn tiềm ẩn...\n**Visual Motifs:**\n- Màu cam ấm áp của Miu...\n**Color Palette:**\n- Pastel mềm mại phù hợp 2-3 tuổi..."
    },
    {
      "type": 0,
      "title": "moral_lesson",
      "content": "**Core Theme:**\nChia sẻ không làm mất đi mà còn nhân đôi niềm vui...\n**Emotional Journey:**\n1. Lo lắng khi phải chia sẻ...\n2. Phát hiện niềm vui khi chơi cùng..."
    }
  ],
  "params": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "formatGenre": 1,
    "contentGenre": 2,
    "storyIdea": "Chú mèo Miu học cách chia sẻ đồ chơi",
    "writingStyle": 1,
    "eraId": "era-modern-uuid",
    "locationId": "loc-garden-uuid"
  },
  "turnNumber": 1
}
```

### Subsequent Call Request
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "Thêm nhân vật chó Bông vào, và đổi sang phong cách thơ vần"
}
```

### Subsequent Call Response
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Đã thêm chó Bông vào câu chuyện và chuyển sang phong cách thơ vần! Giờ truyện có 3 bạn nhỏ: Miu, Thỏ và chó Bông cùng chơi đồ chơi.",
  "docs": [
    {
      "type": 0,
      "title": "manuscript",
      "content": "Trong vườn hoa xinh tươi,\nCó chú mèo Miu cười,\nĐồ chơi nhiều lắm lắm,\nMột mình chơi vui vui...(thơ vần ~400 từ)..."
    },
    {
      "type": 0,
      "title": "story_structure",
      "content": "...(updated với chó Bông)..."
    },
    {
      "type": 0,
      "title": "artistic_imagery",
      "content": "...(updated)..."
    },
    {
      "type": 0,
      "title": "moral_lesson",
      "content": "...(updated)..."
    }
  ],
  "params": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "formatGenre": 1,
    "contentGenre": 2,
    "storyIdea": "Chú mèo Miu học cách chia sẻ đồ chơi",
    "writingStyle": 2,
    "eraId": "era-modern-uuid",
    "locationId": "loc-garden-uuid"
  },
  "turnNumber": 2
}
```

---

## Notes

- **First call detection:** Check if last assistant message has `docs` field in content
- **Manuscript word limit:** AI MUST keep manuscript <500 từ to allow for expansion during actual story creation
- **Param persistence:** autoSelectedParams persist across turns unless user explicitly requests change
- **Docs update strategy:** AI updates only docs affected by user feedback, preserves unchanged docs
