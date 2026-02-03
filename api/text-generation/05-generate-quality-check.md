# generate-quality-check

## Description
**Tester Agents** - Kiểm tra chất lượng nội dung qua nhiều góc nhìn: Story Consistency, Plot, và Age Appropriateness.

## DB Schema Dependencies

### Tables Referenced
- `books`: Đọc metadata (title, target_audience, target_core_value)
- `snapshots`: Đọc toàn bộ docs[], characters[], props[], stages[], spreads[]
- `flags`: TẠO MỚI các flag records cho issues tìm thấy

### Fields Used
- `flags.id`, `flags.user_id`, `flags.book_id`, `flags.title`, `flags.content`, `flags.type`, `flags.status`
- `books.title`, `books.target_audience`, `books.target_core_value`

**Note:** Không đọc `books.artstyle_id` - chưa được set ở thời điểm này (Step 5 chạy trong background job trước Phase 2).

## Parameters
```typescript
interface GenerateQualityCheckParams {
  bookId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateQualityCheckResult {
  success: boolean;
  testResults: {
    storyConsistency: TestResult;
    plot: TestResult;
    ageAppropriateness: TestResult;
  };
  flags: FlagItem[];              // Aggregated issues từ tất cả tests
  overallScore: number;           // 0-100
  recommendation: "approved" | "needs_revision" | "major_issues";
}

interface TestResult {
  testerType: string;
  passed: boolean;
  score: number;                  // 0-100
  issues: Issue[];
  suggestions: string[];
}

interface Issue {
  severity: number;               // 0: suggestion, 1: minor, 2: major, 3: critical
  category: string;
  location: string;               // e.g., "character:@miu_cat" or "spread:3"
  description: string;
  suggestion: string;
}

// Map với bảng flags trong DB (simplified format)
interface FlagItem {
  title: string;                  // Format: "[Tester] [Severity] Category: Brief description"
  content: string;                // Format: "Location: xxx\n\nIssue: xxx\n\nSuggestion: xxx"
  type: number;                   // 0: consistency, 1: plot, 2: age_inappropriate, 3: other
  status: number;                 // 0: open, 1: in_progress, 2: resolved, 3: ignored (default: 0)
}

// Enum mappings for reference
enum FlagType {
  CONSISTENCY = 0,
  PLOT = 1,
  AGE_INAPPROPRIATE = 2,
  OTHER = 3
}

enum FlagStatus {
  OPEN = 0,
  IN_PROGRESS = 1,
  RESOLVED = 2,
  IGNORED = 3
}
```

## Prompt

> **DB Template Names:**
> - System: `TESTER_SYSTEM`
> - User: `QUALITY_CHECK_USER_TEMPLATE`

### System Prompt
```
You are a team of quality assurance testers for children's picture books. You evaluate content from multiple perspectives to ensure the highest quality.

You consist of three specialized testers:

1. **Story Consistency Tester**: Ensures all story elements are internally consistent
   - Character personality vs. actions in spreads
   - Character appearance vs. visual descriptions
   - Prop usage consistency
   - Stage descriptions vs. spread usage
   - Timeline/sequence logic
   - Character relationships consistency

2. **Plot Tester**: Evaluates narrative structure, logic, and lesson effectiveness
   - Plot holes detection
   - Logical contradictions
   - Pacing issues
   - Character arc completion
   - Moral/lesson clarity and depth
   - Emotional journey coherence
   - Ending satisfaction

3. **Age Appropriateness Tester**: Reviews content from both child and parent perspectives
   - Vocabulary complexity for reading level
   - Sentence length appropriateness
   - Theme suitability
   - Scary/disturbing content assessment
   - Concept understandability
   - Entertainment value for child
   - Educational value for parent

Your evaluation is thorough but constructive. You identify issues and provide actionable suggestions for improvement.
```

### User Prompt Template
```
Evaluate the following picture book manuscript:

## STORY METADATA
- Title: {%title%}
- Target Audience: {%target_audience%}
- Core Values: {%target_core_value%}

## DOCUMENTS
{%docs_text%}

## CHARACTERS
{%characters_json%}

## PROPS
{%props_json%}

## STAGES
{%stages_json%}

## SPREADS
{%spreads_json%}

---

## AGE-SPECIFIC GUIDELINES
{%age_guidelines%}

---

## EVALUATION TASKS

### 1. STORY CONSISTENCY TEST
Check for:
- [ ] Character personality matches their actions
- [ ] Character appearance consistent across all visual descriptions
- [ ] Props used correctly according to their defined purpose
- [ ] Stages described consistently
- [ ] Timeline makes sense
- [ ] Character relationships are consistent
- [ ] No contradictions between docs and data structures

### 2. PLOT TEST
Check for:
- [ ] No plot holes
- [ ] No logical contradictions
- [ ] Pacing is appropriate for length
- [ ] Character arcs are complete
- [ ] Moral lesson is clear but not preachy
- [ ] Emotional journey is coherent
- [ ] Ending is satisfying

### 3. AGE APPROPRIATENESS TEST
Based on the AGE-SPECIFIC GUIDELINES above, check for:
- [ ] Vocabulary matches reading level
- [ ] Sentence length appropriate
- [ ] Themes appropriate
- [ ] No scary/disturbing content (unless intended and handled well)
- [ ] Concepts are understandable
- [ ] Entertainment value for child
- [ ] Value for parent (educational, bonding opportunity)

---

## OUTPUT FORMAT

Return JSON:
{
  "testResults": {
    "storyConsistency": {
      "testerType": "Story Consistency",
      "passed": true/false,
      "score": 85,
      "issues": [
        {
          "severity": 2,
          "category": "character_consistency",
          "location": "character:@miu_cat",
          "description": "In spread 3, Miu acts bravely despite being described as fearful",
          "suggestion": "Add internal dialogue showing Miu overcoming fear"
        }
      ],
      "suggestions": ["Consider adding more foreshadowing for Miu's character growth"]
    },
    "plot": {
      "testerType": "Plot",
      "passed": true/false,
      "score": 90,
      "issues": [...],
      "suggestions": [...]
    },
    "ageAppropriateness": {
      "testerType": "Age Appropriateness",
      "passed": true/false,
      "score": 88,
      "issues": [...],
      "suggestions": [...]
    }
  },
  "flags": [
    {
      "title": "[Story Consistency] [MAJOR] character_consistency: Miu behavior inconsistent",
      "content": "Location: character:@miu_cat, spread:3\n\nIssue: In spread 3, Miu acts bravely despite being described as fearful in character bible.\n\nSuggestion: Add internal dialogue showing Miu overcoming fear.",
      "type": 0,
      "status": 0
    }
  ],
  "overallScore": 87,
  "recommendation": "approved" | "needs_revision" | "major_issues"
}

## FLAGS FORMAT

**Title Format:** `[Tester] [Severity] Category: Brief description`
- **Tester:** Story Consistency | Plot | Age Appropriateness
- **Severity:** SUGGESTION | MINOR | MAJOR | CRITICAL
- **Category:** character_consistency | plot_hole | vocabulary_level | etc.

**Content Format:**
Location: [target_type]:[target_id], [additional_context]

Issue: [Detailed description of the problem]

Suggestion: [Actionable recommendation to fix the issue]

## SCORING GUIDELINES
- 90-100: Excellent, ready to publish
- 80-89: Good, minor revisions recommended
- 70-79: Acceptable, revisions needed
- 60-69: Needs work, several issues to address
- Below 60: Major issues, significant revision required

## RECOMMENDATION RULES
- "approved": score >= 80 AND no critical issues
- "needs_revision": score >= 60 AND score < 80 OR has major issues
- "major_issues": score < 60 OR has critical issues
```

## Age Guidelines Mapping

| Value | Name | Guidelines |
|-------|------|------------|
| 1 | Kindergarten (2-3 tuổi) | Vocabulary: very simple, 3-5 words per sentence. Repetition and rhythm important. No scary/intense content. Basic concepts only. Very short attention span. |
| 2 | Preschool (4-5 tuổi) | Vocabulary: simple, 5-10 words per sentence. Age-appropriate themes. Minimal scary content. Concept complexity appropriate. Short attention span. |
| 3 | Primary (6-8 tuổi) | Vocabulary: varied with context clues. Reading level match. Emotional intensity appropriate. Educational value. Can handle compound sentences. |
| 4 | Middle Grade (9+ tuổi) | Vocabulary: sophisticated. Depth of themes. Complex emotions allowed. Subtext and nuance. |

**Parent perspective (all ages):**
- Hidden meanings that might be inappropriate
- Values alignment
- Educational merit
- Entertainment value
- Conversation starters

## Flow
```
1. Validate input parameters (bookId, snapshotId)

2. Lấy prompt templates từ DB:
   - TESTER_SYSTEM → system prompt + model
   - QUALITY_CHECK_USER_TEMPLATE → user prompt template

3. Lấy book metadata từ DB (bookId):
   - title
   - target_audience (SMALLINT 1-4)
   - target_core_value (SMALLINT 1-21)
   Note: KHÔNG lấy artstyle_id (chưa được set)

4. Lấy full snapshot data từ DB (snapshotId):
   - docs[], characters[], props[], stages[], spreads[]

5. Build prompt variables:
   a. title = book.title
   b. target_audience = map_target_audience_to_text(book.target_audience)
      // 1 → "Kindergarten (2-3 tuổi)"
      // 2 → "Preschool (4-5 tuổi)"
      // 3 → "Primary (6-8 tuổi)"
      // 4 → "Middle Grade (9+ tuổi)"
   c. target_core_value = map_core_value_to_text(book.target_core_value)
      // 1 → "Dũng cảm", 2 → "Quan tâm", etc.
   d. age_guidelines = get_age_guidelines(book.target_audience)
      // Lấy từ Age Guidelines Mapping table
   e. docs_text = format_docs(snapshot.docs)
      // Format: "### {type}: {title}\n{content}..."
   f. characters_json = json.dumps(snapshot.characters)
   g. props_json = json.dumps(snapshot.props)
   h. stages_json = json.dumps(snapshot.stages)
   i. spreads_json = json.dumps(snapshot.spreads)

6. Render user prompt template với variables

7. Call LLM

8. Parse JSON response

9. Lưu flags[] vào DB (bảng flags):
   - user_id = system_user_id
   - book_id = bookId
   - title, content, type, status từ LLM output
   - type: sử dụng trực tiếp từ LLM (0-3)
   - status: default = 0 (open)

10. Return test results
```

## Error Handling
- LLM response không parse được → Retry 1 lần
- Vẫn fail → Return error với message
- Flag creation fail → Log warning, continue với các flags còn lại

## Notes
- Simplified design: Gộp các thông tin (tester, severity, category, suggestion, location) vào `title` và `content`
- `title` format: `[Tester] [Severity] Category: Brief description`
- `content` format: Multi-line text với Location, Issue, Suggestion sections
- DB flag type và LLM output type dùng chung enum (0-3)
