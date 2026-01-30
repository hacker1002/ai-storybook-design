# generate-manuscript

## Description
**Orchestrator API** - Tạo story record và queue background job cho việc generate manuscript. Job chạy 5 steps tuần tự trong background.

**Entry point:** POST `/api/manuscript/generate` - được gọi khi user click "Tạo truyện" từ brainstorming flow.

## DB Schema Dependencies

### Tables Referenced
- `stories`: Tạo record mới với StoryParams (chưa có dimension, artstyle_id)
- `background_jobs`: Tạo job record với params và track progress
- `ai_conversations`: Update story_id và step
- `eras`: Truy vấn era info
- `locations`: Truy vấn location info

## Shared Types

```typescript
// From brainstorming
interface StoryParams {
  targetAudience: 1 | 2 | 3 | 4;       // 1: kindergarten, 2: preschool, 3: primary, 4: middle grade
  targetCoreValue: number;              // 1-21: Dũng cảm, Quan tâm, etc. (see stories.target_core_value)
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;  // Narrative, Lullaby, Concept, Non-fiction, Early Reader, Wordless
  contentGenre: number;                 // 1-11: Mystery, Fantasy, etc.
  writingStyle: 1 | 2 | 3;             // 1: Narrative, 2: Rhyming, 3: Humorous Fiction
  eraId: string;                        // UUID → eras
  locationId: string;                   // UUID → locations
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

## Parameters
```typescript
// POST /api/manuscript/generate
interface GenerateManuscriptRequest {
  conversationId: string;               // AI conversation ID
  params: StoryParams;
  storyIdea: string;                    // "Một chú mèo con tên Miu lạc đường..."
  storyIdeaExplanation: string;         // Chi tiết giải thích ý tưởng
}
```

## Result
```typescript
interface GenerateManuscriptResponse {
  storyId: string;                      // UUID của story được tạo
  jobId: string;                        // UUID của background job
}
```

## Flow (Phase 1: Trigger)

```
1. Validate input parameters

2. Create story record:
   - title = "" (empty, user đặt tên sau)
   - owner_id = user_id (lấy từ API access key)
   - book_type = 1 (sách tranh)
   - original_language = user.settings.language
   - target_audience, target_core_value, format_genre, content_genre
   - writing_style, era_id, location_id
   - dimension = NULL (chọn ở Phase 2)
   - artstyle_id = NULL (chọn ở Phase 2)

3. Create background_jobs record:
   - type = 'generate_manuscript'
   - story_id = story.id
   - status = 'queued'
   - current_step = 0
   - total_steps = 5
   - step_details = { step1: 'pending', step2: 'pending', ... }
   - params = { storyIdea, storyIdeaExplanation, ...StoryParams }

4. Update ai_conversations:
   - story_id = story.id
   - step = 'generating'

5. Queue job for background processing

6. Return { storyId, jobId }
```

## Background Job Flow (Phase 3)

```
Job Runner reads params from background_jobs.params

Step 1: Story Draft (Story Teller)
├─ Input: job.params (storyIdea, StoryParams)
├─ Output: docs[], characters[], props[], stages[], spreads[]
├─ Save to: snapshot
└─ Note: Chưa có dimension, artstyle_id

Step 2: Visual Plan (Art Director P1)
├─ Input: snapshot từ Step 1
├─ Output: visual_description cho entities, images[]
└─ Note: Tạo visual_description KHÔNG cần art_style
         Art style applied later during image generation

Step 3: Text Refinement (Word Smith)
├─ Input: snapshot từ Step 1-2
├─ Output: Refined spreads[].textboxes[].text
└─ Note: Dựa trên target_audience để adjust

Step 4: Composition (Art Director P2)
├─ Input: snapshot từ Step 1-3
├─ Output: geometry cho images[], textboxes[]
└─ Note: Uses default layout ratios, actual dimensions applied on render

Step 5: Quality Check (Tester)
├─ Input: Full snapshot
├─ Output: flags[] (nếu có issues)
└─ Note: Final validation
```

## Step Dependencies
```
Step 1 (Story Draft)
    ↓
Step 2 (Visual Plan) ─── Không cần art_style
    ↓
Step 3 (Text Refinement) ─── Cần target_audience
    ↓
Step 4 (Composition) ─── Uses default layout ratios
    ↓
Step 5 (Quality Check) ─── Phụ thuộc tất cả steps trước
```

## Progress Tracking

### Client Subscription
- **Primary:** Supabase Realtime subscription on `background_jobs` table
- **Fallback:** Polling GET `/api/jobs/{jobId}` every 3s

### Job Status Response
```typescript
interface JobStatusResponse {
  id: string;
  status: JobStatus;
  currentStep: number;
  totalSteps: number;
  stepDetails: StepDetails;
  errorMessage?: string;
}
```

## Error Handling
| Step | Error | Action |
|------|-------|--------|
| 1 | Story Draft fail | status = 'failed', không tạo snapshot |
| 2 | Visual Plan fail | status = 'failed', giữ Step 1 data |
| 3 | Text Refinement fail | status = 'failed', giữ Step 1-2 data |
| 4 | Composition fail | status = 'failed', giữ Step 1-3 data |
| 5 | QC fail | status = 'failed', giữ tất cả data |

## Notes
- `dimension` và `artstyle_id` được chọn ở Phase 2 (Settings), KHÔNG cần cho job processing
- Step 4 (Composition) sử dụng layout ratios mặc định, actual dimensions applied on render
- Step 2 tạo `visual_description` chung, art_style được apply khi generate image
- Client cần block redirect until both: job completed + settings saved
