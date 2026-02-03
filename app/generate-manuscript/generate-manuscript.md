# Generate Manuscript

## Description
Tạo manuscript hoàn chỉnh sau khi user kết thúc brainstorming. User click "Tạo truyện" → API tạo story + queue job → Job chạy song song với user chọn settings → Tạo snapshot.

**Entry point:** User click "Tạo truyện" kết thúc flow `ai-assistant/story-idea-brainstorming.md`

## Flow Overview
```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 1: TRIGGER                                                          │
│   [CLIENT → API]                                                            │
│                                                                             │
│   User click "Tạo truyện" → API tạo story + queue job → Return IDs          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ bookId, jobId
                                 ▼
        ┌────────────────────────┴────────────────────────┐
        │                    PARALLEL                      │
        ▼                                                  ▼
┌───────────────────────────────┐    ┌───────────────────────────────────────┐
│   Phase 2: SETTINGS           │    │   Phase 3: BACKGROUND JOB             │
│   [CLIENT → DB]               │    │   [API BACKGROUND]                    │
│                               │    │                                       │
│   Modal chọn dimension +      │    │   5 steps: Draft → Visual Plan →      │
│   artstyle_id → Save          │    │   Text Refine → Composition → QC      │
└───────────────┬───────────────┘    └───────────────────┬───────────────────┘
                │                                        │
                └────────────────────┬───────────────────┘
                                     │ both completed
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 4: COMPLETION                                                       │
│   [CLIENT]                                                                  │
│                                                                             │
│   Redirect to story editor                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Shared Types

```typescript
// From brainstorming
interface StoryParams {
  targetAudience: 1 | 2 | 3 | 4;
  targetCoreValue: number;              // 1-21: SMALLINT (see books.target_core_value)
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  writingStyle: 1 | 2 | 3;
  eraId: string;
  locationId: string;
}

// Job tracking
type JobStatus = "queued" | "running" | "completed" | "failed";
type StepStatus = "pending" | "running" | "completed" | "failed" | "skipped";

interface StepDetails {
  step1_storyDraft: StepStatus;
  step2_visualPlan: StepStatus;
  step3_textRefinement: StepStatus;
  step4_composition: StepStatus;
  step5_qualityCheck: StepStatus;
}
```

---

## Phase 1: Trigger

### Flow Pattern
`[CLIENT → API]`

### Responsibilities

**CLIENT:**
- Collect params + storyIdea + storyIdeaExplanation từ BrainstormingState
- Send request, handle loading
- Store bookId + jobId on success

**API:**
- Validate request
- Create book với params (book_type = 1, original_language từ user settings, chưa có dimension, artstyle_id)
- Create + queue background job (status = 'queued', params = {storyIdea, storyIdeaExplanation, ...StoryParams})
- Update ai_conversation: book_id, step = 'generating'
- Return bookId + jobId

### Request/Response
```typescript
// POST /api/manuscript/generate
interface Request {
  conversationId: string;
  params: StoryParams;
  storyIdea: string;
  storyIdeaExplanation: string;
}

interface Response {
  bookId: string;
  jobId: string;
}
```

### Exit Condition
Response received → Phase 2 & 3 (parallel)

---

## Phase 2: Settings Selection

### Flow Pattern
`[CLIENT → DB]`

### Description
Modal cho user chọn dimension và artstyle_id. Chạy song song với Phase 3 (background job).

### Responsibilities

**CLIENT:**
- Fetch art_styles từ DB
- Display modal (dimension + art style selector)
- Validate và save to books table

### Dimension Options
| Value | Name | Size |
|-------|------|------|
| 1 | Square 8.5x8.5 | 216x216mm |
| 2 | Portrait 8x10 | 203x254mm |
| 3 | Portrait 6x9 | 152x229mm |
| 4 | Portrait 8.5x11 | 216x279mm |
| 5 | Portrait A4 | 210x297mm |
| 6 | Square 8.25x8.25 | 210x210mm |
| 7 | Square 8x8 | 203x203mm |

### DB Operations
| Operation | Table | Description |
|-----------|-------|-------------|
| SELECT | `art_styles` | Fetch list (id, name, description, image_references) |
| UPDATE | `books` | Save dimension, artstyle_id |

### Exit Condition
User confirms → Settings saved

---

## Phase 3: Background Job

### Flow Pattern
`[API BACKGROUND]`

### Description
Job chạy ngay sau Phase 1, song song với Phase 2 (settings). Job reads input từ `background_jobs.params`, chạy 5 steps, tạo snapshot. Client subscribe/poll để hiển thị progress.

### Job Steps
| Step | Agent | Input | Output |
|------|-------|-------|--------|
| 1 | Story Teller | job.params | characters[], props[], stages[], spreads[], docs[] |
| 2 | Art Director P1 | Step 1 output | visual_description cho entities, images[] |
| 3 | Word Smith | Step 1-2 output | Refined spreads[].textboxes[].text |
| 4 | Art Director P2 | Step 1-3 output | geometry cho images[], textboxes[] (layout ratios) |
| 5 | Tester | Full snapshot | flags[] (nếu có issues) |

**Note:** Step 2 generates visual_description without requiring art_style. Step 4 uses default layout ratios, actual dimensions applied on render.

### Job Table Schema
```sql
CREATE TABLE background_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,           -- 'generate_manuscript'
  book_id UUID REFERENCES books(id),
  status VARCHAR(20) NOT NULL,         -- 'queued', 'running', 'completed', 'failed'
  current_step SMALLINT DEFAULT 0,
  total_steps SMALLINT DEFAULT 5,
  step_details JSONB,
  params JSONB,                        -- { storyIdea, storyIdeaExplanation, ...StoryParams }
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Progress Tracking
- **Primary:** Supabase Realtime subscription on `background_jobs` table
- **Fallback:** Polling GET `/api/jobs/{jobId}` every 3s

### Exit Condition
Job status = 'completed' | 'failed'

---

## Phase 4: Completion

### Flow Pattern
`[CLIENT]`

### Responsibilities
- Cleanup subscriptions
- Update ai_conversation: step = 'book_editing'
- Redirect to `/books/{bookId}/editor`

### Completion Requirements
| Condition | Required |
|-----------|----------|
| Job completed | ✓ |
| Settings saved | ✓ |

**Note:** Block redirect until both conditions met (parallel flow).

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/manuscript/generate` | Phase 1: Create story + queue job |
| GET | `/api/jobs/{jobId}` | Poll job status |

---

## DB Operations Summary

| Phase | Operation | Table | Description |
|-------|-----------|-------|-------------|
| 1 | INSERT | `books` | Create với StoryParams, book_type=1, original_language từ user settings |
| 1 | INSERT | `background_jobs` | Create job với params = {storyIdea, storyIdeaExplanation, ...StoryParams} |
| 1 | UPDATE | `ai_conversations` | Set book_id, step = 'generating' |
| 2 | SELECT | `art_styles` | Fetch options |
| 2 | UPDATE | `books` | Save dimension, artstyle_id |
| 3 | UPDATE | `background_jobs` | Progress (worker) |
| 3 | INSERT | `snapshots` | Create với AI-generated content |
| 4 | UPDATE | `ai_conversations` | Set step = 'book_editing' |

---

## State Management

```typescript
interface GenerateManuscriptState {
  // Phase 1
  trigger: {
    status: "idle" | "loading" | "success" | "error";
    bookId: string | null;
    jobId: string | null;
  };

  // Phase 2
  settings: {
    status: "selecting" | "saving" | "saved";
    selectedDimension: 1 | 2 | 3 | 4 | 5 | 6 | 7 | null;
    selectedArtStyleId: string | null;
    artStyles: Array<{
      id: string;
      name: string;
      description: string;
      image_references: { title: string; media_url: string }[];
    }>;
  };

  // Phase 3
  job: {
    status: JobStatus;
    currentStep: number;
    stepDetails: StepDetails;
  };

  // Phase 4
  completion: {
    status: "pending" | "ready" | "redirecting";
    summary?: {
      title: string;
      spreadCount: number;
      characterCount: number;
      flagCount: number;
    };
  };

  error: string | null;
}
```

---

## Error Handling

| Phase | Error | Action |
|-------|-------|--------|
| 1 | Network/DB fail | Retry, preserve brainstorming data |
| 2 | Settings save fail | Retry |
| 3 | Step fail | Save partial, offer continue/restart |
| 4 | — | — |

---

## Edge Cases

| Case | Solution |
|------|----------|
| Browser close during job | Job continues. On return, check for incomplete jobs |
| Settings not saved, job done | Block redirect, show settings modal |
| Partial job failure | Save partial result, offer "Continue" / "Restart" |
| Duplicate "Tạo truyện" click | Return existing bookId + jobId, don't create duplicates |
| art_styles empty | Show error, block settings save |

---

## Conversation Step Transitions

```
brainstorming → generating → book_editing
     │              │            │
     │              │            └─ Phase 4
     │              └─ Phase 1
     └─ Before trigger
```
