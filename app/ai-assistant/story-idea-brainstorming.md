# Story Idea Brainstorming

## Description
Tính năng tạo truyện từ ý tưởng người dùng. User nhập prompt bất kỳ → AI extract `storyIdea` + params → Client hỏi params còn thiếu qua Clarification → AI tạo 4 docs (manuscript, story_structure, artistic_imagery, moral_lesson) → User và AI brainstorm cho tới khi hài lòng → Tạo truyện.

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
│   First call: Send params + storyIdea → AI tạo 4 docs (manuscript <500 từ,  │
│               story_structure, artistic_imagery, moral_lesson)              │
│     AI tự chọn writing_style, era_id, location_id nếu user không specify    │
│   Next calls: User và AI brainstorm, AI update docs theo feedback           │
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
- Handle skip/back navigation
- Merge answers with extracted params

### Question Order (Strict)
Hỏi tuần tự **CHỈ NẾU param còn thiếu**:

| # | Param | Question | Options | Default | Skip |
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
| Answer | Set param value, next question |
| Skip | Giữ default value, next question |
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

**First call:** AI nhận tất cả params + `storyIdea` → Tạo 4 loại docs (manuscript <500 từ, story_structure, artistic_imagery, moral_lesson). AI tự chọn `writing_style`, `era_id`, `location_id` nếu user không specify.

**Next calls:** User và AI brainstorm. AI trả lời câu hỏi, update docs theo feedback của user.

**Entry:** All params collected từ Clarification phase + storyIdea từ Phase 0.

### Responsibilities

**CLIENT:**
- Display chat interface
- Show current docs (manuscript, story_structure, artistic_imagery, moral_lesson)
- Send messages to API
- Update docs from AI response
- Handle "Tạo truyện" button click

**API:**
- Load conversation history
- Build prompt with context (params, storyIdea, history, current docs)
- Call AI
- Save messages to DB
- Return response with updated docs

**AI:**
- Act as Story Consultant
- First call: Generate 4 docs (manuscript <500 từ, story_structure, artistic_imagery, moral_lesson)
- Auto-select: `writingStyle`, `eraId`, `locationId` based on story context
- Help develop/refine docs based on user feedback
- Answer user questions about the story
- Update docs when user requests changes

### Initial Draft Generation
Khi vào Phase 2, API call đầu tiên sẽ:
1. Nhận tất cả params từ Clarification + `storyIdea` từ Phase 0
2. AI generate 4 loại docs:
   - `manuscript`: Draft truyện (<500 từ)
   - `story_structure`: Cấu trúc 3 hồi, conflicts, pacing
   - `artistic_imagery`: Metaphors, symbols, hình tượng nghệ thuật
   - `moral_lesson`: Bài học đạo đức, themes, emotional journey
3. AI tự chọn `writingStyle`, `eraId`, `locationId` phù hợp với story
4. Return docs[] + auto-selected params cho client

### User Actions
| Action | Description |
|--------|-------------|
| Send message | Chat với AI để refine docs |
| "Tạo truyện" | Kết thúc brainstorming, tạo truyện từ docs |

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
  docs: StoryDoc[];              // 4 types of docs (see Shared Types)
  params: StoryParams;           // All params (extracted + auto-selected)
  turnNumber: number;
}

interface StoryParams {
  // ExtractedParams (from Phase 0 + Clarification)
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  storyIdea: string | null;
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

**Docs Structure:**
| title | type | Mô tả |
|-------|------|-------|
| `manuscript` | 0 | Draft truyện (<500 từ) |
| `story_structure` | 0 | Cấu trúc 3 hồi, conflicts, pacing |
| `artistic_imagery` | 0 | Metaphors, symbols, hình tượng nghệ thuật |
| `moral_lesson` | 0 | Bài học đạo đức, themes, emotional journey |

### State Management
```typescript
interface BrainstormingState {
  conversationId: string;
  messages: ChatMessage[];
  docs: StoryDoc[];              // 4 types of docs
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
  storyIdea: string | null;
  // AI auto-selected
  writingStyle: 1 | 2 | 3;
  eraId: string;
  locationId: string;
}

interface StoryDoc {
  type: 0 | 1 | 2;               // 0: system, 1: user edit, 2: user read-only
  title: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  content: string;
}

type CurrentPhase = "initial" | "clarification" | "brainstorming";
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/chat/story-brainstorming/initial` | Phase 0: Extract storyIdea + params |
| POST | `/api/chat/story-brainstorming/draft` | Phase 2: Generate/refine docs |

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

### POST /api/chat/story-brainstorming/draft
Phase 2: Generate và refine docs. Unified interface cho cả init và chat.

```typescript
interface BrainstormingChatRequest {
  conversationId: string;
  userMessage: string;           // First call: JSON.stringify(params + storyIdea)
}

interface BrainstormingResponse {
  conversationId: string;
  message: string;
  docs: StoryDoc[];
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
- **Solution:** Clarification: reset to Phase 0. Brainstorming: AI resets docs

### 3. Conflicting format/content genre
- **Trigger:** User selects invalid contentGenre for formatGenre
- **Layer:** CLIENT
- **Solution:** Client filters contentGenre options based on formatGenre

### 4. User wants to change auto-selected params
- **Trigger:** "Đổi sang phong cách thơ vần" trong Brainstorming
- **Layer:** AI
- **Solution:** AI updates autoSelectedParams và adjust docs accordingly

### 5. User closes browser mid-flow
- **Trigger:** Browser close/refresh
- **Layer:** CLIENT
- **Solution:** Phase 0+2 persist in DB (can resume). Clarification local state lost (restart from Phase 0)

### 6. User directly says "Tạo truyện" without brainstorming
- **Trigger:** User satisfied with first docs
- **Layer:** CLIENT
- **Solution:** Allow immediate story creation from first docs
