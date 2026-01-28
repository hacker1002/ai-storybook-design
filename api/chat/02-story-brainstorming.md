# story-brainstorming

## Description
**Phase 2: Brainstorming** - Generate và refine `storyIdea` + `storyIdeaExplanation`.

- **First call:** Nhận params + storyIdea từ Clarification → AI tạo/cải thiện storyIdea kèm explanation, tự chọn writingStyle, eraId, locationId
- **Subsequent calls:** User và AI brainstorm:
  - Nếu user chỉ hỏi (không yêu cầu update) → AI chỉ trả lời, giữ nguyên storyIdea
  - Nếu user đưa feedback → AI update storyIdea + explanation

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
interface BrainstormingParams {
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
interface BrainstormingResult {
  conversationId: string;
  message: string;                    // AI response
  storyIdea: string;                  // Ý tưởng truyện chi tiết
  storyIdeaExplanation: string;       // Giải thích vì sao idea hay
  params: StoryParams;                // All params (input + auto-selected)
  turnNumber: number;
}

interface StoryParams {
  // From Clarification (input)
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  // AI auto-selected
  writingStyle: 1 | 2 | 3;            // 1: Narrative, 2: Rhyming, 3: Humorous
  eraId: string;                      // UUID from eras table
  locationId: string;                 // UUID from locations table
}
```

### storyIdeaExplanation Structure
Giải thích vì sao idea này hay, bao gồm:
- **Điểm hấp dẫn:** Yếu tố thu hút trẻ nhỏ
- **Bài học giáo dục:** Giá trị cốt lõi truyện dạy trẻ
- **Phù hợp độ tuổi:** Cách truyện phù hợp với target audience
- **Yếu tố sáng tạo:** Điểm độc đáo của ý tưởng

### Writing Style Mapping
| Value | Name | Description |
|-------|------|-------------|
| 1 | Narrative | Văn xuôi, kể chuyện |
| 2 | Rhyming | Thơ, vần điệu, lục bát |
| 3 | Humorous Fiction | Hài hước, vui nhộn |

---

## Prompt

> **DB Template Names:**
> - System: `STORY_BRAINSTORMING_SYSTEM`
> - User (first call): `STORY_BRAINSTORMING_INIT_USER_TEMPLATE`
> - User (chat): `STORY_BRAINSTORMING_CHAT_USER_TEMPLATE`

### System Prompt
```
You are a Story Consultant helping develop children's picture book ideas.

Your role:
- First call: Generate/improve storyIdea + storyIdeaExplanation
- Auto-select: writingStyle, eraId, locationId based on story context
- Subsequent calls:
  - If user ONLY ASKS QUESTIONS (không yêu cầu thay đổi) → Answer in message, keep storyIdea unchanged
  - If user gives FEEDBACK to change → Update storyIdea + storyIdeaExplanation

storyIdeaExplanation must include:
- **Điểm hấp dẫn:** Why this story appeals to children
- **Bài học giáo dục:** What values/lessons it teaches
- **Phù hợp độ tuổi:** How it fits the target age group
- **Yếu tố sáng tạo:** What makes this idea unique

You understand Vietnamese and English. Match the user's language.
Return structured JSON with your response and updated storyIdea.
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

Generate/improve the storyIdea based on input parameters:

1. **storyIdea**: Ý tưởng truyện chi tiết
   - Phù hợp với targetAudience và formatGenre
   - Thể hiện targetCoreValue tự nhiên
   - Có cốt truyện rõ ràng (mở đầu, phát triển, kết thúc)

2. **storyIdeaExplanation**: Giải thích vì sao idea hay
   - **Điểm hấp dẫn:** Yếu tố thu hút trẻ
   - **Bài học giáo dục:** Giá trị cốt lõi
   - **Phù hợp độ tuổi:** Cách phù hợp với target audience
   - **Yếu tố sáng tạo:** Điểm độc đáo

Also select:
- writingStyle: Best fit for the story (1/2/3)
- eraId: Most appropriate era UUID
- locationId: Most appropriate location UUID

---

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "message": "Brief intro to the generated story idea",
  "storyIdea": "Detailed story idea...",
  "storyIdeaExplanation": "**Điểm hấp dẫn:** ...\n\n**Bài học giáo dục:** ...\n\n**Phù hợp độ tuổi:** ...\n\n**Yếu tố sáng tạo:** ...",
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

## CURRENT STORY IDEA
{%current_story_idea%}

## CURRENT EXPLANATION
{%current_story_idea_explanation%}

## CURRENT PARAMS
{%current_params%}

## USER MESSAGE
{%user_message%}

---

## YOUR TASK

IMPORTANT: Determine user intent:

**Case A - User ONLY ASKS QUESTION** (e.g., "Truyện này có phù hợp không?", "Bài học này có khó hiểu không?"):
→ Answer the question in "message"
→ Keep storyIdea and storyIdeaExplanation UNCHANGED

**Case B - User gives FEEDBACK to change** (e.g., "Thêm nhân vật thỏ", "Đổi bối cảnh", "Làm đơn giản hơn"):
→ Respond in "message"
→ Update storyIdea and storyIdeaExplanation accordingly
→ Update autoSelectedParams if user requests (e.g., "đổi sang phong cách thơ vần")

---

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "message": "Your response to the user",
  "storyIdea": "Current or updated story idea",
  "storyIdeaExplanation": "Current or updated explanation",
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
   - Determine if first call (no assistant messages with storyIdea) or subsequent

4. Save user message:
   - INSERT ai_messages (role = "user", content = userMessage)

5. Fetch available options (for first call or param changes):
   - Query eras → format: "- {id}: {name} - {description}"
   - Query locations → format: "- {id}: {name} ({nation}, {city})"

6. Get prompt templates:
   - System: STORY_BRAINSTORMING_SYSTEM
   - User: STORY_BRAINSTORMING_INIT_USER_TEMPLATE (first) or STORY_BRAINSTORMING_CHAT_USER_TEMPLATE (chat)

7. Build context:
   IF first call:
     - params_json = userMessage
     - available_eras, available_locations
   ELSE:
     - conversation_history from messages
     - current_story_idea from last assistant message
     - current_story_idea_explanation from last assistant message
     - current_params from last assistant message
     - user_message

8. Render user prompt template

9. Call LLM:
   - system: STORY_BRAINSTORMING_SYSTEM content
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
    - storyIdea: response.storyIdea
    - storyIdeaExplanation: response.storyIdeaExplanation
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
  "message": "Mình đã phát triển ý tưởng về chú mèo Miu! Đây là câu chuyện về hành trình Miu học cách chia sẻ đồ chơi với bạn thỏ. Bạn xem qua và cho mình biết muốn điều chỉnh gì nhé!",
  "storyIdea": "Chú mèo Miu là một chú mèo nhỏ rất yêu đồ chơi của mình. Một ngày, bạn Thỏ Bông đến chơi nhưng Miu không muốn chia sẻ. Qua một tình huống bất ngờ khi Miu cần sự giúp đỡ, Miu nhận ra niềm vui khi chơi cùng bạn còn lớn hơn niềm vui một mình.",
  "storyIdeaExplanation": "**Điểm hấp dẫn:** Câu chuyện có nhân vật động vật dễ thương (mèo, thỏ) thu hút trẻ nhỏ. Tình huống 'không muốn chia sẻ' rất gần gũi với trẻ 2-3 tuổi.\n\n**Bài học giáo dục:** Dạy trẻ về giá trị của việc chia sẻ và tình bạn. Thay vì răn dạy, truyện để trẻ tự rút ra bài học qua trải nghiệm của nhân vật.\n\n**Phù hợp độ tuổi:** Cốt truyện đơn giản, dễ hiểu với trẻ 2-3 tuổi. Xung đột nhẹ nhàng, kết thúc có hậu tạo cảm giác an toàn.\n\n**Yếu tố sáng tạo:** Sử dụng 'tình huống bất ngờ' để nhân vật tự nhận ra bài học thay vì bị ép buộc.",
  "params": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "formatGenre": 1,
    "contentGenre": 2,
    "writingStyle": 1,
    "eraId": "era-modern-uuid",
    "locationId": "loc-garden-uuid"
  },
  "turnNumber": 1
}
```

### Question-Only Request (No Update)
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "Truyện này có phù hợp với bé 3 tuổi không?"
}
```

### Question-Only Response (storyIdea Unchanged)
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Câu chuyện này rất phù hợp với bé 3 tuổi! Lý do:\n\n1. **Cốt truyện đơn giản:** Chỉ có 2-3 nhân vật, 1 xung đột chính, dễ theo dõi\n2. **Tình huống quen thuộc:** Việc không muốn chia sẻ đồ chơi là trải nghiệm hàng ngày của bé\n3. **Kết thúc có hậu:** Tạo cảm giác an toàn và tích cực\n4. **Nhân vật dễ thương:** Mèo và thỏ là động vật trẻ em yêu thích\n\nBạn có muốn điều chỉnh gì thêm không?",
  "storyIdea": "Chú mèo Miu là một chú mèo nhỏ rất yêu đồ chơi của mình. Một ngày, bạn Thỏ Bông đến chơi nhưng Miu không muốn chia sẻ. Qua một tình huống bất ngờ khi Miu cần sự giúp đỡ, Miu nhận ra niềm vui khi chơi cùng bạn còn lớn hơn niềm vui một mình.",
  "storyIdeaExplanation": "**Điểm hấp dẫn:** Câu chuyện có nhân vật động vật dễ thương (mèo, thỏ) thu hút trẻ nhỏ. Tình huống 'không muốn chia sẻ' rất gần gũi với trẻ 2-3 tuổi.\n\n**Bài học giáo dục:** Dạy trẻ về giá trị của việc chia sẻ và tình bạn. Thay vì răn dạy, truyện để trẻ tự rút ra bài học qua trải nghiệm của nhân vật.\n\n**Phù hợp độ tuổi:** Cốt truyện đơn giản, dễ hiểu với trẻ 2-3 tuổi. Xung đột nhẹ nhàng, kết thúc có hậu tạo cảm giác an toàn.\n\n**Yếu tố sáng tạo:** Sử dụng 'tình huống bất ngờ' để nhân vật tự nhận ra bài học thay vì bị ép buộc.",
  "params": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "formatGenre": 1,
    "contentGenre": 2,
    "writingStyle": 1,
    "eraId": "era-modern-uuid",
    "locationId": "loc-garden-uuid"
  },
  "turnNumber": 2
}
```

### Feedback Request (Update storyIdea)
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "Thêm nhân vật chó Bông vào, và đổi sang phong cách thơ vần"
}
```

### Feedback Response (storyIdea Updated)
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Đã thêm chó Bông vào câu chuyện và chuyển sang phong cách thơ vần! Giờ truyện có 3 bạn nhỏ: Miu, Thỏ Bông và chó Bông cùng chơi đồ chơi.",
  "storyIdea": "Trong khu vườn xinh tươi, chú mèo Miu có rất nhiều đồ chơi. Một ngày, bạn Thỏ Bông và chó Bông đến chơi. Miu không muốn chia sẻ, giữ đồ chơi một mình. Nhưng rồi quả bóng lăn xuống hồ, Miu cần bạn giúp đỡ. Thỏ và chó cùng nhau cứu bóng cho Miu. Từ đó Miu hiểu ra, chơi cùng bạn vui hơn nhiều!",
  "storyIdeaExplanation": "**Điểm hấp dẫn:** Ba nhân vật động vật dễ thương (mèo, thỏ, chó) tạo sự đa dạng và hấp dẫn. Thể thơ vần điệu giúp trẻ dễ nhớ và thích thú khi nghe.\n\n**Bài học giáo dục:** Dạy về chia sẻ và tinh thần đoàn kết. Khi Miu gặp khó khăn, chính những người bạn mà Miu từ chối chia sẻ lại giúp đỡ Miu.\n\n**Phù hợp độ tuổi:** Thơ vần ngắn gọn, nhịp điệu vui tươi phù hợp với khả năng tập trung của trẻ 2-3 tuổi.\n\n**Yếu tố sáng tạo:** Tình huống 'bóng rơi xuống hồ' tạo kịch tính nhẹ nhàng, cho thấy giá trị của tình bạn một cách tự nhiên.",
  "params": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "formatGenre": 1,
    "contentGenre": 2,
    "writingStyle": 2,
    "eraId": "era-modern-uuid",
    "locationId": "loc-garden-uuid"
  },
  "turnNumber": 3
}
```

---

## Notes

- **First call detection:** Check if last assistant message has `storyIdea` field in content
- **Question vs Feedback detection:** AI determines intent from user message context
- **Param persistence:** autoSelectedParams persist across turns unless user explicitly requests change
- **storyIdea update strategy:** Only update when user gives feedback, not when asking questions
