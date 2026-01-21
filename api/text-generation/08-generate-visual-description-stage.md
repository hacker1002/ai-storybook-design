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
**Name:** {{name}}
**Mention Name:** @{{key}}

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
(Lấy từ story.artstyle_id → art_styles.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_audience}}
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

## Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value, era_id, location_id)
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
