# 8. generate-visual-description-spread

## Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage).

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value
- `snapshots`: spreads[], characters[], props[], stages[]
- `locations`: id, name, description, image_references[]
- `art_styles`: id, name, description, image_references[]

### JSONB Fields
- `spreads[]` structure (from snapshots):
  - number
  - left_page: { number, type }
  - right_page: { number, type }
  - manuscript
  - tiny_sketch_media_url
  - images[]: { title, geometry, visual_description, stage, actions, temporal, sensory, emotional, image_references[], sketch[], illustration[], final_hires_media_url, interaction }
  - videos[], textboxes[], shapes[]

- `characters[]`, `props[]`, `stages[]` structures (referenced via @mentions)

## Parameters
```
- storyId: string                // ID của story trong DB
- spreadNumber: number           // Spread number, dùng để lấy thông tin spread từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả
- language?: string              // Ngôn ngữ output - nếu không truyền, lấy từ story.original_language
```

## Result
```
- visualDescription: string      // Mô tả chính đã tối ưu cho image generation
- keywords: string[]             // Tags cho search/tagging (5-10 từ khóa)
- negativePrompt: string         // Những gì cần tránh (luôn trả về)
- suggestedReferences: string[]  // Gợi ý tìm ảnh reference (2-3 gợi ý)
```

## Prompt

> **DB Template Names:**
> - System: `VISUAL_DESC_SPREAD_SYSTEM`
> - User: `VISUAL_DESC_SPREAD_USER_TEMPLATE`

### System Prompt
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

### User Prompt Template
```
Generate a visual description for a spread with the following information:

## Spread Info
**Spread Number:** {%spread_number%}
**Left Page:** {%left_page_number%} (type: {%left_page_type%})
**Right Page:** {%right_page_number%} (type: {%right_page_type%})

## Scene Composition

### Image Details
{ // from spread.images[]:
- title, visual_description, geometry, stage, actions
- temporal: { era, season, weather, time_of_day, duration }
- sensory: { atmosphere, soundscape, lighting, color_palette }
- emotional: { mood }
}
{%images_text%}

### Text Content (for context)
{ // from spread.textboxes[].language[].text:
- "Text content 1..."
- "Text content 2..."
}
{%textboxes_text%}

### Characters in Scene
{ // Lấy từ snapshot.characters[] dựa trên @mentions trong images[].actions:
- @character_key: name, visual_description
}
{%characters_in_scene%}

### Stage/Background
{ // Lấy từ snapshot.stages[] dựa trên @stage trong images[]:
- @stage_key: name, visual_description, location info
}
{%stage_info%}

### Props in Scene
{ // Lấy từ snapshot.props[] dựa trên @mentions:
- @prop_key: name, visual_description
}
{%props_in_scene%}

## Art Style
**Style Reference:** {%art_style_description%}
{ // Lấy từ story.artstyle_id → art_styles.description }

## Story Context
- Title: {%title%}
- Genre: {%genre%}
- Target Age: {%target_audience%}
- Core Value: {%target_core_value%}
- Spread Position: {%spread_number%} of {%total_spreads%}

### Previous Spread (for continuity)
{ // visual_description của spread trước (nếu có):
}
{%previous_spread_description%}

## Output Requirements
- **Length:** {%target_length%}
- **Language:** {%language%}

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

## Flow
```
1. Validate input parameters (storyId, spreadNumber, targetLength, language)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "VISUAL_DESC_SPREAD_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_SPREAD_USER_TEMPLATE" → user prompt template
3. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value)
4. Lấy spread info từ snapshot.spreads[] bằng spreadNumber
5. Parse @mentions trong images[].stage và images[].actions
   → Lấy danh sách characters, props, stages liên quan
6. Lấy visual descriptions của các entities liên quan từ snapshot
7. Với mỗi stage liên quan, lấy location info từ bảng locations bằng stage.location_id
   → Lấy name, description của location
8. Lấy art_style description từ bảng art_styles
9. Lấy previous spread description nếu có (để đảm bảo continuity)
10. Render user prompt template với variables:
    - spread_number, left_page_number, right_page_number, left_page_type, right_page_type
    - images_text, textboxes_text, characters_in_scene, stage_info, props_in_scene
    - art_style_description, title, genre, target_audience, target_core_value
    - total_spreads, previous_spread_description, target_length, language
11. Call LLM với system prompt và rendered user prompt
12. Return result
```
