# story-brainstorming-initial

## Description
**Phase 0: Initial Prompt** - Tạo conversation mới và extract `storyIdea` + params từ user prompt đầu tiên. AI phân tích message để nhận diện ý tưởng truyện và các thông số cơ bản (audience, core value, format genre, content genre).

**Related Feature:** `app/ai-assistant/story-idea-brainstorming.md`

## DB Schema Dependencies

### Tables Used
| Table | Purpose |
|-------|---------|
| `ai_conversations` | Tạo session mới với `step = "brainstorming"` |
| `ai_messages` | Lưu messages (user + assistant) |
| `prompt_templates` | Lấy system prompt và user template |

### Fields Used
- `ai_conversations.id`, `ai_conversations.user_id`, `ai_conversations.step`
- `ai_messages.id`, `ai_messages.conversation_id`, `ai_messages.role`, `ai_messages.content`
- `prompt_templates.name`, `prompt_templates.content`, `prompt_templates.model`

---

## Parameters

```typescript
interface InitialPromptParams {
  userMessage: string;  // User's initial prompt
}
```

---

## Result

```typescript
interface InitialPromptResult {
  conversationId: string;
  message: string;                    // AI response to user
  storyIdea: string | null;           // Extracted story concept (null if only preferences)
  extractedParams: ExtractedParams;   // Params explicitly mentioned
  turnNumber: number;                 // Always 1 for initial
}

interface ExtractedParams {
  targetAudience?: 1 | 2 | 3 | 4;     // 1: 2-3, 2: 3-5, 3: 6-8, 4: 9+
  targetCoreValue?: 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21;
  formatGenre?: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre?: number;              // 1-11, depends on formatGenre
}
```

### Target Audience Mapping
| Value | Age Range | User Input Examples |
|-------|-----------|---------------------|
| 1 | 2-3 tuổi | "bé 2 tuổi", "bé 3 tuổi", "sơ sinh" |
| 2 | 3-5 tuổi | "bé 4 tuổi", "mẫu giáo", "mầm non" |
| 3 | 6-8 tuổi | "bé 7 tuổi", "tiểu học", "lớp 1-2" |
| 4 | 9+ tuổi | "bé 9 tuổi", "lớp 4-5", "middle grade" |

### Target Core Value Mapping
| Value | Name (VI) | User Input Examples |
|-------|-----------|---------------------|
| 1 | Dũng cảm | "dũng cảm", "mạnh mẽ", "brave" |
| 2 | Quan tâm | "quan tâm", "chăm sóc", "caring" |
| 3 | Trung thực | "trung thực", "thật thà", "honest" |
| 4 | Kiên trì | "kiên trì", "cố gắng", "persistence" |
| 5 | Biết ơn | "biết ơn", "cảm ơn", "grateful" |
| 6 | Bản lĩnh | "bản lĩnh", "tự tin", "confidence" |
| 7 | Thấu cảm | "thấu cảm", "đồng cảm", "empathy" |
| 8 | Chính trực | "chính trực", "ngay thẳng", "integrity" |
| 9 | Vị tha | "vị tha", "chia sẻ", "selfless" |
| 10 | Tự thức | "tự thức", "tự nhận thức", "self-awareness" |
| 11 | Tình bạn | "tình bạn", "bạn bè", "friendship" |
| 12 | Hợp tác | "hợp tác", "làm việc nhóm", "teamwork" |
| 13 | Chấp nhận sự khác biệt | "khác biệt", "đa dạng", "diversity" |
| 14 | Tử tế | "tử tế", "tốt bụng", "kindness" |
| 15 | Tò mò | "tò mò", "khám phá", "curiosity" |
| 16 | Tự lập | "tự lập", "độc lập", "independence" |
| 17 | Xử lý nỗi sợ | "nỗi sợ", "sợ hãi", "overcome fear" |
| 18 | Quản lý cảm xúc | "cảm xúc", "bình tĩnh", "emotional regulation" |
| 19 | Chuyển giao | "chuyển giao", "thay đổi", "transition" |
| 20 | Bảo vệ môi trường | "môi trường", "thiên nhiên", "environment" |
| 21 | Trí tưởng tượng | "tưởng tượng", "sáng tạo", "imagination" |

### Format Genre Mapping
| Value | Name | User Input Examples |
|-------|------|---------------------|
| 1 | Narrative Picture Books | "sách tranh", "truyện kể", "narrative" |
| 2 | Lullaby/Bedtime Books | "ru ngủ", "bedtime", "trước khi ngủ" |
| 3 | Concept Books | "sách khái niệm", "học màu sắc", "concept" |
| 4 | Non-fiction Picture Books | "phi hư cấu", "non-fiction", "thực tế" |
| 5 | Early Reader | "tập đọc", "early reader" |
| 6 | Wordless Picture Books | "không chữ", "wordless" |

### Content Genre Mapping
| Value | Name | User Input Examples |
|-------|------|---------------------|
| 1 | Mystery | "bí ẩn", "thám tử", "mystery" |
| 2 | Fantasy | "kỳ ảo", "phép thuật", "fantasy" |
| 3 | Realistic Fiction | "đời thường", "realistic" |
| 4 | Historical Fiction | "dã sử", "lịch sử", "historical" |
| 5 | Science Fiction | "viễn tưởng", "sci-fi", "robot" |
| 6 | Folklore/Fairy Tales | "cổ tích", "dân gian", "fairy tale" |
| 7 | Humor | "hài hước", "vui nhộn", "funny" |
| 8 | Horror/Scary | "kinh dị nhẹ", "ma", "scary" |
| 9 | Biography | "tiểu sử", "biography" |
| 10 | Informational | "kiến thức", "informational" |
| 11 | Memoir | "hồi ký", "memoir" |

---

## Prompt

> **DB Template Names:**
> - System: `STORY_EXTRACTOR_SYSTEM`
> - User: `STORY_EXTRACTOR_USER_TEMPLATE`

### System Prompt
```
You are a Story Idea Extractor. Analyze user's initial message to:
1. Extract the story idea (if user describes a specific concept)
2. Extract any mentioned parameters (audience, core value, format genre, content genre)
3. Respond naturally to encourage further conversation

You understand Vietnamese and English. Match the user's language.
Return structured JSON with your analysis.
```

### User Prompt Template
```
## USER MESSAGE
{%user_message%}

## YOUR TASK
Analyze the message and extract:
1. storyIdea: The story concept if user describes one (null if only preferences)
2. extractedParams: Only params explicitly mentioned
3. message: A friendly response to continue the conversation

## PARAMETER EXTRACTION RULES

### storyIdea
- Extract when user describes characters, plot, or specific scenario
- Keep null if user only mentions preferences (age, genre, etc.)
- Examples:
  - "Truyện về chú mèo học chia sẻ" → "Chú mèo học cách chia sẻ"
  - "Truyện cho bé 3 tuổi" → null (only preference)

### targetAudience (extract when age mentioned)
- 1: 2-3 tuổi
- 2: 3-5 tuổi
- 3: 6-8 tuổi
- 4: 9+ tuổi

### targetCoreValue (SMALLINT 1-21)
- ONLY extract when user EXPLICITLY states the lesson/moral
- Map to closest enum value from Core Value Mapping table
- "dạy bé về tình bạn" → 11, "bài học là sự dũng cảm" → 1
- "truyện về bạn bè" → DO NOT extract (not explicit lesson)

### formatGenre (extract when book format mentioned)
- 1: Narrative/sách tranh kể chuyện
- 2: Lullaby/ru ngủ
- 3: Concept/khái niệm
- 4: Non-fiction/phi hư cấu
- 5: Early Reader/tập đọc
- 6: Wordless/không chữ

### contentGenre (extract when story genre mentioned)
- 1: Mystery, 2: Fantasy, 3: Realistic, 4: Historical, 5: Sci-Fi
- 6: Folklore, 7: Humor, 8: Horror, 9: Biography, 10: Informational, 11: Memoir

---

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "message": "Your friendly response to the user",
  "storyIdea": "Extracted story concept" | null,
  "extractedParams": {
    "targetAudience": null | 1 | 2 | 3 | 4,
    "targetCoreValue": null | 1-21,
    "formatGenre": null | 1-6,
    "contentGenre": null | 1-11
  }
}
```

---

## Flow

```
1. Validate input:
   - userMessage không được rỗng

2. Create conversation:
   - INSERT ai_conversations (step = "brainstorming")
   - conversationId = new ID

3. Save user message:
   - INSERT ai_messages (role = "user", content = userMessage)

4. Get prompt templates:
   - Query prompt_templates với name = "STORY_EXTRACTOR_SYSTEM"
   - Query prompt_templates với name = "STORY_EXTRACTOR_USER_TEMPLATE"

5. Render user prompt:
   - Replace {%user_message%} với userMessage

6. Call LLM:
   - system: STORY_EXTRACTOR_SYSTEM content
   - user: rendered template
   - model: từ prompt_templates

7. Parse JSON response

8. Save assistant message:
   - INSERT ai_messages (role = "assistant", content = JSON.stringify(response))

9. Return result:
   - conversationId
   - message: response.message
   - storyIdea: response.storyIdea
   - extractedParams: response.extractedParams
   - turnNumber: 1
```

---

## Error Handling

| Error | Code | Message |
|-------|------|---------|
| Empty userMessage | 400 | "User message is required" |
| LLM call failed | 500 | "AI service unavailable" |
| Invalid LLM response | 500 | "Failed to parse AI response" |
| DB write failed | 500 | "Database error" |

---

## Example

### Request
```json
{
  "userMessage": "Mình muốn viết truyện cổ tích về chú mèo học chia sẻ cho bé 3 tuổi"
}
```

### Response
```json
{
  "conversationId": "conv-uuid-123",
  "message": "Ý tưởng hay! Truyện cổ tích về chú mèo học chia sẻ rất phù hợp cho bé 3 tuổi. Bạn đã nghĩ chú mèo sẽ gặp ai và chia sẻ điều gì chưa?",
  "storyIdea": "Chú mèo học cách chia sẻ",
  "extractedParams": {
    "targetAudience": 1,
    "targetCoreValue": 9,
    "formatGenre": null,
    "contentGenre": 6
  },
  "turnNumber": 1
}
```

### Extraction Examples

| User Input | storyIdea | extractedParams |
|------------|-----------|-----------------|
| "Truyện cho bé 3 tuổi" | null | `{ targetAudience: 1 }` |
| "Mình muốn dạy bé về tình bạn" | null | `{ targetCoreValue: 11 }` |
| "Truyện cổ tích ru ngủ" | null | `{ formatGenre: 2, contentGenre: 6 }` |
| "Kể về chú mèo Miu học chia sẻ đồ chơi" | "Chú mèo Miu học cách chia sẻ đồ chơi" | `{ targetCoreValue: 9 }` |
| "Câu chuyện về cậu bé đi lạc trong rừng" | "Cậu bé đi lạc trong rừng" | `{}` |
