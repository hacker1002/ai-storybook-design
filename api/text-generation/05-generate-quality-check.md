# generate-quality-check

## Description
**Tester Agents** - Kiểm tra chất lượng nội dung qua nhiều góc nhìn: Story Consistency, Plot, và Age Appropriateness.

## DB Schema Dependencies

### Tables Referenced
- `story`: Đọc metadata (title, target_audience, target_core_value, artstyle_id)
- `snapshot`: Đọc toàn bộ docs[], characters[], props[], stages[], spreads[]
- `flags`: TẠO MỚI các flag records cho issues tìm thấy
- `art_styles`: Đọc description

### Fields Used
- `flags.id`, `flags.user_id`, `flags.story_id`, `flags.title`, `flags.content`, `flags.type`, `flags.status`
- `story.title`, `story.target_audience`, `story.target_core_value`, `story.artstyle_id`

## Parameters
```typescript
interface GenerateQualityCheckParams {
  storyId: string;
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
                                  // Example: "[Story Consistency] [MAJOR] character_consistency: Miu behavior inconsistent"
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

## Sub-Agents

### 4.1 Story Consistency Tester
Kiểm tra tính nhất quán của characters, props, stages với cốt truyện và visual design.

**Checks:**
- Character personality vs. actions in spreads
- Character appearance vs. visual descriptions
- Prop usage consistency
- Stage descriptions vs. spread usage
- Timeline/sequence logic
- Character relationships consistency

### 4.2 Plot Tester
Kiểm tra cốt truyện có vấn đề logic hoặc narrative.

**Checks:**
- Plot holes (lỗ hổng cốt truyện)
- Contradictions (mâu thuẫn)
- Pacing issues (nhịp truyện)
- Character arc completion
- Moral/lesson clarity and depth
- Emotional journey coherence
- Ending satisfaction

### 4.3 Age Appropriateness Tester (Parent + Child perspective)
Kiểm tra nội dung phù hợp với độ tuổi target.

**Checks based on audience:**

**kids-3-6:**
- Vocabulary complexity (should be simple)
- Sentence length (5-10 words)
- Scary/intense content
- Concept complexity
- Attention span consideration

**kids-7-12:**
- Age-appropriate themes
- Reading level match
- Emotional intensity appropriate
- Educational value

**teens:**
- Depth of themes
- Relatable content
- Not too childish, not too mature

**Parent perspective:**
- Hidden meanings that might be inappropriate
- Values alignment
- Educational merit
- Entertainment value
- Conversation starters

## Prompt

> **DB Template Names:**
> - System: `TESTER_SYSTEM`
> - User: `QUALITY_CHECK_USER_TEMPLATE`

### System Prompt
```
You are a team of quality assurance testers for children's picture books. You evaluate content from multiple perspectives to ensure the highest quality.

You consist of three specialized testers:

1. **Story Consistency Tester**: Ensures all story elements are internally consistent
2. **Plot Tester**: Evaluates narrative structure, logic, and lesson effectiveness
3. **Age Appropriateness Tester**: Reviews content from both child and parent perspectives

Your evaluation is thorough but constructive. You identify issues and provide actionable suggestions for improvement.
```

### User Prompt Template
```
Evaluate the following picture book manuscript:

## STORY METADATA
- Title: {%title%}
- Target Audience: {%target_audience%}
- Art Style: {%art_style_description%}
- Core Values: {%target_core_value%}

## DOCUMENTS
{ // from snapshot.docs[]:
- type: "manuscript", content: "..."
- type: "story_structure", content: "..."
- type: "artistic_imagery", content: "..."
- type: "moral_lesson", content: "..."
}
{%docs_text%}

## CHARACTERS
{ // from snapshot.characters[] with visual_description:
- key: "@miu_cat", name: "Miu", visual_description: "...", basic_info: {...}, personality: {...}
- ...
}
{%characters_json%}

## PROPS
{ // from snapshot.props[] with visual_description:
- key: "@red_bow", name: "Chiếc nơ đỏ", visual_description: "...", type: "narrative"
- ...
}
{%props_json%}

## STAGES
{ // from snapshot.stages[] with visual_description:
- key: "@forest_1", name: "Khu rừng", visual_description: "..."
- ...
}
{%stages_json%}

## SPREADS
{ // from snapshot.spreads[] with images[]:
- number: 1, manuscript: "...", images: [...], textboxes: [...]
- ...
}
{%spreads_json%}

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
For target audience "{audience}":
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

## SIMPLIFIED FLAGS FORMAT

**Title Format:** `[Tester] [Severity] Category: Brief description`
- **Tester:** Story Consistency | Plot | Age Appropriateness
- **Severity:** SUGGESTION | MINOR | MAJOR | CRITICAL
- **Category:** character_consistency | plot_hole | vocabulary_level | etc.
- **Brief description:** Short summary of the issue

**Content Format:** Multi-line text with three sections:
```
Location: [target_type]:[target_id], [additional_context]

Issue: [Detailed description of the problem]

Suggestion: [Actionable recommendation to fix the issue]
```

**Example:**
{
  "title": "[Plot] [CRITICAL] plot_hole: Missing resolution for conflict",
  "content": "Location: spread:5-7\n\nIssue: The conflict between Miu and the forest guardian is introduced in spread 5 but never resolved. The story jumps to the ending without addressing this plotline.\n\nSuggestion: Add a spread showing Miu apologizing to the guardian or finding a way to make amends.",
  "type": 1,
  "status": 0
}

## ENUM MAPPINGS
- **type**: 0=consistency, 1=plot, 2=age_inappropriate, 3=other
- **status**: 0=open, 1=in_progress, 2=resolved, 3=ignored

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

## Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "TESTER_SYSTEM" → system prompt + model
   - Query `prompt_templates` với name = "QUALITY_CHECK_USER_TEMPLATE" → user prompt template
3. Lấy full snapshot data từ DB (docs, characters, props, stages, spreads)
4. Lấy story metadata (title, target_audience, target_core_value, artstyle_id → art_styles.description)
5. Render user prompt template với variables:
   - title, target_audience, art_style_description, target_core_value
   - docs_text, characters_json, props_json, stages_json, spreads_json
6. Call LLM với:
   - system prompt content
   - rendered user prompt
   - model từ prompt_templates (dynamic, không hardcode)
7. Parse JSON response
8. Lưu flags[] vào DB (bảng flags)
9. Return test results
```


**Notes:**
- Simplified design: Gộp các thông tin (tester, severity, category, suggestion, location, target_type, target_id) vào `title` và `content`
- `title` format: `[Tester] [Severity] Category: Brief description`
- `content` format: Multi-line text với Location, Issue, Suggestion sections
