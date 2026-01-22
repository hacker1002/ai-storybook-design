# 7. generate-visual-description-stage

## Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background.

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value, era_id, location_id
- `snapshots`: stages[]
- `locations`: id, name, description, nation, city, image_references[]
- `eras`: id, name, description, image_references[]
- `art_styles`: id, name, description, image_references[]

### JSONB Fields
- `stages[]` structure (from snapshots):
  - order, name, key
  - visual_description
  - location_id (FK → locations)
  - sketch[], illustration[], image_references[]
  - sounds[]: [{ title, media_url }]

## Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Stage key, dùng để lấy thông tin stage từ DB
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
> - System: `VISUAL_DESC_STAGE_SYSTEM`
> - User: `VISUAL_DESC_STAGE_USER_TEMPLATE`

### System Prompt
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

### User Prompt Template
```
Generate a visual description for a stage/background with the following information:

## Basic Information
**Name:** {%name%}
**Mention Name:** @{%key%}
**Current Description:** {%current_description%}

## Stage/Setting Details
- Location: {%location_name%} - {%location_description%}
  { // Lấy từ location_id → query bảng locations }
- Location Image References:
  { // from location.image_references[]:
  - title: "...", media_url: "..."
  }
  {%location_image_references%}

## Art Style
**Style Reference:** {%art_style_description%}
{ // Lấy từ story.artstyle_id → art_styles.description }

## Story Context
- Title: {%title%}
- Genre: {%genre%}
- Target Age: {%target_audience%}
- Core Value: {%target_core_value%}
- Era: {%era_name%} - {%era_description%}
  { // Lấy từ story.era_id → query bảng eras }
- Story Location Setting: {%story_location_name%} - {%story_location_description%}
  { // Lấy từ story.location_id → query bảng locations }

### Existing Descriptions (for consistency)
{ // visual_description của các stages khác trong story:
- @other_stage_key: "..."
- ...
}
{%existing_visual_descriptions%}

## Output Requirements
- **Length:** {%target_length%}
- **Language:** {%language%}

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

## Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "VISUAL_DESC_STAGE_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_STAGE_USER_TEMPLATE" → user prompt template
3. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value, era_id, location_id)
4. Lấy stage info từ snapshot.stages[] bằng mentionName
5. Lấy location info từ bảng locations bằng stage.location_id
   → Lấy name, description, image_references của location
6. Lấy era info từ bảng eras bằng story.era_id
   → Lấy name, description, image_references của era
7. Lấy story location info từ bảng locations bằng story.location_id
   → Lấy name, description của story location setting
8. Lấy art_style description từ bảng art_styles
9. Lấy existing visual descriptions của các stages khác để đảm bảo consistency
10. Render user prompt template với variables:
    - name, key, current_description, location_name, location_description, location_image_references
    - art_style_description, title, genre, target_audience, target_core_value
    - era_name, era_description, story_location_name, story_location_description
    - existing_visual_descriptions, target_length, language
11. Call LLM với system prompt và rendered user prompt
12. Return result
```
