# Story Idea Brainstorming

## Description
Tính năng chat với AI để brainstorm ý tưởng truyện. User trao đổi qua lại với AI để phát triển ý tưởng, AI tự động trích xuất các parameters từ cuộc hội thoại, sau đó tạo manuscript và chuyển sang Editor.

**Conversation được persist trong DB:**
- `ai_conversations`: Lưu session với `step = "brainstorming"`
- `ai_messages`: Lưu từng message (user + assistant)

## Flow Overview
```
┌─────────────────┐
│  Brainstorming  │ ←→ User-AI conversation (persist to DB)
│   (multi-turn)  │    AI extracts params, saved in message content
└────────┬────────┘
         │ user confirms or requests "create story"
         ▼
┌─────────────────┐
│  Clarification  │ AI asks only missing params
│   (optional)    │ User can Skip → AI auto-fills
└────────┬────────┘
         │ all params collected
         ▼
┌─────────────────┐
│    Summary      │ Display: story idea + all params
│                 │ User clicks "Create this story"
└────────┬────────┘
         │ confirmed
         ▼
┌─────────────────┐
│  API Call       │ generate-manuscript with params
└────────┬────────┘
         │ success
         ▼
┌─────────────────┐
│    Editor       │ Navigate to story editor
└─────────────────┘
```

---

## Phase 1: Brainstorming

### Description
Cuộc hội thoại nhiều lượt giữa User và AI để phát triển ý tưởng truyện. AI đóng vai Story Consultant, giúp user phát triển ý tưởng và tự động trích xuất parameters khi user đề cập.

### Extractable Parameters
| Parameter | Type | DB Field | Description |
|-----------|------|----------|-------------|
| `dimension` | SMALLINT | `story.dimension` | 1: Square (20x20cm), 2: A4 Landscape, 3: A4 Portrait |
| `targetAudience` | SMALLINT | `story.target_audience` | 1: preschool (2-5), 2: primary (6-8), 3: tweens (9-10) |
| `genre` | SMALLINT | `story.genre` | 1: fantasy, 2: scifi, 3: mystery, 4: romance, 5: horror |
| `writingStyle` | SMALLINT | `story.writing_style` | 1: Narrative, 2: Rhyming, 3: Humorous Fiction |
| `eraId` | UUID | `story.era_id` | FK → eras table |
| `locationId` | UUID | `story.location_id` | FK → locations table |
| `artstyleId` | UUID | `story.artstyle_id` | FK → art_styles table |

### Extraction Examples
| User Input | Extracted |
|------------|-----------|
| "Truyện cho bé 3-4 tuổi" | `targetAudience: 1` |
| "Truyện viễn tưởng về robot" | `genre: 2` (scifi) |
| "Truyện thơ lục bát" | `writingStyle: 2` (rhyming) |
| "Bối cảnh thời Hùng Vương" | AI queries `eras` → `eraId` |
| "Phong cách tranh màu nước" | AI queries `art_styles` → `artstyleId` |

### Exit Conditions
Chuyển sang Phase 2 khi:
- User xác nhận ý tưởng: "OK, tạo truyện đi" / "Mình thích ý này rồi"
- User yêu cầu tạo: "Create story" / "Bắt đầu tạo truyện"
- User nói "done" / "xong" / "hoàn thành"

### State Management
```typescript
// Frontend state - sync với DB
interface BrainstormingState {
  conversationId: string | null;  // null = chưa có conversation
  messages: ChatMessage[];        // Load từ ai_messages
  storyIdea: string;              // Parse từ assistant message cuối
  extractedParams: ExtractedParams; // Merge từ tất cả assistant messages
  status: "active" | "completed";
}

// Message hiển thị trên UI
interface ChatMessage {
  id: string;
  role: "user" | "assistant";
  content: string;                // Với assistant: chỉ hiển thị field "message" từ JSON
  timestamp: Date;
  extractedParams?: ExtractedParams;  // Parse từ JSON content (assistant only)
}

interface ExtractedParams {
  dimension?: 1 | 2 | 3;
  targetAudience?: 1 | 2 | 3;
  genre?: 1 | 2 | 3 | 4 | 5;
  writingStyle?: 1 | 2 | 3;
  eraId?: string;
  locationId?: string;
  artstyleId?: string;
}
```

### Data Flow
```
1. User gửi message
   ↓
2. Call API: POST /api/chat/story-brainstorming
   - conversationId: state.conversationId
   - userMessage: input
   - availableOptions: { eras, locations, artStyles }
   ↓
3. API tự động:
   - Create/verify conversation trong ai_conversations
   - Save user message vào ai_messages
   - Load history, call LLM
   - Save assistant response vào ai_messages
   ↓
4. Response trả về:
   - conversationId (lưu vào state nếu mới)
   - message (hiển thị)
   - extractedParams (merge vào state)
   - storyIdea (cập nhật state)
   - shouldEndBrainstorming (check để chuyển phase)
```

---

## Phase 2: Clarification

### Description
AI kiểm tra các parameters còn thiếu và chỉ hỏi những thông tin chưa có. User có thể Skip để AI tự chọn giá trị phù hợp.

### Required Parameters
Tất cả parameters đều required cho `generate-manuscript`:
- `dimension` (required: general)
- `targetAudience` (required: general)
- `targetCoreValue` (required: general) - **không extract từ chat, hỏi riêng**
- `genre` (required: creative)
- `writingStyle` (required: creative)
- `eraId` (required: creative)
- `locationId` (required: creative)
- `artstyleId` (required: creative)

### Question Flow
```typescript
interface ClarificationQuestion {
  paramKey: string;
  question: string;
  options?: SelectOption[];
  allowSkip: boolean;
  defaultValue?: any;
}

interface SelectOption {
  value: string | number;
  label: string;
  description?: string;
}
```

### Example Questions
| Missing Param | Question |
|---------------|----------|
| `dimension` | "Bạn muốn sách có kích thước nào? (Vuông 20x20cm, A4 ngang, A4 dọc)" |
| `targetAudience` | "Truyện này dành cho độ tuổi nào? (2-5 tuổi, 6-8 tuổi, 9-10 tuổi)" |
| `targetCoreValue` | "Bài học chính của truyện là gì? (VD: Sự dũng cảm, Lòng tốt...)" |
| `genre` | "Thể loại truyện? (Fantasy, Sci-Fi, Mystery, Romance, Horror)" |
| `writingStyle` | "Phong cách viết? (Kể chuyện, Thơ vần, Hài hước)" |
| `eraId` | "Bối cảnh thời đại? (Hiện đại, Cổ tích, Thời tiền sử...)" |
| `locationId` | "Địa điểm diễn ra? (Việt Nam, Nhật Bản, Vương quốc tưởng tượng...)" |
| `artstyleId` | "Phong cách vẽ? (Màu nước, Chibi, Tranh giấy...)" |

### Skip Behavior
Khi user skip, AI tự động chọn giá trị dựa trên:
1. Context từ story idea
2. Các parameters đã có
3. Default phổ biến nhất

```typescript
function getDefaultValue(paramKey: string, context: BrainstormingState): any {
  switch (paramKey) {
    case "dimension":
      return 1;  // Square là phổ biến nhất
    case "targetAudience":
      return 1;  // Preschool nếu không rõ
    case "genre":
      return context.storyIdea.includes("phép thuật") ? 1 : 1;  // Fantasy default
    // ... analyze context for better defaults
  }
}
```

### State Management
```typescript
interface ClarificationState {
  missingParams: string[];
  currentQuestion: ClarificationQuestion | null;
  answeredParams: Record<string, any>;
  skippedParams: string[];
  status: "asking" | "completed";
}
```

---

## Phase 3: Summary

### Description
AI tổng hợp và hiển thị toàn bộ thông tin đã thu thập để user xác nhận trước khi tạo truyện.

### Display Format
```
┌────────────────────────────────────────┐
│  📖 TÓM TẮT Ý TƯỞNG TRUYỆN            │
├────────────────────────────────────────┤
│                                        │
│  Ý tưởng:                              │
│  [Story idea description...]           │
│                                        │
├────────────────────────────────────────┤
│  Thông số:                             │
│  • Kích thước: Vuông (20x20cm)         │
│  • Độ tuổi: 2-5 tuổi                   │
│  • Bài học: Sự dũng cảm                │
│  • Thể loại: Fantasy                   │
│  • Phong cách viết: Kể chuyện          │
│  • Thời đại: Hiện đại                  │
│  • Địa điểm: Việt Nam                  │
│  • Phong cách vẽ: Màu nước             │
│                                        │
├────────────────────────────────────────┤
│  [   Chỉnh sửa   ]  [ Tạo truyện ✓ ]   │
└────────────────────────────────────────┘
```

### Actions
| Action | Description |
|--------|-------------|
| "Chỉnh sửa" | Quay lại Clarification để sửa params |
| "Tạo truyện" | Xác nhận và gọi API |

### State
```typescript
interface SummaryState {
  storyIdea: string;
  finalParams: GenerateManuscriptParams["attributes"];
  status: "reviewing" | "confirmed" | "editing";
}
```

---

## Phase 4: API Call

### Endpoint
`POST /api/text-generation/generate-manuscript`

### Request
```typescript
interface GenerateManuscriptRequest {
  storyIdea: string;
  attributes: {
    dimension: 1 | 2 | 3;
    targetAudience: 1 | 2 | 3;
    targetCoreValue: string;
    genre: 1 | 2 | 3 | 4 | 5;
    writingStyle: 1 | 2 | 3;
    eraId: string;
    locationId: string;
    artstyleId: string;
  };
  language?: string;  // Default: "vi"
}
```

### Response Handling
```typescript
// Success
interface SuccessResponse {
  storyId: string;
  snapshotId: string;
  status: "completed" | "partial";
}

// Error
interface ErrorResponse {
  error: string;
  code: string;
  details?: any;
}
```

### Post-Success Actions
```typescript
// Update conversation khi tạo story thành công
await supabase
  .from('ai_conversations')
  .update({
    story_id: storyId,
    step: 'complete',
    updated_at: new Date()
  })
  .eq('id', conversationId);
```

### Loading State
- Show progress indicator
- Display current step being processed (1-5)
- Allow cancel (with confirmation)

---

## Phase 5: Navigate to Editor

### Success Flow
```typescript
router.push(`/editor/${storyId}`);
```

### Error Handling
| Error | Action |
|-------|--------|
| API timeout | Show retry option, keep all params |
| Partial result | Navigate to Editor, show warning about incomplete steps |
| Server error | Show error message, allow retry or edit params |

---

## UI Components

### ChatInterface
```typescript
interface ChatInterfaceProps {
  conversationId: string | null;
  messages: ChatMessage[];
  onSendMessage: (content: string) => void;
  isLoading: boolean;
  extractedParams: ExtractedParams;  // Show in sidebar
}
```

### ParamSelector
```typescript
interface ParamSelectorProps {
  question: ClarificationQuestion;
  onSelect: (value: any) => void;
  onSkip: () => void;
}
```

### SummaryCard
```typescript
interface SummaryCardProps {
  storyIdea: string;
  params: FinalParams;
  onEdit: () => void;
  onCreate: () => void;
  isCreating: boolean;
}
```

---

## AI Prompt Templates

### STORY_CONSULTANT_SYSTEM
System prompt cho AI trong phase Brainstorming:
- Đóng vai Story Consultant
- Hỏi gợi mở để phát triển ý tưởng
- Trích xuất parameters khi user đề cập
- Nhận biết khi user muốn kết thúc brainstorming

### STORY_CONSULTANT_USER_TEMPLATE
User prompt template:
- `{%conversation_history%}`: Lịch sử chat (bao gồm message mới nhất)
- `{%current_params%}`: Params đã extract từ các turns trước
- `{%available_eras%}`: Danh sách eras từ DB
- `{%available_locations%}`: Danh sách locations từ DB
- `{%available_art_styles%}`: Danh sách art styles từ DB

---

## Edge Cases

### 1. User provides conflicting params
- AI clarifies: "Bạn vừa nói cho bé 3 tuổi, nhưng thể loại Horror có thể không phù hợp. Bạn muốn đổi thể loại hay độ tuổi?"

### 2. User wants to start over
- Trigger phrases: "Làm lại", "Bắt đầu lại", "Reset"
- Tạo conversation mới (không xóa conversation cũ)

### 3. User mentions unrecognized era/location/artstyle
- AI: "Mình chưa có phong cách 'X' trong thư viện. Bạn có thể chọn từ: [list options]"

### 4. Long brainstorming session
- Periodically summarize extracted params
- Max 20 turns trước khi suggest kết thúc

### 5. Resume previous conversation
- Load messages từ ai_messages WHERE conversation_id = X
- Parse tất cả assistant messages để rebuild extractedParams

---

## Dependencies

### Supabase Queries

**Fetch options:**
```typescript
supabase.from('eras').select('id, name, description')
supabase.from('locations').select('id, name, description, nation, city')
supabase.from('art_styles').select('id, name, description')
```

**Load conversation history (resume):**
```typescript
// Get conversation
const { data: conversation } = await supabase
  .from('ai_conversations')
  .select('*')
  .eq('id', conversationId)
  .single();

// Get messages
const { data: messages } = await supabase
  .from('ai_messages')
  .select('*')
  .eq('conversation_id', conversationId)
  .order('created_at', { ascending: true });
```

### API Endpoints
- `POST /api/chat/story-brainstorming` - Chat với AI (xem `api/chat/00-story-brainstorming.md`)
- `POST /api/text-generation/generate-manuscript` - Create story

### DB Tables
- `ai_conversations`: Store chat sessions
- `ai_messages`: Store messages
- `eras`: Query for era options
- `locations`: Query for location options
- `art_styles`: Query for art style options
- `story`: Created by generate-manuscript
- `snapshot`: Created by generate-manuscript
