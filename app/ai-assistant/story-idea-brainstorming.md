# Story Idea Brainstorming

## Description
Tính năng tạo truyện từ ý tưởng người dùng. User nhập prompt bất kỳ → AI extract thông tin → Client hỏi params còn thiếu qua Clarification → User tạo truyện ngay hoặc chat thêm với AI để phát triển ý tưởng.

**Conversation persistence:**
- `ai_conversations`: Session với `step = "brainstorming"` hoặc `step = "story_editing"`
- `ai_messages`: Messages (user + assistant)

## Flow Overview
```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 0: INITIAL PROMPT                                                   │
│   [CLIENT → API → AI]                                                       │
│                                                                             │
│   User nhập prompt → API tạo conversation + AI extract params               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ API response với extractedParams
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 1: CLARIFICATION                                                    │
│   [CLIENT ONLY]                                                             │
│                                                                             │
│   Hỏi tuần tự params còn thiếu (no API calls)                               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ all params collected
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 2: SUMMARY                                                          │
│   [CLIENT ONLY]                                                             │
│                                                                             │
│   Display summary → User chọn [Tạo truyện] hoặc [Chỉnh sửa thêm]            │
└───────────────────────┬─────────────────────────┬───────────────────────────┘
          "Tạo truyện"  │                         │  "Chỉnh sửa thêm"
                        ▼                         ▼
┌───────────────────────────────┐   ┌─────────────────────────────────────────┐
│   → generate-manuscript API   │   │   Phase 3: BRAINSTORMING                │
│   → Navigate to EDITOR        │   │   [CLIENT → API → AI]                   │
└───────────────────────────────┘   │                                         │
                                    │   Multi-turn chat → shouldEndBrainstorming│
                                    └────────────────────┬────────────────────┘
                                                         │ shouldEndBrainstorming = true
                                                         ▼
                                    ┌─────────────────────────────────────────┐
                                    │   → generate-manuscript API             │
                                    │   → Navigate to EDITOR                  │
                                    └─────────────────────────────────────────┘
```

---

## Phase 0: Initial Prompt

### Flow Pattern
`[CLIENT → API → AI]`

### Description
User nhập prompt đầu tiên → API tạo conversation mới + AI phân tích extract story idea và params.

### Responsibilities

**CLIENT:**
- Collect user's initial prompt
- Handle loading/error states
- Transition to Clarification with extracted data

**API:**
- Create conversation in DB
- Build prompt with user message
- Call AI to extract story idea + params
- Save messages to DB
- Return extracted data

**AI:**
- Analyze user message
- Extract story idea (nếu rõ ràng)
- Extract any mentioned params (audience, style, etc.)

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
  storyIdea: string | null;
  extractedParams: ExtractedParams;
  turnNumber: 1;
}
```

### Extraction Examples
| User Input | Extracted |
|------------|-----------|
| "Truyện cho bé 3-4 tuổi" | `targetAudience: 1` |
| "Mình muốn dạy bé về tình bạn" | `targetCoreValue: "Tình bạn"` |
| "Truyện viễn tưởng về robot" | `storyIdea: "Truyện về robot"`, `genre: 2` |
| "Truyện thơ lục bát" | `writingStyle: 2` |
| "Bối cảnh thời Hùng Vương" | AI queries `eras` → `eraId` |
| "Phong cách tranh màu nước" | AI queries `art_styles` → `artstyleId` |

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

### Responsibilities

**CLIENT:**
- Determine which params still missing
- Display questions sequentially
- Handle skip/back navigation
- Merge answers with extracted params

### Question Order (Strict)
Hỏi tuần tự **CHỈ NẾU param còn thiếu**:

| # | Param | Question | Options | Default | Skip |
|---|-------|----------|---------|---------|------|
| 1 | `storyIdea` | "Bạn muốn tạo truyện về chủ đề gì?" | Text | placeholder: "Ví dụ: Chú mèo con đi phiêu lưu" | ✅ |
| 2 | `targetCoreValue` | "Truyện này truyền tải bài học gì?" | Text | placeholder: "Ví dụ: Tình bạn, Sự dũng cảm" | ✅ |
| 3 | `targetAudience` | "Dành cho độ tuổi nào?" | 1: 2-5, 2: 6-8, 3: 9-10 | **1** (2-5 tuổi) | ✅ |
| 4 | `dimension` | "Kích thước sách?" | 1: Vuông, 2: A4 ngang, 3: A4 dọc | **1** (Vuông) | ✅ |
| 5 | `genre` | "Thể loại?" | 1-5: Fantasy/Sci-Fi/Mystery/Romance/Horror | **1** (Fantasy) | ✅ |
| 6 | `writingStyle` | "Phong cách viết?" | 1: Kể chuyện, 2: Thơ vần, 3: Hài hước | **1** (Kể chuyện) | ✅ |
| 7 | `eraId` | "Bối cảnh thời đại?" | Select from `eras` | **first item** | ✅ |
| 8 | `locationId` | "Địa điểm?" | Select from `locations` | **first item** | ✅ |
| 9 | `artstyleId` | "Phong cách vẽ?" | Select from `art_styles` | **first item** | ✅ |

**Default Value Rules:**
- **Selection options:** Luôn pre-select option đầu tiên làm default → tránh null
- **Text fields:** Hiển thị placeholder gợi ý, không bắt buộc user nhập
- **DB-driven options (era, location, artstyle):** Pre-select item đầu tiên từ query result

### User Actions
| Action | Description |
|--------|-------------|
| Answer | Set param value, next question |
| Skip | Giữ default value, next question (if allowSkip) |
| Back | Previous question |

### State Management
```typescript
interface ClarificationState {
  conversationId: string;
  storyIdea: string | null;
  params: ExtractedParams;      // Initialized with default values
  currentQuestionIndex: number;
  status: "asking" | "completed";
}
```

### Exit Condition
All params answered (với default hoặc user input) → Transition to Phase 2

**Guaranteed:** Từ Phase 2 trở đi, tất cả params đều có giá trị (không null).

---

## Phase 2: Summary

### Flow Pattern
`[CLIENT ONLY]`

### Description
Hiển thị tổng hợp story idea + params. User chọn tạo truyện ngay hoặc vào Brainstorming.

### Responsibilities

**CLIENT:**
- Display summary of all collected data (không có null values)
- Handle user's choice

### User Actions
| Action | Description |
|--------|-------------|
| "Tạo truyện" | Call generate-manuscript → Navigate to Editor |
| "Chỉnh sửa thêm" | Transition to Phase 3 (Brainstorming) |

### State Management
```typescript
interface SummaryState {
  conversationId: string;
  storyIdea: string;
  params: ExtractedParams;  // All params have values (no nulls)
  status: "reviewing" | "creating" | "editing";
}
```

### Exit Conditions
- "Tạo truyện" → `POST /api/text-generation/generate-manuscript` → Editor
- "Chỉnh sửa thêm" → Phase 3

---

## Phase 3: Brainstorming

### Flow Pattern
`[CLIENT → API → AI]`

### Description
Multi-turn chat với AI (Story Consultant) để phát triển ý tưởng. AI detects khi user muốn kết thúc → `shouldEndBrainstorming = true`.

**Entry:** User click "Chỉnh sửa thêm" từ Summary.

### Responsibilities

**CLIENT:**
- Display chat interface
- Show current storyIdea + params in sidebar
- Send messages to API
- Merge updated params from AI response
- Handle shouldEndBrainstorming → trigger story creation

**API:**
- Load conversation history
- Build prompt with context (story idea, params, history)
- Call AI
- Save messages to DB
- Return response with updated params

**AI:**
- Act as Story Consultant
- Help develop/refine story idea
- Update params based on conversation
- Detect end signals ("OK xong rồi", "Tạo truyện đi")
- Return `shouldEndBrainstorming: true` when appropriate

### Initial Message
```
"Bạn muốn chỉnh sửa thêm gì? Mình có thể giúp bạn:
- Phát triển thêm cốt truyện
- Thêm/sửa thông tin nhân vật
- Đổi bối cảnh, thời đại
- Thay đổi phong cách viết
- Hoặc bất kỳ điều gì khác!"
```

### User Actions
| Action | Description |
|--------|-------------|
| Send message | Chat với AI để refine story idea |
| End signals | "OK xong rồi", "Tạo truyện đi" → AI returns shouldEnd |

### Request/Response
```typescript
interface BrainstormingRequest {
  conversationId: string;
  userMessage: string;
}

interface BrainstormingResponse {
  conversationId: string;
  message: string;
  extractedParams: ExtractedParams;
  storyIdea: string;
  shouldEndBrainstorming: boolean;
  turnNumber: number;
}
```

### State Management
```typescript
interface BrainstormingState {
  conversationId: string;
  messages: ChatMessage[];
  storyIdea: string;
  extractedParams: ExtractedParams;
  status: "active" | "ending" | "completed";
}

interface ChatMessage {
  id: string;
  role: "user" | "assistant";
  content: string;
  timestamp: Date;
}
```

### Exit Condition
`shouldEndBrainstorming = true` → Call generate-manuscript → Navigate to Editor

---

## Shared Types

```typescript
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

type CurrentPhase = "initial" | "clarification" | "summary" | "brainstorming";
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/chat/story-brainstorming` | Initial prompt (Phase 0) + Brainstorming chat (Phase 3) |
| POST | `/api/text-generation/generate-manuscript` | Create story from final params |

### POST /api/chat/story-brainstorming
```typescript
interface ChatRequest {
  conversationId?: string;      // null = Phase 0, existing = Phase 3
  userMessage: string;
}

interface ChatResponse {
  conversationId: string;
  message: string;
  storyIdea: string | null;
  extractedParams: ExtractedParams;
  shouldEndBrainstorming: boolean;
  turnNumber: number;
}
```

### POST /api/text-generation/generate-manuscript
```typescript
interface GenerateManuscriptRequest {
  storyIdea: string;
  attributes: {
    dimension: 1 | 2 | 3;           // Has default from Clarification
    targetAudience: 1 | 2 | 3;      // Has default from Clarification
    targetCoreValue: string;         // Has default from Clarification
    genre: 1 | 2 | 3 | 4 | 5;       // Has default from Clarification
    writingStyle: 1 | 2 | 3;        // Has default from Clarification
    eraId: string;                   // Has default from Clarification
    locationId: string;              // Has default from Clarification
    artstyleId: string;              // Has default from Clarification
  };
  language?: string;                 // Default: "vi"
}

interface GenerateManuscriptResponse {
  storyId: string;
  snapshotId: string;
  status: "completed" | "partial";
}
```

**Note:** Tất cả params đều có giá trị (từ Clarification phase với default values).

---

## DB Queries (Supabase)

| Operation | Table | Description |
|-----------|-------|-------------|
| SELECT | `eras` | Fetch era options for Clarification |
| SELECT | `locations` | Fetch location options for Clarification |
| SELECT | `art_styles` | Fetch art style options for Clarification |
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
| API | generate-manuscript failed | Return error, keep params for retry |

---

## Edge Cases

### 1. User provides complete info in first prompt
- **Trigger:** "Tạo truyện về chú mèo dũng cảm cho bé 3 tuổi, phong cách màu nước"
- **Layer:** API (Phase 0)
- **Solution:** AI extracts all → Clarification skips already-extracted params

### 2. User wants to start over
- **Trigger:** "Làm lại", "Bắt đầu lại"
- **Layer:** CLIENT (Clarification) or AI (Brainstorming)
- **Solution:** Clarification: reset to Phase 0. Brainstorming: AI resets context in same conversation

### 3. Conflicting params
- **Trigger:** "Truyện horror cho bé 3 tuổi"
- **Layer:** AI
- **Solution:** AI clarifies conflict in response

### 4. Unrecognized era/location/artstyle
- **Trigger:** User mentions unknown value
- **Layer:** API + CLIENT
- **Solution:** API returns null → Clarification shows available options

### 5. User closes browser mid-flow
- **Trigger:** Browser close/refresh
- **Layer:** CLIENT
- **Solution:** Phase 0+3 persist in DB (can resume). Clarification local state lost (restart from Phase 0)
