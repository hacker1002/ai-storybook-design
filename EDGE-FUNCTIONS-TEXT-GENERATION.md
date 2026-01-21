# Edge Functions - Text Generation

Tài liệu thiết kế chi tiết các Edge Functions cho AI Text Generation trong AI Storybook Canvas.

---

## Mục lục

### Manuscript Generation (Multi-Step)
1. [generate-manuscript](#1-generate-manuscript) - Orchestrator
2. [generate-story-draft](#2-generate-story-draft) - Story Teller Agent
3. [generate-art-direction](#3-generate-art-direction) - Art Director Agent
4. [generate-quality-check](#4-generate-quality-check) - Tester Agents

### Visual Description Generation
5. [generate-visual-description-character](#5-generate-visual-description-character)
6. [generate-visual-description-prop](#6-generate-visual-description-prop)
7. [generate-visual-description-stage](#7-generate-visual-description-stage)
8. [generate-visual-description-spread](#8-generate-visual-description-spread)

### Other
9. [translate-content](#9-translate-content)

---

## 1. generate-manuscript

### Description
**Orchestrator function** - Điều phối việc tạo manuscript hoàn chỉnh qua 3 bước tuần tự. Function này gọi lần lượt 3 step functions và quản lý flow tổng thể.

### Parameters
```typescript
interface GenerateManuscriptParams {
  storyIdea: string;              // "Một chú mèo con tên Miu lạc đường..."
  attributes: {
    storyType: string[];          // ["adventure", "drama"]
    audience: string;             // "kids-3-6" | "kids-7-12" | "teens"
    length: "short" | "medium" | "long";
    artStyleId: string;           // UUID của art_style trong DB
  };
  language?: string;              // "vi" | "en" - nếu không truyền, dùng "vi"
  options?: {
    skipStep2?: boolean;          // Bỏ qua Art Direction (default: false)
    skipStep3?: boolean;          // Bỏ qua Quality Check (default: false)
  };
}
```

### Result
```typescript
interface GenerateManuscriptResult {
  storyId: string;                // UUID của story được tạo
  snapshotId: string;             // UUID của snapshot được tạo
  status: "completed" | "partial";
  stepsCompleted: {
    step1: boolean;               // Story Analysis
    step2: boolean;               // Art Direction
    step3: boolean;               // Quality Check
  };
  summary: {
    title: string;
    spreadCount: number;
    characterCount: number;
    propCount: number;
    stageCount: number;
    flagCount: number;            // Số lượng issues từ Step 3
  };
}
```

### Flow
```
1. Validate input parameters
2. Gọi generate-story-draft (truyền storyIdea, attributes, language)
   → Tạo Story + Snapshot trong DB
   → Lưu: docs[], characters[], props[], stages[], spreads[]
3. Nếu không skipStep2:
   Gọi generate-art-direction (truyền storyId, snapshotId)
   → Update: characters[].visual_description, props[], stages[], spreads[].images[]
4. Nếu không skipStep3:
   Gọi generate-quality-check (truyền storyId, snapshotId)
   → Lưu: flags[] (nếu có issues)
5. Return result
```

### Error Handling
- Nếu Step 1 fail → Return error, không tạo story
- Nếu Step 2 fail → Return partial result, vẫn giữ Step 1 data
- Nếu Step 3 fail → Return partial result, vẫn giữ Step 1 + Step 2 data

---

## 2. generate-story-draft

### Description
**Story Teller Agent** - Phân tích ý tưởng và tạo khung truyện ban đầu bao gồm: character bible, prop bible, stage bible, phân tích nghệ thuật, và phân chia nội dung từng spread. Function này cũng tạo Story và Snapshot record trong DB.

### Parameters
```typescript
interface GenerateStoryAnalysisParams {
  storyIdea: string;              // "Một chú mèo con tên Miu lạc đường..."
  attributes: {
    storyType: string[];          // ["adventure", "drama"]
    audience: string;             // "kids-3-6" | "kids-7-12" | "teens"
    length: "short" | "medium" | "long";
    artStyleId: string;           // UUID của art_style trong DB
  };
  language?: string;              // "vi" | "en" - nếu không truyền, dùng "vi"
}
```

### Result
```typescript
interface GenerateStoryAnalysisResult {
  success: boolean;
  storyId: string;                // UUID của story được tạo
  snapshotId: string;             // UUID của snapshot được tạo
  data: LLMOutput;                // Output từ LLM (đã parse)
}

// Output JSON từ LLM
interface LLMOutput {
  metadata: {
    title: string;
    summary: string;
    targetCoreValue: string[];
  };
  docs: DocItem[];                // 3 documents: structure, imagery, lesson
  characters: CharacterItem[];
  props: PropItem[];
  stages: StageItem[];
  spreads: SpreadItem[];
}

interface DocItem {
  type: "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson";
  title: string;
  content: string;                // Markdown content
}
```

### Prompt

#### System Prompt
```
You are an expert Story Teller specializing in children's picture books. Your role is to deeply analyze story ideas and create comprehensive story frameworks.

Your expertise includes:
- Child psychology and age-appropriate storytelling
- Character development with multi-dimensional personalities
- Story structure (three-act, hero's journey, etc.)
- Literary devices and symbolic imagery
- Moral lessons that resonate without being preachy

You output structured JSON that can be directly used by the system.
```

#### User Prompt Template
```
Analyze the following story idea and create a complete story framework:

## STORY IDEA
{storyIdea}

## ATTRIBUTES
- Story Types: {storyType}
- Target Audience: {audience}
- Length: {length}
- Art Style Reference: {artStyleDescription}
- Language: {language}

## LENGTH SPECIFICATIONS
{lengthSpec based on attributes.length}

## RESOURCES TABLE 

### asset_categories:
{ // list from DB as below:
- uuid: id, name: "Abc", type: "human", description: ""
- ...
}

---

## OUTPUT FORMAT

Return a JSON object with the following structure:

{
  "metadata": {
    "title": "...",
    "summary": "...",
    "targetCoreValue": ["..."]
  },
  "docs": [...],
  "characters": [...],
  "props": [...],
  "stages": [...],
  "spreads": [...]
}

---

## DETAILED REQUIREMENTS

### 1. metadata
- **title**: Tiêu đề truyện
- **summary**: Tóm tắt 2-3 câu về nội dung và bài học
- **targetCoreValue**: Mảng các giá trị cốt lõi (ví dụ: ["kindness", "courage", "friendship"])

### 2. docs[] - 4 documents
Mỗi document là một object với cấu trúc:
{
  "type": "manuscript" | "story_structure" | "artistic_imagery" | "moral_lesson",
  "title": "...",
  "content": "..." // Markdown content
}

**Document 1: manuscript** - Cốt truyện hoàn chỉnh
- Nội dung truyện đầy đủ từ đầu đến cuối
- Viết theo dạng văn xuôi, dễ đọc
- Bao gồm lời thoại của nhân vật
- Phù hợp với reading level của target audience
- Đây là bản gốc để tham khảo khi viết text cho từng spread

**Document 2: story_structure** - Cấu trúc truyện
- Three-act structure breakdown
- Conflict types (Internal/External)
- Pacing và rhythm
- Key plot points

**Document 3: artistic_imagery** - Hình tượng nghệ thuật
- Metaphors và symbols
- Visual motifs
- Color symbolism
- Recurring imagery

**Document 4: moral_lesson** - Bài học đạo đức
- Main theme và sub-themes
- How lesson is conveyed (subtle, not preachy)
- Emotional journey của độc giả
- Age-appropriate messaging

### 3. characters[] - Danh sách nhân vật
Mỗi character là một object:
{
  "mention_name": "miu_cat",
  "name": "Miu",
  "basic_info": {
    "description": "Mô tả ngắn về nhân vật",
    "gender": "male",
    "age": "3 tuổi (tương đương trẻ 5 tuổi)",
    "category_id": "uuid-of-category",  // FK → asset_categories
    "role": "main character"
  },
  "personality": {
    "core_essence": "Bản chất cốt lõi",
    "flaws": "Điểm yếu",
    "emotions": "Cách biểu đạt cảm xúc",
    "reactions": "Phản ứng hành vi đặc trưng",
    "desires": "Mong muốn sâu xa",
    "likes": "Sở thích",
    "fears": "Nỗi sợ",
    "contradictions": "Mâu thuẫn nội tâm"
  },
  "appearance": {
    "height": 30,
    "hair": "Mô tả lông/tóc",
    "eyes": "Mô tả mắt",
    "face": "Mô tả khuôn mặt",
    "build": "Thể hình",
    "outfit": "Trang phục (nếu có)"
  }
}

### 4. props[] - Danh sách đạo cụ
{
  "mention_name": "red_bow",
  "name": "Chiếc nơ đỏ",
  "category_id": "uuid-of-category",  // FK → asset_categories 
  "type": "narrative | anchor"
}

**Giải thích:**
- **type**:
  - `narrative`: Đồ vật dẫn chuyện, tương tác với character
  - `anchor`: Đồ vật nằm trong stages, tạo sự nhất quán

### 5. stages[] - Danh sách bối cảnh
{
  "mention_name": "forest_1",
  "name": "Khu rừng bí ẩn",
  "location_id": "uuid-of-location"  // FK → locations (địa danh cụ thể)
}

### 6. spreads[] - Danh sách các trang
{
  "number": 1,
  "left_page": { "number": 1, "type": "front_matter | story | back_matter" },
  "right_page": { "number": 2, "type": "story" },
  "manuscript": "trang 1: <cốt truyện trang 1>, trang 2: <cốt truyện trang 2>",
  "textboxes": [{
    "order": 1,
    "language": [{ "text": "Nội dung văn bản hiển thị trên trang..." }]
  }]
}

**Giải thích:**
- **number**: Số thứ tự của spread (bắt đầu từ 1)
- **left_page/right_page**: Thông tin trang trái/phải
  - `number`: Số trang
  - `type`: Loại trang (front_matter, story, back_matter)
- **manuscript**: Cốt truyện tương ứng cho spread này
  - Nếu spread có 2 trang: "trang 1: <nội dung>, trang 2: <nội dung>"
  - Nếu spread chỉ có 1 trang: không cần chia
  - Đây là reference cho Art Director khi tạo images[]
- **textboxes[]**: Nội dung text hiển thị trên trang
  - `order`: Thứ tự đọc của textbox
  - `language[]`: Mảng chứa text theo ngôn ngữ

---

## IMPORTANT RULES
1. All @mention_name must be unique, lowercase, using underscores
2. Text content must match target audience reading level
3. Include emotional progression across spreads
4. Clear beginning, middle, end with meaningful moral
5. KHÔNG tạo visual_description ở step này (sẽ được tạo ở generate-art-direction)
6. Output phải là valid JSON
```

### Flow
```
1. Validate input parameters
2. Lấy art_style description từ DB (bảng art_styles)
3. Lấy danh sách asset_categories từ DB (để cung cấp context cho LLM)
4. Lấy danh sách locations từ DB (để cung cấp context cho LLM)
5. Call LLM với story idea, attributes, và danh sách categories/locations có sẵn
6. Parse JSON response
7. Tạo Story record trong DB với step = 1 (manuscript), target_audience, artstyle_id
8. Tạo Snapshot record ban đầu
9. Lưu vào snapshot:
   - docs[] (4 documents: manuscript, story_structure, artistic_imagery, moral_lesson)
   - characters[] (basic info với category_id → FK asset_categories, personality, appearance - chưa có visual_description)
   - props[] (name, mention_name, category_id → FK asset_categories, type - chưa có visual_description)
   - stages[] (name, mention_name, location_id → FK locations - chưa có visual_description)
   - spreads[] (number, left_page, right_page, manuscript, textboxes - chưa có images[])
10. Update story metadata (title, summary, target_core_value)
11. Return { storyId, snapshotId, data }
```

---

## 3. generate-art-direction

### Description
**Art Director Agent** - Tạo hình (visual design) cho characters, props, stages và kịch bản hình ảnh chi tiết cho từng trang.

### Parameters
```typescript
interface GenerateArtDirectionParams {
  storyId: string;
  snapshotId: string;
}
```

### Result
```typescript
interface GenerateArtDirectionResult {
  success: boolean;
  updatedCharacters: {
    mentionName: string;
    visualDescription: string;
    appearanceNotes: string;      // Ghi chú cho illustrator
  }[];
  updatedProps: {
    mentionName: string;
    visualDescription: string;
  }[];
  updatedStages: {
    mentionName: string;
    visualDescription: string;
  }[];
  updatedSpreads: {
    number: number;
    images: ImageItem[];          // Kịch bản hình ảnh
  }[];
}

interface ImageItem {
  title: string;
  geometry: { x: number; y: number; w: number; h: number; rotation: number };
  visual_description: string;
  stage: string;                  // @mention
  actions: string;                // @character doing something
  temporal: {
    era: string;
    season: string;
    weather: string;
    time_of_day: string;
    duration: string;
  };
  sensory: {
    atmosphere: string;
    soundscape: string;
    lighting: string;
    color_palette: string;
  };
  emotional: {
    mood: string;
  };
  composition_notes: string;      // Ghi chú bố cục cho illustrator
}
```

### Prompt

#### System Prompt
```
You are an expert Art Director for children's picture books. Your role is to translate story elements into compelling visual designs.

Your expertise includes:
- Character design for picture books (appealing, recognizable, consistent)
- Color theory and palette selection
- Composition and visual storytelling
- Age-appropriate visual complexity
- Consistency across multiple illustrations
- Art style adaptation

You work with the Story Teller's analysis to create visual designs that:
- Capture personality through visual elements
- Maintain consistency across all illustrations
- Support the emotional journey of the story
- Are practical for illustration execution
```

#### User Prompt Template
```
Based on the story analysis, create visual designs for all elements:

## STORY CONTEXT
- Title: {title}
- Target Audience: {audience}
- Art Style: {artStyleDescription}
- Language: {language}

## EXISTING ANALYSIS
{Get all info from DB Snapshot: docs, characters, props, stages, spreads}

---

## OUTPUT REQUIREMENTS

### 1. CHARACTER VISUAL DESIGNS
For each character in characters[], create:

**Visual Description** (optimized for AI image generation):
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: colors, textures, proportions, distinctive features
- Typical expressions and poses that reflect personality
- Length: 80-150 words per character

Example format:
```
@miu_cat:
Visual Description: "A small orange tabby kitten with cream-colored stripes running along his back and an unusually fluffy tail that curls at the tip. His large, round eyes are bright emerald green with golden flecks near the pupils, always wide with curiosity and wonder. A tiny pink nose sits in the center of his round, expressive face. He wears a faded red polka-dot bow around his neck, slightly askew. His fur is soft and slightly messy, giving him a lovable, approachable appearance. His ears are large relative to his head, often perked forward attentively."
```

### 2. PROP VISUAL DESIGNS
For each prop in props[]:

**Visual Description** (50-80 words):
- Material, texture, color
- Size relative to characters
- Distinctive features
- Condition (new, worn, magical glow, etc.)

### 3. STAGE VISUAL DESIGNS
For each stage in stages[]:

**Visual Description** (100-150 words):
- Foreground, midground, background elements
- Color palette and lighting
- Atmosphere and mood
- Key landmarks or elements
- How characters interact with the space

### 4. SPREAD IMAGE SCRIPTS
For each spread, create detailed image specifications:

{
  "spread_number": 1,
  "images": [{
    "title": "Miu discovers the forest path",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
    "visual_description": "...",
    "stage": "@forest_1",
    "actions": "@miu_cat standing at forest edge, looking curiously at winding path",
    "temporal": {
      "era": "contemporary",
      "season": "autumn",
      "weather": "clear with soft clouds",
      "time_of_day": "late afternoon",
      "duration": "moment"
    },
    "sensory": {
      "atmosphere": "mysterious yet inviting",
      "soundscape": "rustling leaves, distant bird calls",
      "lighting": "warm golden hour light filtering through trees",
      "color_palette": "warm oranges, golden yellows, deep greens"
    },
    "emotional": {
      "mood": "curious, slightly anxious, hopeful"
    }
  }]
}

---

## VISUAL CONSISTENCY RULES
1. Characters must look identical across all spreads (same colors, proportions, features)
2. Stage elements that appear in multiple spreads must be consistent
3. Lighting should match the time_of_day specified
4. Color palette should support emotional tone of each scene
5. Art style must be consistent throughout

## OUTPUT FORMAT
Return JSON with:
- characters: array of { mentionName, visualDescription }
- props: array of { mentionName, visualDescription }
- stages: array of { mentionName, visualDescription }
- spreads: array of { number, images[] }
```

### Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy snapshot data từ DB (docs, characters, props, stages, spreads từ Step 1)
3. Lấy art_style description từ story.art_style_id
4. Call LLM với story context và existing analysis
5. Parse JSON response
6. Update snapshot:
   - characters[].visual_description
   - props[].visual_description
   - stages[].visual_description
   - spreads[].images[]
7. Return result
```

---

## 4. generate-quality-check

### Description
**Tester Agents** - Kiểm tra chất lượng nội dung qua nhiều góc nhìn: Story Consistency, Plot, và Age Appropriateness.

### Parameters
```typescript
interface GenerateQualityCheckParams {
  storyId: string;
  snapshotId: string;
}
```

### Result
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

// Map với bảng flags trong DB
interface FlagItem {
  title: string;                  // Tiêu đề vấn đề
  content: string;                // Mô tả chi tiết vấn đề
  suggestion: string;             // Gợi ý cách khắc phục
  type: number;                   // 0: consistency, 1: plot, 2: age_inappropriate, 3: other
  severity: number;               // 0: suggestion, 1: minor, 2: major, 3: critical
  target_type: number;            // 0: story, 1: character, 2: prop, 3: stage, 4: spread
  target_id: string;              // mention_name hoặc spread number
  tester: string;                 // "Story Consistency" | "Plot" | "Age Appropriateness"
}

// Enum mappings for reference
enum FlagType {
  CONSISTENCY = 0,
  PLOT = 1,
  AGE_INAPPROPRIATE = 2,
  OTHER = 3
}

enum FlagSeverity {
  SUGGESTION = 0,
  MINOR = 1,
  MAJOR = 2,
  CRITICAL = 3
}

enum FlagTargetType {
  STORY = 0,
  CHARACTER = 1,
  PROP = 2,
  STAGE = 3,
  SPREAD = 4
}

enum FlagStatus {
  OPEN = 0,
  IN_PROGRESS = 1,
  RESOLVED = 2,
  IGNORED = 3
}
```

### Sub-Agents

#### 4.1 Story Consistency Tester
Kiểm tra tính nhất quán của characters, props, stages với cốt truyện và visual design.

**Checks:**
- Character personality vs. actions in spreads
- Character appearance vs. visual descriptions
- Prop usage consistency
- Stage descriptions vs. spread usage
- Timeline/sequence logic
- Character relationships consistency

#### 4.2 Plot Tester
Kiểm tra cốt truyện có vấn đề logic hoặc narrative.

**Checks:**
- Plot holes (lỗ hổng cốt truyện)
- Contradictions (mâu thuẫn)
- Pacing issues (nhịp truyện)
- Character arc completion
- Moral/lesson clarity and depth
- Emotional journey coherence
- Ending satisfaction

#### 4.3 Age Appropriateness Tester (Parent + Child perspective)
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

### Prompt

#### System Prompt
```
You are a team of quality assurance testers for children's picture books. You evaluate content from multiple perspectives to ensure the highest quality.

You consist of three specialized testers:

1. **Story Consistency Tester**: Ensures all story elements are internally consistent
2. **Plot Tester**: Evaluates narrative structure, logic, and lesson effectiveness
3. **Age Appropriateness Tester**: Reviews content from both child and parent perspectives

Your evaluation is thorough but constructive. You identify issues and provide actionable suggestions for improvement.
```

#### User Prompt Template
```
Evaluate the following picture book manuscript:

## STORY METADATA
- Title: {title}
- Target Audience: {audience}
- Art Style: {artStyleDescription}
- Core Values: {targetCoreValue}

## DOCUMENTS
{characterBible}
{propBible}
{stageBible}
{literaryAnalysis}
{spreadOutline}

## CHARACTERS
{characters JSON with visual_description}

## PROPS
{props JSON with visual_description}

## STAGES
{stages JSON with visual_description}

## SPREADS
{spreads JSON with images[]}

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
      "title": "Character behavior inconsistency",
      "content": "In spread 3, Miu acts bravely despite being described as fearful in character bible",
      "suggestion": "Add internal dialogue showing Miu overcoming fear",
      "type": 0,
      "severity": 2,
      "target_type": 1,
      "target_id": "@miu_cat",
      "tester": "Story Consistency"
    }
  ],
  "overallScore": 87,
  "recommendation": "approved" | "needs_revision" | "major_issues"
}

## ENUM MAPPINGS
- **type**: 0=consistency, 1=plot, 2=age_inappropriate, 3=other
- **severity**: 0=suggestion, 1=minor, 2=major, 3=critical
- **target_type**: 0=story, 1=character, 2=prop, 3=stage, 4=spread

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

### Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy full snapshot data từ DB (docs, characters, props, stages, spreads)
3. Call LLM với all three tester prompts
4. Parse JSON response
5. Lưu flags[] vào DB (bảng flags)
6. Return test results
```

### Database: flags table
```sql
CREATE TABLE flags (
  id UUID PRIMARY KEY,
  user_id UUID,                   -- ID người tạo flag (tester hoặc system)
  story_id UUID REFERENCES story(id),
  snapshot_id UUID REFERENCES snapshot(id),
  title VARCHAR(255),
  content TEXT,                   -- Mô tả chi tiết vấn đề
  suggestion TEXT,                -- Gợi ý cách khắc phục
  type SMALLINT,                  -- 0: consistency, 1: plot, 2: age_inappropriate, 3: other
  severity SMALLINT,              -- 0: suggestion, 1: minor, 2: major, 3: critical
  target_type SMALLINT,           -- 0: story, 1: character, 2: prop, 3: stage, 4: spread
  target_id VARCHAR(100),         -- mention_name hoặc spread number
  tester VARCHAR(50),             -- "Story Consistency" | "Plot" | "Age Appropriateness"
  status SMALLINT DEFAULT 0,      -- 0: open, 1: in_progress, 2: resolved, 3: ignored
  created_at TIMESTAMP DEFAULT NOW(),
  resolved_at TIMESTAMP,
  resolved_by UUID
);
```

---

## 5. generate-visual-description-character

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (Art Direction) để tối ưu visual description.

### Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character.

### Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Character mention_name, dùng để lấy thông tin character từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả (short: 50-80, medium: 80-120, detailed: 120-200)
- language?: string              // Ngôn ngữ output - nếu không truyền, lấy từ story.original_language
```

### Result
```
- visualDescription: string      // Mô tả chính đã tối ưu cho image generation
- keywords: string[]             // Tags cho search/tagging (5-10 từ khóa)
- negativePrompt: string         // Những gì cần tránh (luôn trả về)
- suggestedReferences: string[]  // Gợi ý tìm ảnh reference (2-3 gợi ý)
```

### Prompt

#### System Prompt
```
You are a visual description writer for children's picture book illustrations.
Transform basic entity information into vivid descriptions optimized for AI image generation.

Rules:
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: colors, textures, proportions, lighting, mood
- Keep child-friendly (ages 2-8)
- Focus on distinctive visual features that make the character recognizable
- Consider consistency across multiple images
- Always provide a negative prompt to avoid unwanted elements
```

#### User Prompt Template
```
Generate a visual description for a character with the following information:

## Basic Information
{{#if basicInfo.name}}
**Name:** {{basicInfo.name}}
{{/if}}

{{#if basicInfo.description}}
**Description:** {{basicInfo.description}}
{{/if}}

## Character Details

{{#if basicInfo}}
### Basic Info
- Gender: {{basicInfo.gender}}
- Age: {{basicInfo.age}}
- Category: {{categoryInfo.name}} - {{categoryInfo.description}}
  (Lấy từ basicInfo.category_id → query bảng asset_categories)
- Role: {{basicInfo.role}}
{{/if}}

{{#if personality}}
### Personality (for expression/pose guidance)
- Core Essence: {{personality.core_essence}}
- Flaws: {{personality.flaws}}
- Emotions: {{personality.emotions}}
- Reactions: {{personality.reactions}}
- Desires: {{personality.desires}}
- Likes: {{personality.likes}}
- Fears: {{personality.fears}}
- Contradictions: {{personality.contradictions}}
{{/if}}

{{#if appearance}}
### Appearance
- Height: {{appearance.height}}
- Hair: {{appearance.hair}}
- Eyes: {{appearance.eyes}}
- Face: {{appearance.face}}
- Build: {{appearance.build}}
- Outfit: {{appearance.outfit}}
{{/if}}

{{#if inseparable_item}}
### Inseparable Item
{{inseparable_item}}
{{/if}}

## Art Style
**Style Reference:** {{artStyleDescription}}
(Lấy từ story.art_style_id → art_style.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_age}}
- Core Value: {{storyContext.target_core_value}}

{{#if storyContext.existingVisualDescriptions}}
### Existing Descriptions (for consistency)
{{#each storyContext.existingVisualDescriptions}}
- {{this.name}}: {{this.visualDescription}}
{{/each}}
{{/if}}

## Output Requirements
- **Length:** {{targetLength}}
- **Language:** {{language}}

---

Please generate:
1. A visual description optimized for AI image generation
2. 5-10 relevant keywords
3. A negative prompt listing what to avoid
4. 2-3 suggested reference search terms

Respond in JSON format:
{
  "visualDescription": "...",
  "keywords": ["...", "..."],
  "negativePrompt": "...",
  "suggestedReferences": ["...", "..."]
}
```

### Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy story info từ DB (art_style_id, original_language, target_age, genre, target_core_value)
3. Lấy character info từ snapshot.characters[] bằng mentionName
4. Lấy category info từ bảng asset_categories bằng character.basic_info.category_id
   → Lấy name, type, description của category
5. Lấy art_style description từ bảng art_styles
6. Lấy existing visual descriptions của các characters khác để đảm bảo consistency
7. Call LLM
8. Return result
```

---

## 6. generate-visual-description-prop

### Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop.

### Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Prop mention_name, dùng để lấy thông tin prop từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả
- language?: string              // Ngôn ngữ output - nếu không truyền, lấy từ story.original_language
```

### Result
```
- visualDescription: string      // Mô tả chính đã tối ưu cho image generation
- keywords: string[]             // Tags cho search/tagging (5-10 từ khóa)
- negativePrompt: string         // Những gì cần tránh (luôn trả về)
- suggestedReferences: string[]  // Gợi ý tìm ảnh reference (2-3 gợi ý)
```

### Prompt

#### System Prompt
```
You are a visual description writer for children's picture book illustrations.
Transform basic entity information into vivid descriptions optimized for AI image generation.

Rules:
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: colors, textures, proportions, materials, lighting
- Keep child-friendly (ages 2-8)
- Consider how the prop interacts with characters
- Describe at appropriate scale for picture book illustration
- Always provide a negative prompt to avoid unwanted elements
```

#### User Prompt Template
```
Generate a visual description for a prop with the following information:

## Basic Information
**Name:** {{name}}
**Mention Name:** @{{mention_name}}

{{#if visual_description}}
**Current Description:** {{visual_description}}
{{/if}}

## Prop Details
- Category: {{categoryInfo.name}} - {{categoryInfo.description}}
  (Lấy từ category_id → query bảng asset_categories)
- Type: {{type}}
  (narrative: đồ vật dẫn chuyện, tương tác với character)
  (anchor: đồ vật nằm trong stages, tạo ra sự nhất quán)

{{#if sounds}}
### Associated Sounds
{{#each sounds}}
- {{this.title}}: {{this.media_url}}
{{/each}}
{{/if}}

## Art Style
**Style Reference:** {{artStyleDescription}}
(Lấy từ story.art_style_id → art_style.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_age}}
- Core Value: {{storyContext.target_core_value}}

{{#if storyContext.existingVisualDescriptions}}
### Existing Descriptions (for consistency)
{{#each storyContext.existingVisualDescriptions}}
- {{this.name}}: {{this.visualDescription}}
{{/each}}
{{/if}}

## Output Requirements
- **Length:** {{targetLength}}
- **Language:** {{language}}

---

Please generate:
1. A visual description optimized for AI image generation
2. 5-10 relevant keywords
3. A negative prompt listing what to avoid
4. 2-3 suggested reference search terms

Respond in JSON format:
{
  "visualDescription": "...",
  "keywords": ["...", "..."],
  "negativePrompt": "...",
  "suggestedReferences": ["...", "..."]
}
```

### Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy story info từ DB (art_style_id, original_language, target_age, genre, target_core_value)
3. Lấy prop info từ snapshot.props[] bằng mentionName
4. Lấy category info từ bảng asset_categories bằng prop.category_id
   → Lấy name, type, description của category
5. Lấy art_style description từ bảng art_styles
6. Lấy existing visual descriptions của các props khác để đảm bảo consistency
7. Call LLM
8. Return result
```

---

## 7. generate-visual-description-stage

### Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background.

### Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Stage mention_name, dùng để lấy thông tin stage từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả
- language?: string              // Ngôn ngữ output - nếu không truyền, lấy từ story.original_language
```

### Result
```
- visualDescription: string      // Mô tả chính đã tối ưu cho image generation
- keywords: string[]             // Tags cho search/tagging (5-10 từ khóa)
- negativePrompt: string         // Những gì cần tránh (luôn trả về)
- suggestedReferences: string[]  // Gợi ý tìm ảnh reference (2-3 gợi ý)
```

### Prompt

#### System Prompt
```
You are a visual description writer for children's picture book illustrations.
Transform basic entity information into vivid descriptions optimized for AI image generation.

Rules:
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: colors, textures, depth, lighting, atmosphere, mood
- Keep child-friendly (ages 2-8)
- Describe foreground, midground, and background elements
- Consider how characters will interact with the environment
- Always provide a negative prompt to avoid unwanted elements
```

#### User Prompt Template
```
Generate a visual description for a stage/background with the following information:

## Basic Information
**Name:** {{name}}
**Mention Name:** @{{mention_name}}

{{#if visual_description}}
**Current Description:** {{visual_description}}
{{/if}}

## Stage/Setting Details
- Location: {{locationInfo.title}} - {{locationInfo.description}}
  (Lấy từ location_id → query bảng locations)
{{#if locationInfo.image_references}}
- Location Image References:
  {{#each locationInfo.image_references}}
  - {{this.title}}: {{this.media_url}}
  {{/each}}
{{/if}}

## Art Style
**Style Reference:** {{artStyleDescription}}
(Lấy từ story.art_style_id → art_styles.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_age}}
- Core Value: {{storyContext.target_core_value}}
- Era: {{eraInfo.title}} - {{eraInfo.description}}
  (Lấy từ story.era_id → query bảng eras)
- Story Location Setting: {{storyLocationInfo.title}} - {{storyLocationInfo.description}}
  (Lấy từ story.location_id → query bảng locations)

{{#if storyContext.existingVisualDescriptions}}
### Existing Descriptions (for consistency)
{{#each storyContext.existingVisualDescriptions}}
- {{this.name}}: {{this.visualDescription}}
{{/each}}
{{/if}}

## Output Requirements
- **Length:** {{targetLength}}
- **Language:** {{language}}

---

Please generate:
1. A visual description optimized for AI image generation
2. 5-10 relevant keywords
3. A negative prompt listing what to avoid
4. 2-3 suggested reference search terms

Respond in JSON format:
{
  "visualDescription": "...",
  "keywords": ["...", "..."],
  "negativePrompt": "...",
  "suggestedReferences": ["...", "..."]
}
```

### Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy story info từ DB (art_style_id, original_language, target_age, genre, target_core_value, era_id, location_id)
3. Lấy stage info từ snapshot.stages[] bằng mentionName
4. Lấy location info từ bảng locations bằng stage.location_id
   → Lấy title, description, image_references của location
5. Lấy era info từ bảng eras bằng story.era_id
   → Lấy title, description, image_references của era
6. Lấy story location info từ bảng locations bằng story.location_id
   → Lấy title, description của story location setting
7. Lấy art_style description từ bảng art_styles
8. Lấy existing visual descriptions của các stages khác để đảm bảo consistency
9. Call LLM
10. Return result
```

---

## 8. generate-visual-description-spread

### Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage).

### Parameters
```
- storyId: string                // ID của story trong DB
- spreadNumber: number           // Spread number, dùng để lấy thông tin spread từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả
- language?: string              // Ngôn ngữ output - nếu không truyền, lấy từ story.original_language
```

### Result
```
- visualDescription: string      // Mô tả chính đã tối ưu cho image generation
- keywords: string[]             // Tags cho search/tagging (5-10 từ khóa)
- negativePrompt: string         // Những gì cần tránh (luôn trả về)
- suggestedReferences: string[]  // Gợi ý tìm ảnh reference (2-3 gợi ý)
```

### Prompt

#### System Prompt
```
You are a visual description writer for children's picture book illustrations.
Transform scene information into vivid descriptions optimized for AI image generation.

Rules:
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: composition, character positions, actions, emotions, lighting, mood
- Keep child-friendly (ages 2-8)
- Describe the scene as a single cohesive illustration
- Maintain consistency with existing character and stage descriptions
- Focus on the key moment/action of the scene
- Always provide a negative prompt to avoid unwanted elements
```

#### User Prompt Template
```
Generate a visual description for a spread with the following information:

## Spread Info
**Spread Number:** {{number}}
**Pages:** {{left_page.number}}, {{right_page.number}}
**Page Types:** {{left_page.type}}, {{right_page.type}}

## Scene Composition

### Image Details
{{#each images}}
**Image: {{this.title}}**
- Current Visual Description: {{this.visual_description}}
- Geometry: x={{this.geometry.x}}, y={{this.geometry.y}}, w={{this.geometry.w}}, h={{this.geometry.h}}, rotation={{this.geometry.rotation}}

#### Scene Context
- Stage: @{{this.stage}}
- Actions: {{this.actions}}

#### Temporal Settings
- Era: {{this.temporal.era}}
- Season: {{this.temporal.season}}
- Weather: {{this.temporal.weather}}
- Time of Day: {{this.temporal.time_of_day}}
- Duration: {{this.temporal.duration}}

#### Sensory Details
- Atmosphere: {{this.sensory.atmosphere}}
- Soundscape: {{this.sensory.soundscape}}
- Lighting: {{this.sensory.lighting}}
- Color Palette: {{this.sensory.color_palette}}

#### Emotional
- Mood: {{this.emotional.mood}}

---
{{/each}}

### Text Content (for context)
{{#each textboxes}}
- {{this.language[0].text}}
{{/each}}

### Characters in Scene
(Lấy từ snapshot.characters[] dựa trên @mentions trong images[].actions)
{{#each charactersInScene}}
- **{{this.name}}** (@{{this.mention_name}}):
  - Visual Description: {{this.visual_description}}
{{/each}}

### Stage/Background
(Lấy từ snapshot.stages[] dựa trên @stage trong images[])
{{#if stage}}
- **{{stage.name}}** (@{{stage.mention_name}})
- Visual Description: {{stage.visual_description}}
- Location: {{stageLocationInfo.title}} - {{stageLocationInfo.description}}
  (Lấy từ stage.location_id → query bảng locations)
{{/if}}

### Props in Scene
(Lấy từ snapshot.props[] dựa trên @mentions)
{{#each propsInScene}}
- **{{this.name}}** (@{{this.mention_name}}): {{this.visual_description}}
{{/each}}

## Art Style
**Style Reference:** {{artStyleDescription}}
(Lấy từ story.art_style_id → art_style.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_age}}
- Core Value: {{storyContext.target_core_value}}
- Spread Position: {{spreadNumber}} of {{storyContext.totalSpreads}}

{{#if storyContext.previousSpreadDescription}}
### Previous Spread (for continuity)
{{storyContext.previousSpreadDescription}}
{{/if}}

## Output Requirements
- **Length:** {{targetLength}}
- **Language:** {{language}}

---

Please generate:
1. A visual description optimized for AI image generation that captures the entire scene as a single illustration
2. 5-10 relevant keywords
3. A negative prompt listing what to avoid
4. 2-3 suggested reference search terms

Respond in JSON format:
{
  "visualDescription": "...",
  "keywords": ["...", "..."],
  "negativePrompt": "...",
  "suggestedReferences": ["...", "..."]
}
```

### Flow
```
1. Validate input parameters (storyId, spreadNumber, targetLength, language)
2. Lấy story info từ DB (art_style_id, original_language, target_age, genre, target_core_value)
3. Lấy spread info từ snapshot.spreads[] bằng spreadNumber
4. Parse @mentions trong images[].stage và images[].actions
   → Lấy danh sách characters, props, stages liên quan
5. Lấy visual descriptions của các entities liên quan từ snapshot
6. Với mỗi stage liên quan, lấy location info từ bảng locations bằng stage.location_id
   → Lấy title, description của location
7. Lấy art_style description từ bảng art_styles
8. Lấy previous spread description nếu có (để đảm bảo continuity)
9. Call LLM
10. Return result
```

---

## 9. translate-content

### Description
Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả.

### Parameters
```
- storyId: string                // ID của story trong DB
- content: string | Array<{ id: string; text: string }>  // Nội dung cần dịch
- sourceLanguage?: string        // Ngôn ngữ nguồn - nếu không truyền, lấy từ story.original_language
- targetLanguage: string         // Ngôn ngữ đích (bắt buộc)
- contentType: "manuscript" | "textbox" | "visual_description" | "metadata"
- context?: {
    characterNames?: Record<string, string>;  // Name mappings (optional)
  }
- preserveFormatting?: boolean   // Giữ nguyên markdown/formatting (default: true)
```

### Result
```
- translations: Array<{
    id?: string;              // ID nếu input là array
    originalText: string;
    translatedText: string;
  }>
- languagePair: string          // e.g., "vi→en"
```

### Prompt

#### System Prompt
```
You are a professional translator specializing in children's literature.
Translate content while:
- Maintaining the reading level appropriate for the target age group
- Preserving emotional tone and storytelling rhythm
- Adapting cultural references when necessary
- Keeping character names as specified in the name mappings (or unchanged if not specified)
- Preserving all markdown formatting and structure

Rules:
- Do NOT translate @mention_name references
- Preserve line breaks and paragraph structure
- Keep onomatopoeia culturally appropriate for target language
- Maintain the same sentence length ratio as source
```

#### User Prompt Template
```
Translate the following content from {{sourceLanguage}} to {{targetLanguage}}:

## Context
- Story Title: {{storyContext.title}}
- Target Age: {{storyContext.target_age}}
- Genre: {{storyContext.genre}}

{{#if context.characterNames}}
## Character Name Mappings
{{#each context.characterNames}}
- {{@key}} → {{this}}
{{/each}}
{{/if}}

## Content Type
{{contentType}}

## Content to Translate
{{#if contentIsArray}}
{{#each content}}
### Item {{this.id}}
{{this.text}}

---
{{/each}}
{{else}}
{{content}}
{{/if}}

## Requirements
- Preserve formatting: {{preserveFormatting}}
- Do NOT translate @mention_name (e.g., @miu_cat, @forest_stage)
- Keep the same emotional tone
- Use vocabulary appropriate for {{storyContext.target_age}}

---

Respond in JSON format:
{
  "translations": [
    {
      "id": "..." (if applicable),
      "originalText": "...",
      "translatedText": "..."
    }
  ]
}
```

### Flow
```
1. Validate input parameters (storyId, content, targetLanguage, contentType)
2. Lấy story info từ DB (original_language, target_age, genre, title)
3. Nếu sourceLanguage không truyền, dùng story.original_language
4. Call LLM
5. Return translations
```

---

## Tóm tắt Edge Functions - Text Generation

### Manuscript Generation (Multi-Step Pipeline)

| # | Function | Agent | Mục đích | Priority |
|---|----------|-------|----------|----------|
| 1 | generate-manuscript | Orchestrator | Điều phối tạo manuscript qua 3 bước | P0 |
| 2 | generate-story-draft | Story Teller | Tạo Story/Snapshot, phân tích văn học, tạo characters/props/stages/spreads | P0 |
| 3 | generate-art-direction | Art Director | Tạo hình, kịch bản hình ảnh | P0 |
| 4 | generate-quality-check | Testers | Kiểm tra chất lượng, phát hiện issues | P0 |

**Pipeline Flow:**
```
generate-story-draft → generate-art-direction → generate-quality-check
         ↓                        ↓                        ↓
  Story + Snapshot         visual_description          flags[]
  docs[] (4 docs)          spreads[].images[]
  characters[]
  props[]
  stages[]
  spreads[]
```

**docs[] structure (4 documents):**
- `manuscript`: Cốt truyện hoàn chỉnh
- `story_structure`: Cấu trúc truyện
- `artistic_imagery`: Hình tượng nghệ thuật
- `moral_lesson`: Bài học đạo đức

### Visual Description Generation

| # | Function | Mục đích | Priority |
|---|----------|----------|----------|
| 5 | generate-visual-description-character | Tối ưu visual description cho character | P0 |
| 6 | generate-visual-description-prop | Tối ưu visual description cho prop | P0 |
| 7 | generate-visual-description-stage | Tối ưu visual description cho stage | P0 |
| 8 | generate-visual-description-spread | Tối ưu visual description cho spread | P0 |

### Other

| # | Function | Mục đích | Priority |
|---|----------|----------|----------|
| 9 | translate-content | Dịch nội dung sang ngôn ngữ khác | P1 |

**Priority Legend:**
- P0: MVP - Cần có cho launch
- P1: Important - Cần có trong 1-2 tháng sau launch

**AI Provider:** Claude / GPT-4 (cho tất cả Text Generation functions)

**Database Tables Affected:**
- `story`: Metadata, title, summary
- `snapshot`: docs[], characters[], props[], stages[], spreads[]
- `flags`: Quality check issues (new table)
- `art_style`: Reference for visual consistency

---

> **Note:** Chi tiết về DB Schema và quy tắc implement xem tại [CLAUDE.md](./CLAUDE.md)
