# Generate Manuscript

## Description
Tạo manuscript hoàn chỉnh sau khi user kết thúc brainstorming. User click "Tạo truyện" → API tạo story + snapshot + background job → Client hiển thị progress + modal chọn settings → Background job chạy 5 steps.

**Entry point:** User click "Tạo truyện" kết thúc flow `ai-assistant/story-idea-brainstorming.md`

## Flow Overview
```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Phase 1: TRIGGER                                                          │
│   [CLIENT → API]                                                            │
│                                                                             │
│   User click "Tạo truyện" → API tạo story + snapshot + job → Return IDs     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ storyId, snapshotId, jobId
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
  targetAudience: 1 | 2 | 3;
  targetCoreValue: string;
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
  contentGenre: number;
  storyIdea: string | null;
  writingStyle: 1 | 2 | 3;
  eraId: string;
  locationId: string;
}

interface StoryDoc {
  type: 0 | 1 | 2;               // 0: system, 1: user edit, 2: user read-only
  title: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  content: string;
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
- Collect params + docs từ BrainstormingState
- Send request, handle loading
- Store IDs on success

**API:**
- Validate request
- Create story (chưa có dimension, artstyle_id)
- Create snapshot với docs[]
- Create + queue background job
- Update ai_conversation: story_id, step = 'generating'
- Return IDs

### Request/Response
```typescript
// POST /api/manuscript/generate
interface Request {
  conversationId: string;
  params: StoryParams;
  docs: StoryDoc[];
}

interface Response {
  storyId: string;
  snapshotId: string;
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
Modal cho user chọn dimension và artstyle_id. Chạy song song với Phase 3.

### Responsibilities

**CLIENT:**
- Fetch art_styles từ DB
- Display modal (dimension + art style selector)
- Validate và save to story table

### Dimension Options
| Value | Name | Size | Spreads | Words/Spread |
|-------|------|------|---------|--------------|
| 1 | Square | 20x20cm | 12 | 30-50 |
| 2 | A4 Landscape | 29.7x21cm | 16 | 40-60 |
| 3 | A4 Portrait | 21x29.7cm | 20 | 50-80 |

### DB Operations
| Operation | Table | Description |
|-----------|-------|-------------|
| SELECT | `art_styles` | Fetch list (id, name, description, image_references) |
| UPDATE | `stories` | Save dimension, artstyle_id |

### Exit Condition
User confirms → Settings saved

---

## Phase 3: Background Job

### Flow Pattern
`[API BACKGROUND]`

### Description
Background job chạy 5 steps. Client subscribe/poll để hiển thị progress.

### Job Steps
| Step | Agent | Output |
|------|-------|--------|
| 1 | Story Teller | characters[], props[], stages[], spreads[] |
| 2 | Art Director P1 | visual_description cho entities, images[] |
| 3 | Word Smith | Refined spreads[].textboxes[].text |
| 4 | Art Director P2 | geometry cho images[], textboxes[] |
| 5 | Tester | flags[] (nếu có issues) |

### Job Table Schema
```sql
CREATE TABLE background_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,           -- 'generate_manuscript'
  story_id UUID REFERENCES stories(id),
  status VARCHAR(20) NOT NULL,         -- 'queued', 'running', 'completed', 'failed'
  current_step SMALLINT DEFAULT 0,
  total_steps SMALLINT DEFAULT 5,
  step_details JSONB,
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
- Update ai_conversation: step = 'complete'
- Redirect to `/stories/{storyId}/editor`

### Completion Requirements
| Condition | Required |
|-----------|----------|
| Job completed | ✓ |
| Settings saved | ✓ |

**Note:** Block redirect until both conditions met.

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/manuscript/generate` | Trigger generation |
| GET | `/api/jobs/{jobId}` | Poll job status |

---

## DB Operations Summary

| Phase | Operation | Table | Description |
|-------|-----------|-------|-------------|
| 1 | INSERT | `stories` | Create với params |
| 1 | INSERT | `snapshots` | Create với docs[] |
| 1 | INSERT | `background_jobs` | Create job |
| 1 | UPDATE | `ai_conversations` | Set story_id, step |
| 2 | SELECT | `art_styles` | Fetch options |
| 2 | UPDATE | `stories` | Save settings |
| 3 | UPDATE | `background_jobs` | Progress (worker) |
| 3 | UPDATE | `snapshots` | Content (worker) |
| 4 | UPDATE | `ai_conversations` | Set step = 'complete' |

---

## State Management

```typescript
interface GenerateManuscriptState {
  // Phase 1
  trigger: {
    status: "idle" | "loading" | "success" | "error";
    storyId: string | null;
    snapshotId: string | null;
    jobId: string | null;
  };

  // Phase 2
  settings: {
    status: "selecting" | "saving" | "saved";
    selectedDimension: 1 | 2 | 3 | null;
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
| Settings not saved, job done | Block redirect, show reminder |
| Partial job failure | Save partial result, offer "Continue" / "Restart" |
| Duplicate "Tạo truyện" click | Return existing IDs, don't create duplicates |
| art_styles empty | Show error, block creation |

---

## Conversation Step Transitions

```
brainstorming → generating → complete
     │              │            │
     │              │            └─ Phase 4
     │              └─ Phase 1
     └─ Before trigger
```
