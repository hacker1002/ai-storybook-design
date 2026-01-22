# generate-spread-visual-plan

## Description
**Art Director Agent - Phase 1** - Tạo visual design cho characters, props, stages và kịch bản hình ảnh cơ bản cho từng spread. Chưa xử lý composition/bố cục cho text.

## DB Schema Dependencies

### Tables Referenced
- `story`: Truy vấn artstyle_id, target_audience, original_language
- `snapshot`: Đọc docs[], characters[], props[], stages[], spreads[] từ Step 1, sau đó UPDATE visual_description và images[]
- `art_styles`: Truy vấn description để đưa vào prompt

### Fields Used/Updated
- `snapshot.characters[].visual_description` - UPDATE
- `snapshot.props[].visual_description` - UPDATE
- `snapshot.stages[].visual_description` - UPDATE
- `snapshot.spreads[].images[]` - UPDATE (thêm mới, chưa có geometry cuối cùng)
- `story.artstyle_id` → `art_styles.description` - READ

## Parameters
```typescript
interface GenerateSpreadVisualPlanParams {
  storyId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateSpreadVisualPlanResult {
  success: boolean;
  updatedCharacters: {
    key: string;              // @key
    visualDescription: string;
  }[];
  updatedProps: {
    key: string;
    visualDescription: string;
  }[];
  updatedStages: {
    key: string;
    visualDescription: string;
  }[];
  updatedSpreads: {
    number: number;
    images: ImageItemBasic[];
  }[];
}

interface ImageItemBasic {
  title: string;
  visual_description: string;    // Mô tả cảnh, CHƯA có ghi chú về text placement
  stage: string;                 // @mention
  actions: string;               // @character doing something
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
}
```

## Prompt

> **DB Template Names:**
> - System: `ART_DIRECTOR_P1_SYSTEM`
> - User: `VISUAL_PLAN_USER_TEMPLATE`

### System Prompt
```
You are an expert Art Director for children's picture books. Your role is to translate story elements into compelling visual designs.

Your expertise includes:
- Character design for picture books (appealing, recognizable, consistent)
- Color theory and palette selection
- Visual storytelling through imagery
- Age-appropriate visual complexity
- Consistency across multiple illustrations
- Art style adaptation

In this phase, you focus on:
- Creating visual identities for characters, props, and stages
- Planning image content for each spread (what to show, not how to layout)
- Ensuring visual consistency across the book

You do NOT handle text placement or composition in this phase.
```

### User Prompt Template
```
Based on the story analysis, create visual designs for all elements:

## STORY CONTEXT
- Title: {%title%}
- Target Audience: {%target_audience%}
- Art Style: {%art_style_description%}
- Language: {%language%}

## DOCUMENTS
{ // from snapshot.docs[]:
- type: "manuscript", title: "...", content: "..."
- type: "story_structure", title: "...", content: "..."
- ...
}
{%docs_text%}

## CHARACTERS
{ // from snapshot.characters[]:
- key: "@miu_cat", name: "Miu", basic_info: {...}, personality: {...}, appearance: {...}
- ...
}
{%characters_text%}

## PROPS
{ // from snapshot.props[]:
- key: "@red_bow", name: "Chiếc nơ đỏ", category_id: "...", type: "narrative"
- ...
}
{%props_text%}

## STAGES
{ // from snapshot.stages[]:
- key: "@forest_1", name: "Khu rừng", location_id: "..."
- ...
}
{%stages_text%}

## SPREADS
{ // from snapshot.spreads[]:
- number: 1, manuscript: "...", textboxes: [...]
- ...
}
{%spreads_text%}

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

### 4. SPREAD IMAGE PLANS
For each spread, define what images to show:

{
  "spread_number": 1,
  "images": [{
    "title": "Miu discovers the forest path",
    "visual_description": "Miu standing at the edge of a forest, looking at a winding path that disappears between tall trees. Fallen autumn leaves scattered on the ground. Warm sunlight filtering through the canopy.",
    "stage": "@forest_1",
    "actions": "@miu_cat standing at forest edge, looking curiously at winding path, tail raised with interest",
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

**Notes:**
- Do NOT include geometry (x, y, w, h) - this will be handled in composition phase
- Do NOT mention text placement in visual_description
- Focus on WHAT to show, not WHERE to place it

---

## VISUAL CONSISTENCY RULES
1. Characters must look identical across all spreads (same colors, proportions, features)
2. Stage elements that appear in multiple spreads must be consistent
3. Lighting should match the time_of_day specified
4. Color palette should support emotional tone of each scene
5. Art style must be consistent throughout

## OUTPUT FORMAT
Return JSON with:
- characters: array of { key, visualDescription }
- props: array of { key, visualDescription }
- stages: array of { key, visualDescription }
- spreads: array of { number, images[] }
```

## Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "ART_DIRECTOR_P1_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_PLAN_USER_TEMPLATE" → user prompt template
3. Lấy snapshot data từ DB (docs, characters, props, stages, spreads từ Step 1)
4. Lấy artstyle description từ story.artstyle_id
5. Render user prompt template với variables:
   - title, target_audience, art_style_description, language
   - docs_text, characters_text, props_text, stages_text, spreads_text
6. Call LLM với system prompt và rendered user prompt
7. Parse JSON response
8. Update snapshot:
   - characters[].visual_description
   - props[].visual_description
   - stages[].visual_description
   - spreads[].images[] (basic, không có geometry)
9. Return result
```

## Error Handling
- Nếu snapshot không tồn tại → Return error
- Nếu LLM response không parse được → Retry 1 lần
- Nếu vẫn fail → Return partial result với những gì đã parse được
