# story-brainstorming

## Description
**Story Consultant Agent** - Hỗ trợ user brainstorm ý tưởng truyện qua conversation. Agent tự động trích xuất story parameters từ context hội thoại và nhận biết khi user muốn kết thúc để chuyển sang tạo truyện.

**Related Feature:** `app/ai-assistant/story-idea-brainstorming.md`

## DB Schema Dependencies

### Tables Used
| Table | Purpose |
|-------|---------|
| `ai_conversations` | Lưu chat session, step = `brainstorming` |
| `ai_messages` | Lưu messages (user + assistant) |
| `eras` | Match user input với era options |
| `locations` | Match user input với location options |
| `art_styles` | Match user input với art style options |
| `prompt_templates` | Lấy system prompt và user template |

### Fields Used
- `ai_conversations.id`, `ai_conversations.user_id`, `ai_conversations.story_id`, `ai_conversations.step`
- `ai_messages.id`, `ai_messages.conversation_id`, `ai_messages.role`, `ai_messages.content`
- `eras.id`, `eras.name`, `eras.description`
- `locations.id`, `locations.name`, `locations.description`, `locations.nation`, `locations.city`
- `art_styles.id`, `art_styles.name`, `art_styles.description`
- `prompt_templates.name`, `prompt_templates.content`, `prompt_templates.model`

---

## Parameters

```typescript
interface StoryBrainstormingParams {
  // Conversation ID (null = create new, string = continue existing)
  conversationId?: string;

  // User's new message
  userMessage: string;

  // Finalization flag - when true, AI will fill all null params with smart defaults
  isFinalize?: boolean;
}
```

### Dimension Mapping
| Value | Size | User Input Examples |
|-------|------|---------------------|
| 1 | Square (20x20cm) | "sách vuông", "20x20", "vuông" |
| 2 | A4 Landscape | "A4 ngang", "landscape", "ngang" |
| 3 | A4 Portrait | "A4 dọc", "portrait", "dọc" |

### Target Audience Mapping
| Value | Age Range | User Input Examples |
|-------|-----------|---------------------|
| 1 | Preschool (2-5 tuổi) | "bé 3 tuổi", "mẫu giáo", "trẻ nhỏ" |
| 2 | Primary (6-8 tuổi) | "bé 7 tuổi", "tiểu học", "lớp 1-2" |
| 3 | Tweens (9-10 tuổi) | "bé 10 tuổi", "lớp 4-5" |

### Genre Mapping
| Value | Name | User Input Examples |
|-------|------|---------------------|
| 1 | Fantasy | "fantasy", "phép thuật", "cổ tích", "kỳ ảo" |
| 2 | Sci-Fi | "sci-fi", "robot", "tương lai", "không gian" |
| 3 | Mystery | "bí ẩn", "thám tử", "mystery" |
| 4 | Romance | "lãng mạn", "tình cảm", "romance" |
| 5 | Horror | "kinh dị", "ma", "horror" |

### Writing Style Mapping
| Value | Name | User Input Examples |
|-------|------|---------------------|
| 1 | Narrative | "văn xuôi", "kể chuyện", "narrative" |
| 2 | Rhyming | "thơ", "vần", "lục bát", "rhyming" |
| 3 | Humorous Fiction | "hài hước", "vui nhộn", "funny" |

---

## Result

```typescript
interface StoryBrainstormingResult {
  // Conversation
  conversationId: string;          // ID của conversation (new hoặc existing)

  // AI response (parsed từ LLM JSON output)
  message: string;                 // AI reply text
  extractedParams: ExtractedParams;
  storyIdea: string;
  shouldEndBrainstorming: boolean;
  isFinalize: boolean;             // Indicates if this is a finalized response

  // Metadata
  turnNumber: number;              // Số lượt chat hiện tại
}

interface ExtractedParams {
  dimension?: 1 | 2 | 3;
  targetAudience?: 1 | 2 | 3;
  targetCoreValue?: string;
  genre?: 1 | 2 | 3 | 4 | 5;
  writingStyle?: 1 | 2 | 3;
  eraId?: string;
  locationId?: string;
  artstyleId?: string;
}
```

---

## Prompt

> **DB Template Names:**
> - System: `STORY_CONSULTANT_SYSTEM`
> - User: `STORY_CONSULTANT_USER_TEMPLATE`

### System Prompt
```
You are a friendly Story Consultant helping users develop children's picture book ideas through conversation.

Your role:
- Ask open-ended questions to develop story ideas
- Extract story parameters when user mentions them naturally
- Guide users toward complete story concepts
- Recognize when user wants to finish brainstorming

You understand Vietnamese and English. Match the user's language.

You output structured JSON that includes both your reply and extracted information.
```

### User Prompt Template
```
## CONVERSATION HISTORY
{%conversation_history%}

## CURRENT EXTRACTED PARAMS
{%current_params%}

## AVAILABLE OPTIONS (for matching)

### Eras
{%available_eras%}

### Locations
{%available_locations%}

### Art Styles
{%available_art_styles%}

---

## FINALIZATION MODE: {%is_finalize%}

---

## YOUR TASK

1. Respond naturally to continue the conversation
2. Extract any story parameters mentioned (see mapping rules below)
3. Summarize the current story idea based on conversation
4. Detect if user wants to finish brainstorming

### IF FINALIZATION MODE = true:
- This is the FINAL response - user wants to proceed with story creation
- Write a compelling, complete story idea summary (2-3 sentences) synthesizing all discussed elements
- **MUST fill ALL null params with smart defaults** based on:
  - Context from conversation (preferred)
  - If no context: dimension=2, targetAudience=1, genre=1, writingStyle=1
  - For eraId/locationId/artstyleId: pick the most appropriate option from available list if not mentioned
- Set shouldEndBrainstorming = true
- Message should confirm the finalized story concept enthusiastically

## PARAMETER EXTRACTION RULES

### Numeric Params (match user input to value)
- dimension: 1=square/vuông, 2=landscape/ngang, 3=portrait/dọc
- targetAudience: 1=preschool/2-5 tuổi, 2=primary/6-8 tuổi, 3=tweens/9-10 tuổi
- genre: 1=fantasy, 2=scifi, 3=mystery, 4=romance, 5=horror
- writingStyle: 1=narrative, 2=rhyming/thơ, 3=humorous

### UUID Params (match with available options)
- eraId: Match user input with era name/description -> return id
- locationId: Match user input with location name -> return id
- artstyleId: Match user input with art style name -> return id

### End Detection Phrases
- "OK tao truyen", "tao truyen di", "create story"
- "xong roi", "done", "hoan thanh"
- "minh thich y nay", "y tuong nay duoc roi"

---

## OUTPUT FORMAT

Return ONLY valid JSON:
{
  "message": "Your conversational response to the user",
  "extractedParams": {
    "dimension": null | 1 | 2 | 3,
    "targetAudience": null | 1 | 2 | 3,
    "targetCoreValue": "The main lesson of the story (e.g., Bravery, Kindness...)",
    "genre": null | 1 | 2 | 3 | 4 | 5,
    "writingStyle": null | 1 | 2 | 3,
    "eraId": null | "uuid-string",
    "locationId": null | "uuid-string",
    "artstyleId": null | "uuid-string"
  },
  "storyIdea": "Current accumulated story concept",
  "shouldEndBrainstorming": false | true,
  "isFinalize": false | true
}
```

---

## Flow

```
1. Validate input:
   - userMessage không được rỗng

2. Handle conversation:
   IF conversationId is null:
     - CREATE new ai_conversations (step = "brainstorming")
     - conversationId = new ID
   ELSE:
     - VERIFY conversation exists và step = "brainstorming"

3. Save user message:
   - INSERT vào ai_messages (role = "user", content = userMessage)

4. Load conversation history:
   - SELECT messages FROM ai_messages WHERE conversation_id = conversationId ORDER BY created_at

5. Build current_params:
   - Lấy extractedParams từ assistant message cuối cùng

6. Lấy prompt templates từ DB:
   - Query prompt_templates với name = "STORY_CONSULTANT_SYSTEM"
   - Query prompt_templates với name = "STORY_CONSULTANT_USER_TEMPLATE"

7. Format conversation history:
   - Convert messages thành text: "User: ...\nAssistant: ..."
   - Với assistant messages, chỉ lấy field "message" từ JSON content

8. Fetch và format available options từ DB:
   - Query bảng `eras` → format: "- {id}: {name} - {description}"
   - Query bảng `locations` → format: "- {id}: {name} ({nation}, {city})"
   - Query bảng `art_styles` → format: "- {id}: {name} - {description}"

9. Render user prompt template với variables

10. Call LLM với system prompt và rendered user prompt

11. Parse JSON response từ LLM

12. Save assistant message:
    - INSERT vào ai_messages (role = "assistant", content = JSON.stringify(llmResponse))

13. Return result:
    - conversationId
    - message: llmResponse.message
    - extractedParams: llmResponse.extractedParams
    - storyIdea: llmResponse.storyIdea
    - shouldEndBrainstorming: llmResponse.shouldEndBrainstorming
    - turnNumber: count user messages
```

---

## Error Handling

| Error | Code | Message |
|-------|------|---------|
| Empty userMessage | 400 | "User message is required" |
| Conversation not found | 404 | "Conversation not found" |
| Wrong conversation step | 400 | "Conversation is not in brainstorming step" |
| LLM call failed | 500 | "AI service unavailable" |
| Invalid LLM response | 500 | "Failed to parse AI response" |

---

## Example

### Request (New Conversation)
```json
{
  "conversationId": null,
  "userMessage": "Mình muốn viết truyện về một chú mèo con đi phiêu lưu"
}
```

### Response
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Ý tưởng hay đó! Chú mèo con này sẽ phiêu lưu ở đâu? Trong rừng, thành phố, hay một vương quốc kỳ diệu?",
  "extractedParams": {},
  "storyIdea": "Chú mèo con đi phiêu lưu",
  "shouldEndBrainstorming": false,
  "isFinalize": false,
  "turnNumber": 1
}
```

### Request (Continue Conversation)
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "Trong rừng nhé, cho bé 4 tuổi đọc, kiểu fantasy. Mình muốn dạy bé về tình bạn"
}
```

### Response
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Tuyệt vời! Truyện fantasy về chú mèo phiêu lưu trong rừng cho bé 4 tuổi với thông điệp về tình bạn - rất phù hợp! Chú mèo có tên gì không?",
  "extractedParams": {
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "genre": 1
  },
  "storyIdea": "Chú mèo con phiêu lưu trong rừng, gặp gỡ các bạn thú. Truyện fantasy dành cho bé 4 tuổi.",
  "shouldEndBrainstorming": false,
  "isFinalize": false,
  "turnNumber": 2
}
```

### Request (Finalization)
```json
{
  "conversationId": "conv-uuid-123",
  "userMessage": "OK mình thích ý tưởng này rồi",
  "isFinalize": true
}
```

### Response (Finalized)
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Tuyệt vời! Mình đã hoàn thiện ý tưởng truyện cho bạn. Câu chuyện về chú mèo Miu phiêu lưu trong khu rừng kỳ diệu, kết bạn với các loài thú và học được giá trị của tình bạn chân thành. Sẵn sàng để bắt đầu tạo truyện!",
  "extractedParams": {
    "dimension": 2,
    "targetAudience": 1,
    "targetCoreValue": "Tình bạn",
    "genre": 1,
    "writingStyle": 1,
    "eraId": "era-uuid-modern",
    "locationId": "location-uuid-forest",
    "artstyleId": "artstyle-uuid-watercolor"
  },
  "storyIdea": "Chú mèo con tên Miu lạc vào khu rừng kỳ diệu, nơi mèo gặp gỡ và kết bạn với các loài thú như thỏ, sóc và cú. Qua những cuộc phiêu lưu đầy màu sắc, Miu học được rằng tình bạn chân thành là kho báu quý giá nhất.",
  "shouldEndBrainstorming": true,
  "isFinalize": true,
  "turnNumber": 3
}
```

