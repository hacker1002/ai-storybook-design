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
- **{{this.name}}** (@{{this.key}}):
  - Visual Description: {{this.visual_description}}
{{/each}}

### Stage/Background
(Lấy từ snapshot.stages[] dựa trên @stage trong images[])
{{#if stage}}
- **{{stage.name}}** (@{{stage.key}})
- Visual Description: {{stage.visual_description}}
- Location: {{stageLocationInfo.title}} - {{stageLocationInfo.description}}
  (Lấy từ stage.location_id → query bảng locations)
{{/if}}

### Props in Scene
(Lấy từ snapshot.props[] dựa trên @mentions)
{{#each propsInScene}}
- **{{this.name}}** (@{{this.key}}): {{this.visual_description}}
{{/each}}

## Art Style
**Style Reference:** {{artStyleDescription}}
(Lấy từ story.artstyle_id → art_styles.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_audience}}
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

## Flow
```
1. Validate input parameters (storyId, spreadNumber, targetLength, language)
2. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value)
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
