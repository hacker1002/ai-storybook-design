# generate-spread-visual-plan

## Description
**Art Director Agent - Phase 1** - Tạo visual design cho characters, props, stages và kịch bản hình ảnh cơ bản cho từng spread.

**Important:** Step này tạo `visual_description` **KHÔNG cần art_style**. Art style được apply sau khi user chọn ở Phase 2 (Settings) và được sử dụng trong image generation.

## DB Schema Dependencies

### Tables Referenced
- `stories`: Truy vấn target_audience, original_language, format_genre, content_genre
- `snapshots`: Đọc docs[], characters[], props[], stages[], spreads[] từ Step 1; UPDATE visual_description và images[]

### Fields Used/Updated
- `snapshots.characters[].variants[].visual_description`, `negative_prompt` - UPDATE
- `snapshots.props[].states[].visual_description`, `negative_prompt` - UPDATE
- `snapshots.stages[].settings[].visual_description`, `negative_prompt`, `temporal`, `sensory`, `emotional` - UPDATE
- `snapshots.spreads[].images[]` - UPDATE (title, setting, visual_description, negative_prompt - chưa có geometry)
- `stories.title`, `stories.target_audience`, `stories.original_language`, `stories.format_genre`, `stories.content_genre` - READ

**Note:** Không đọc `stories.artstyle_id` - chưa được set ở thời điểm này.

## Parameters
```typescript
interface GenerateSpreadVisualPlanInput {
  storyId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateSpreadVisualPlanResult {
  success: boolean;
  updatedCharacters: {
    key: string;                        // character key
    variantKey: string;                 // variant key (usually "default")
    visualDescription: string;
    negativePrompt: string;
  }[];
  updatedProps: {
    key: string;                        // prop key
    stateKey: string;                   // state key (usually "default")
    visualDescription: string;
    negativePrompt: string;
  }[];
  updatedStages: {
    key: string;                        // stage key
    settingKey: string;                 // setting key (usually "default")
    visualDescription: string;
    negativePrompt: string;
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
  }[];
  updatedSpreads: {
    number: number;
    images: ImageItemBasic[];
  }[];
}

interface ImageItemBasic {
  title: string;
  setting: string;                      // Format: @stage_key/setting_key
  visual_description: string;           // Include character actions in description
  negative_prompt: string;              // What to avoid in image generation
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

In this phase, you focus on:
- Creating visual identities for characters, props, and stages
- Planning image content for each spread (what to show, not how to layout)
- Ensuring visual consistency across the book

You do NOT handle:
- Text placement or composition (handled in Phase 2)
- Art style specifics (applied during image generation)
```

### User Prompt Template
```
Based on the story analysis, create visual designs for all elements:

## STORY CONTEXT
- Title: {%title%}
- Target Audience: {%target_audience%}
- Format Genre: {%format_genre%}
- Content Genre: {%content_genre%}
- Language: {%language%}

## DOCUMENTS
{%docs_text%}

## CHARACTERS
{%characters_text%}

## PROPS
{%props_text%}

## STAGES
{%stages_text%}

## SPREADS
{%spreads_text%}

---

## OUTPUT REQUIREMENTS

### 1. CHARACTER VISUAL DESIGNS
For each character variant, create:

**Visual Description** (optimized for AI image generation):
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice"
- Include: colors, textures, proportions, distinctive features
- Typical expressions and poses that reflect personality
- Length: 80-150 words per character
- **Art-style neutral** - describe the character itself, not the rendering style

**Negative Prompt**: What to avoid when generating this character (wrong colors, wrong features, unwanted elements)

Example:
```json
{
  "key": "miu_cat",
  "variantKey": "default",
  "visualDescription": "A small orange tabby kitten with cream-colored stripes running along his back and an unusually fluffy tail that curls at the tip. His large, round eyes are bright emerald green with golden flecks near the pupils, always wide with curiosity. A tiny pink nose sits in the center of his round, expressive face. His ears are large relative to his head, often perked forward attentively.",
  "negativePrompt": "gray fur, blue eyes, short tail, adult cat, realistic style"
}
```

### 2. PROP VISUAL DESIGNS
For each prop state (50-80 words):
- Material, texture, color
- Size relative to characters
- Distinctive features
- Condition (new, worn, magical glow, etc.)
- Include `negativePrompt`

### 3. STAGE VISUAL DESIGNS
For each stage setting (100-150 words):
- Foreground, midground, background elements
- Color palette and lighting
- Atmosphere and mood
- Key landmarks or elements
- How characters interact with the space
- Include `temporal`, `sensory`, `emotional` objects
- Include `negativePrompt`

### 4. SPREAD IMAGE PLANS
For each spread:

```json
{
  "spread_number": 1,
  "images": [{
    "title": "Miu discovers the forest path",
    "setting": "@forest_1/default",
    "visual_description": "@miu_cat standing at forest edge, looking curiously at winding path ahead. Golden afternoon light filters through autumn leaves, casting warm shadows. The path winds mysteriously into the dense forest.",
    "negative_prompt": "night scene, winter, rain, scary atmosphere, realistic style"
  }]
}
```

**Notes:**
- `setting` format: `@stage_key/setting_key` - references stage setting for temporal/sensory/emotional context
- Include character @mentions and actions directly in `visual_description`
- Do NOT include geometry (x, y, w, h) - handled in composition phase
- Do NOT mention text placement in visual_description
- Focus on WHAT to show, not WHERE to place it
- Keep descriptions art-style neutral
- Always include `negative_prompt` for each image

---

## VISUAL CONSISTENCY RULES
1. Characters must look identical across all spreads (same colors, proportions, features)
2. Stage elements that appear in multiple spreads must be consistent
3. Lighting should match the time_of_day specified
4. Color palette should support emotional tone of each scene

## OUTPUT FORMAT
Return JSON with:
- characters: array of { key, variantKey, visualDescription, negativePrompt }
- props: array of { key, stateKey, visualDescription, negativePrompt }
- stages: array of { key, settingKey, visualDescription, negativePrompt, temporal, sensory, emotional }
- spreads: array of { number, images[] } where images = { title, setting, visual_description, negative_prompt }
```

## Flow
```
1. Validate input parameters (storyId, snapshotId)

2. Lấy prompt templates từ DB:
   - ART_DIRECTOR_P1_SYSTEM → system prompt + model
   - VISUAL_PLAN_USER_TEMPLATE → user prompt template

3. Lấy snapshot data từ DB (docs, characters, props, stages, spreads từ Step 1)

4. Lấy story metadata (title, target_audience, format_genre, content_genre, original_language)
   Note: KHÔNG lấy artstyle_id (chưa được set)

5. Render user prompt template với variables:
   - title, target_audience, format_genre, content_genre, language
   - docs_text, characters_text, props_text, stages_text, spreads_text

6. Call LLM

7. Parse JSON response

8. Update snapshot:
   - characters[].variants[variantKey].visual_description, negative_prompt
   - props[].states[stateKey].visual_description, negative_prompt
   - stages[].settings[settingKey].visual_description, negative_prompt, temporal, sensory, emotional
   - spreads[].images[] = { title, setting, visual_description, negative_prompt } (không có geometry)

9. Return result
```

## Error Handling
- Snapshot không tồn tại → Return error
- LLM response không parse được → Retry 1 lần
- Vẫn fail → Return partial result với những gì đã parse được

## Notes
- **Art style independence:** Visual descriptions được viết art-style neutral. Art style sẽ được inject vào prompt khi generate image.
