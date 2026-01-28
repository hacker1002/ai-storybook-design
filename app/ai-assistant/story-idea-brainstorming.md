# Story Idea Brainstorming

## Description
Tính năng tạo truyện từ ý tưởng người dùng. User nhập prompt bất kỳ → AI extract `storyIdea` + params → Client hỏi params còn thiếu qua Clarification → AI tạo/cải thiện `storyIdea` kèm `storyIdeaExplanation` (giải thích vì sao idea hay, dạy trẻ bài học gì, ...) → User và AI brainstorm cho tới khi hài lòng → Tạo truyện.

**Conversation persistence:**
- `ai_conversations`: Session với `step = "brainstorming"` hoặc `step = "story_editing"`
- `ai_messages`: Messages (user + assistant)

## Flow Overview
```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 0: INITIAL PROMPT                                                   │
│   [CLIENT → API → AI]                                                       │
│                                                                             │
│   User nhập prompt → API tạo conversation + AI extract storyIdea + params   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ API response với storyIdea + extractedParams
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 1: CLARIFICATION                                                    │
│   [CLIENT ONLY]                                                             │
│                                                                             │
│   Hỏi tuần tự params còn thiếu (no API calls)                               │
│   Content genre options phụ thuộc vào format genre đã chọn                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ all params collected
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 2: BRAINSTORMING                                                    │
│   [CLIENT → API → AI]                                                       │
│                                                                             │
│   First call: Send params + storyIdea → AI tạo storyIdea + explanation      │
│     (giải thích vì sao idea hay, dạy trẻ bài học gì, ...)                   │
│     AI tự chọn writing_style, era_id, location_id nếu user không specify    │
│   Next calls: User và AI brainstorm                                         │
│     - Nếu user chỉ hỏi → AI trả lời, không update idea                      │
│     - Nếu user feedback → AI update storyIdea + explanation                 │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ User click "Tạo truyện"
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               End                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 0: Initial Prompt

### Flow Pattern
`[CLIENT → API → AI]`

### Description
User nhập prompt đầu tiên → API tạo conversation mới + AI phân tích extract params cơ bản.

### Responsibilities

**CLIENT:**
- Collect user's initial prompt
- Handle loading/error states
- Transition to Clarification with extracted data

**API:**
- Create conversation in DB
- Build prompt with user message
- Call AI to extract params
- Save messages to DB
- Return extracted data

**AI:**
- Analyze user message
- Extract `storyIdea` if user describes a specific story concept (null if only preferences)
- Extract any mentioned params (audience, core value, format genre, content genre)

### User Actions
| Action | Description |
|--------|-------------|
| Submit prompt | Gửi ý tưởng ban đầu → API extract params |

### Request/Response
```typescript
interface InitialPromptRequest {
  userMessage: string;
}

interface InitialPromptResponse {
  conversationId: string;
  message: string;
  storyIdea: string | null;      // Extracted story idea (null if not specified)
  extractedParams: ExtractedParams;
  turnNumber: number;
}
```

### Extraction Examples
| User Input | Extracted |
|------------|-----------|
| "Truyện cho bé 3 tuổi" | `targetAudience: 1`, `storyIdea: null` |
| "Mình muốn dạy bé về tình bạn" | `targetCoreValue: "Tình bạn"`, `storyIdea: null` |
| "Truyện cổ tích ru ngủ" | `formatGenre: 2`, `contentGenre: 6`, `storyIdea: null` |
| "Truyện trinh thám cho bé tiểu học" | `targetAudience: 3`, `contentGenre: 1`, `storyIdea: null` |
| "Kể về chú mèo Miu học cách chia sẻ đồ chơi" | `storyIdea: "Chú mèo Miu học cách chia sẻ đồ chơi"`, `targetCoreValue: "Chia sẻ"` |
| "Câu chuyện về cậu bé đi lạc trong rừng và được thỏ giúp đỡ" | `storyIdea: "Cậu bé đi lạc trong rừng được thỏ giúp đỡ"` |

### State Management
```typescript
interface InitialPromptState {
  userPrompt: string;
  status: "idle" | "loading" | "success" | "error";
  conversationId: string | null;
  storyIdea: string | null;
  extractedParams: ExtractedParams;
  error: string | null;
}
```

### Exit Condition
API response received → Transition to Phase 1 (Clarification)

---

## Phase 1: Clarification

### Flow Pattern
`[CLIENT ONLY]`

### Description
Client-side phase: hỏi các params còn thiếu theo thứ tự cố định. Không gọi API.

**Lưu ý:** `contentGenre` options phụ thuộc vào `formatGenre` đã chọn.

### Responsibilities

**CLIENT:**
- Determine which params still missing
- Display questions sequentially
- Filter contentGenre options based on selected formatGenre
- Handle next/back navigation
- Merge answers with extracted params

### Question Order (Strict)
Hỏi tuần tự **CHỈ NẾU param còn thiếu**:

| # | Param | Question | Options | Default | Next |
|---|-------|----------|---------|---------|------|
| 1 | `targetAudience` | "Dành cho độ tuổi nào?" | 1: 2-3, 2: 4-5, 3: 6-8 | **1** (2-3 tuổi) | ✅ |
| 2 | `targetCoreValue` | "Truyện này truyền tải bài học gì?" | Text | placeholder: "Ví dụ: Tình bạn, Sự dũng cảm" | ✅ |
| 3 | `formatGenre` | "Thể loại sách?" | See Format Genre Options | **1** (Narrative) | ✅ |
| 4 | `contentGenre` | "Thể loại nội dung?" | **Depends on formatGenre** | **first valid option** | ✅ |

### Format Genre Options
| Value | Name (EN) | Name (VI) |
|-------|-----------|-----------|
| 1 | Narrative Picture Books | Sách tranh kể chuyện |
| 2 | Lullaby/Bedtime Books | Sách ru ngủ |
| 3 | Concept Books | Sách khái niệm |
| 4 | Non-fiction Picture Books | Sách tranh phi hư cấu |
| 5 | Early Reader | Sách tập đọc |
| 6 | Wordless Picture Books | Sách tranh không chữ |

### Content Genre Options (by Format Genre)

| formatGenre | Valid contentGenre Values |
|-------------|---------------------------|
| 1 (Narrative) | 1, 2, 3, 4, 5, 6, 7, 8 |
| 2 (Lullaby) | 2, 3, 6 |
| 3 (Concept) | 3, 7, 10 |
| 4 (Non-fiction) | 9, 10, 11 |
| 5 (Early Reader) | 1, 2, 3, 4, 5, 6, 7, 9, 10 |
| 6 (Wordless) | 1, 2, 3, 5, 6, 7 |

### Content Genre Full List
| Value | Name (EN) | Name (VI) |
|-------|-----------|-----------|
| 1 | Mystery | Bí ẩn |
| 2 | Fantasy | Kỳ ảo |
| 3 | Realistic Fiction | Đời thường |
| 4 | Historical Fiction | Dã sử |
| 5 | Science Fiction | Viễn tưởng |
| 6 | Folklore/Fairy Tales | Dân gian |
| 7 | Humor | Hài hước |
| 8 | Horror/Scary | Kinh dị nhẹ |
| 9 | Biography | Tiểu sử |
| 10 | Informational | Kiến thức |
| 11 | Memoir | Hồi ký |

### Content Genre Filter Logic
```typescript
const CONTENT_GENRE_BY_FORMAT: Record<number, number[]> = {
  1: [1, 2, 3, 4, 5, 6, 7, 8],  // Narrative
  2: [2, 3, 6],                  // Lullaby
  3: [3, 7, 10],                 // Concept
  4: [9, 10, 11],                // Non-fiction
  5: [1, 2, 3, 4, 5, 6, 7, 9, 10], // Early Reader
  6: [1, 2, 3, 5, 6, 7],        // Wordless
};

function getContentGenreOptions(formatGenre: number): number[] {
  return CONTENT_GENRE_BY_FORMAT[formatGenre] || [];
}
```

### User Actions
| Action | Description |
|--------|-------------|
| Answer | Set param value |
| Next | Giữ default value, next question |
| Back | Previous question |

### State Management
```typescript
interface ClarificationState {
  conversationId: string;
  params: ExtractedParams;
  currentQuestionIndex: number;
  status: "asking" | "completed";
}
```

### Exit Condition
All params answered (với default hoặc user input) → Transition to Phase 2 (Brainstorming)

---

## Phase 2: Brainstorming

### Flow Pattern
`[CLIENT → API → AI]`

### Description
Multi-turn chat với AI để phát triển ý tưởng truyện.

**First call:** AI nhận tất cả params + `storyIdea` → Tạo/cải thiện `storyIdea` kèm `storyIdeaExplanation` (giải thích vì sao idea này hay, dạy trẻ bài học gì, ...). AI tự chọn `writing_style`, `era_id`, `location_id` nếu user không specify.

**Next calls:** User và AI brainstorm:
- Nếu user chỉ hỏi câu hỏi (không yêu cầu update idea) → AI chỉ trả lời, không update `storyIdea`
- Nếu user đưa feedback muốn thay đổi → AI update `storyIdea` + `storyIdeaExplanation`

**Entry:** All params collected từ Clarification phase + storyIdea từ Phase 0.

### Responsibilities

**CLIENT:**
- Display chat interface
- Show current `storyIdea` + `storyIdeaExplanation`
- Send messages to API
- Update `storyIdea` + `storyIdeaExplanation` from AI response
- Handle "Tạo truyện" button click

**API:**
- Load conversation history
- Build prompt with context (params, storyIdea, storyIdeaExplanation, history)
- Call AI
- Save messages to DB
- Return response with updated `storyIdea` + `storyIdeaExplanation`

**AI:**
- Act as Story Consultant
- First call: Generate `storyIdea` + `storyIdeaExplanation`
- Auto-select: `writingStyle`, `eraId`, `locationId` based on story context
- Answer user questions about the story (không update idea nếu user chỉ hỏi)
- Update `storyIdea` + `storyIdeaExplanation` khi user đưa feedback muốn thay đổi

### Initial Idea Generation
Khi vào Phase 2, API call đầu tiên sẽ:
1. Nhận tất cả params từ Clarification + `storyIdea` từ Phase 0
2. AI generate/cải thiện `storyIdea` + `storyIdeaExplanation`:
   - `storyIdea`: Ý tưởng truyện chi tiết
   - `storyIdeaExplanation`: Giải thích vì sao idea này hay:
     - Truyện có gì hấp dẫn
     - Giáo dục trẻ bài học gì
     - Phù hợp với độ tuổi như thế nào
     - Yếu tố sáng tạo/độc đáo
3. AI tự chọn `writingStyle`, `eraId`, `locationId` phù hợp với story
4. Return `storyIdea` + `storyIdeaExplanation` + auto-selected params cho client

### User Actions
| Action | Description |
|--------|-------------|
| Send message | Chat với AI để refine storyIdea |
| "Tạo truyện" | Kết thúc brainstorming, tạo truyện từ storyIdea |

### Request/Response
```typescript
// Unified request for both init and chat
interface BrainstormingChatRequest {
  conversationId: string;
  userMessage: string;           // First call: JSON.stringify(params + storyIdea)
}

interface BrainstormingResponse {
  conversationId: string;
  message: string;
  storyIdea: string;             // Ý tưởng truyện chi tiết
  storyIdeaExplanation: string;  // Giải thích vì sao idea hay
  params: StoryParams;           // All params (extracted + auto-selected)
  turnNumber: number;
}

interface StoryParams {
  // ExtractedParams (from Phase 0 + Clarification)
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  // AutoSelectedParams (AI chooses based on story)
  writingStyle: 1 | 2 | 3;
  eraId: string;
  locationId: string;
}
```

**First call userMessage format:**
```json
{
  "targetAudience": 1,
  "targetCoreValue": "Tình bạn",
  "formatGenre": 1,
  "contentGenre": 2,
  "storyIdea": "Chú mèo Miu học cách chia sẻ đồ chơi"
}
```

**Response Example:**
```json
{
  "storyIdea": "Chú mèo Miu là một chú mèo nhỏ rất yêu đồ chơi của mình. Một ngày, bạn Thỏ Bông đến chơi nhưng Miu không muốn chia sẻ. Qua một tình huống bất ngờ khi Miu cần sự giúp đỡ, Miu nhận ra niềm vui khi chơi cùng bạn còn lớn hơn niềm vui một mình.",
  "storyIdeaExplanation": "**Điểm hấp dẫn:** Câu chuyện có nhân vật động vật dễ thương (mèo, thỏ) thu hút trẻ nhỏ. Tình huống 'không muốn chia sẻ' rất gần gũi với trẻ 2-3 tuổi.\n\n**Bài học giáo dục:** Dạy trẻ về giá trị của việc chia sẻ và tình bạn. Thay vì răn dạy, truyện để trẻ tự rút ra bài học qua trải nghiệm của nhân vật.\n\n**Phù hợp độ tuổi:** Cốt truyện đơn giản, dễ hiểu với trẻ 2-3 tuổi. Xung đột nhẹ nhàng, kết thúc có hậu tạo cảm giác an toàn.\n\n**Yếu tố sáng tạo:** Sử dụng 'tình huống bất ngờ' để nhân vật tự nhận ra bài học thay vì bị ép buộc."
}
```

### State Management
```typescript
interface BrainstormingState {
  conversationId: string;
  messages: ChatMessage[];
  storyIdea: string;             // Ý tưởng truyện chi tiết
  storyIdeaExplanation: string;  // Giải thích vì sao idea hay
  params: StoryParams;           // All params (extracted + auto-selected)
  status: "active" | "creating";
}

interface ChatMessage {
  id: string;
  role: "user" | "assistant";
  content: string;
  timestamp: Date;
}
```

### Exit Condition
User click "Tạo truyện"

---

## Shared Types

```typescript
// Phase 0: Partial params (some may be null)
interface ExtractedParams {
  targetAudience?: 1 | 2 | 3;
  targetCoreValue?: string;
  formatGenre?: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre?: number;
  storyIdea?: string;
}

// Phase 2: Complete params (all required)
interface StoryParams {
  // From Phase 0 + Clarification
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  // AI auto-selected
  writingStyle: 1 | 2 | 3;
  eraId: string;
  locationId: string;
}

type CurrentPhase = "initial" | "clarification" | "brainstorming";
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/chat/story-brainstorming/initial` | Phase 0: Extract storyIdea + params |
| POST | `/api/chat/story-brainstorming` | Phase 2: Generate/refine storyIdea |

### POST /api/chat/story-brainstorming/initial
Phase 0: Tạo conversation mới, AI extract `storyIdea` và params từ user prompt.

```typescript
interface InitialPromptRequest {
  userMessage: string;
}

interface InitialPromptResponse {
  conversationId: string;
  message: string;
  storyIdea: string | null;
  extractedParams: ExtractedParams;
  turnNumber: number;
}
```

### POST /api/chat/story-brainstorming
Phase 2: Generate và refine storyIdea. Unified interface cho cả init và chat.

**Lưu ý:** Nếu user chỉ hỏi câu hỏi (không yêu cầu update idea) → AI chỉ trả lời trong `message`, giữ nguyên `storyIdea` + `storyIdeaExplanation`.

```typescript
interface BrainstormingChatRequest {
  conversationId: string;
  userMessage: string;           // First call: JSON.stringify(params + storyIdea)
}

interface BrainstormingResponse {
  conversationId: string;
  message: string;
  storyIdea: string;             // Ý tưởng truyện chi tiết
  storyIdeaExplanation: string;  // Giải thích vì sao idea hay
  params: StoryParams;           // All params (extracted + auto-selected)
  turnNumber: number;
}
```

**First call userMessage:**
```json
{"targetAudience":1,"targetCoreValue":"Tình bạn","formatGenre":1,"contentGenre":2,"storyIdea":"Chú mèo Miu học cách chia sẻ"}
```

**Subsequent calls:** Regular chat message (string)
---

## DB Queries (Supabase)

| Operation | Table | Description |
|-----------|-------|-------------|
| SELECT | `eras` | AI fetch để auto-select eraId |
| SELECT | `locations` | AI fetch để auto-select locationId |
| INSERT | `ai_conversations` | Create conversation (API - Phase 0) |
| INSERT | `ai_messages` | Save messages (API) |
| UPDATE | `ai_conversations` | Set `story_id`, `step = 'story_editing'` after creation |

---

## Error Handling

| Layer | Error | Action |
|-------|-------|--------|
| CLIENT | Network timeout | Show retry option, preserve all state |
| CLIENT | Invalid input | Local validation before API call |
| API | AI extraction failed | Return empty extractedParams, continue flow |
| API | DB write failed | Return error, allow retry |

---

## Edge Cases

### 1. User provides complete info in first prompt
- **Trigger:** "Tạo truyện cổ tích về chú mèo học chia sẻ cho bé 3 tuổi"
- **Layer:** API (Phase 0)
- **Solution:** AI extracts storyIdea + all params → Clarification skips already-extracted params

### 2. User wants to start over
- **Trigger:** "Làm lại", "Bắt đầu lại"
- **Layer:** CLIENT (Clarification) or AI (Brainstorming)
- **Solution:** Clarification: reset to Phase 0. Brainstorming: AI resets storyIdea

### 3. Conflicting format/content genre
- **Trigger:** User selects invalid contentGenre for formatGenre
- **Layer:** CLIENT
- **Solution:** Client filters contentGenre options based on formatGenre

### 4. User wants to change auto-selected params
- **Trigger:** "Đổi sang phong cách thơ vần" trong Brainstorming
- **Layer:** AI
- **Solution:** AI updates autoSelectedParams và adjust storyIdea accordingly

### 5. User closes browser mid-flow
- **Trigger:** Browser close/refresh
- **Layer:** CLIENT
- **Solution:** Phase 0+2 persist in DB (can resume). Clarification local state lost (restart from Phase 0)

### 6. User directly says "Tạo truyện" without brainstorming
- **Trigger:** User satisfied with first storyIdea
- **Layer:** CLIENT
- **Solution:** Allow immediate story creation from first storyIdea

### 7. User only asks questions without wanting to update idea
- **Trigger:** "Truyện này có phù hợp với bé 3 tuổi không?", "Bài học này có quá khó hiểu không?"
- **Layer:** AI
- **Solution:** AI chỉ trả lời câu hỏi trong `message`, giữ nguyên `storyIdea` + `storyIdeaExplanation`
