# 3. generate-art-direction

## Description
**Art Director Agent** - Tạo hình (visual design) cho characters, props, stages và kịch bản hình ảnh chi tiết cho từng trang.

## DB Schema Dependencies

### Tables Referenced
- `story`: Truy vấn artstyle_id, target_audience, original_language
- `snapshot`: Đọc docs[], characters[], props[], stages[], spreads[] từ Step 1, sau đó UPDATE visual_description và images[]
- `art_styles`: Truy vấn description để đưa vào prompt

### Fields Used/Updated
- `snapshot.characters[].visual_description` - UPDATE
- `snapshot.props[].visual_description` - UPDATE
- `snapshot.stages[].visual_description` - UPDATE
- `snapshot.spreads[].images[]` - UPDATE (thêm mới)
- `story.artstyle_id` → `art_styles.description` - READ

## Parameters
```typescript
interface GenerateArtDirectionParams {
  storyId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateArtDirectionResult {
  success: boolean;
  updatedCharacters: {
    mentionName: string;
    visualDescription: string;
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

## Prompt

### System Prompt
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

### User Prompt Template
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

## Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy snapshot data từ DB (docs, characters, props, stages, spreads từ Step 1)
3. Lấy artstyle description từ story.artstyle_id
4. Call LLM với story context và existing analysis
5. Parse JSON response
6. Update snapshot:
   - characters[].visual_description
   - props[].visual_description
   - stages[].visual_description
   - spreads[].images[]
7. Return result
```
