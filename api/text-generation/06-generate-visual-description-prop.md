# 6. generate-visual-description-prop

## Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop.

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value
- `snapshots`: props[]
- `asset_categories`: id, name, type, description
- `art_styles`: id, name, description, image_references[]

### JSONB Fields
- `props[]` structure (from snapshots):
  - order, name, key
  - category_id
  - visual_description
  - type: "narrative" | "anchor"
  - sketch[], illustration[], image_references[]
  - sounds[]: [{ title, media_url }]

## Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Prop key, dùng để lấy thông tin prop từ DB
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
- Include: colors, textures, proportions, materials, lighting
- Keep child-friendly (ages 2-8)
- Consider how the prop interacts with characters
- Describe at appropriate scale for picture book illustration
- Always provide a negative prompt to avoid unwanted elements
```

### User Prompt Template
```
Generate a visual description for a prop with the following information:

## Basic Information
**Name:** {{name}}
**Mention Name:** @{{key}}

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
(Lấy từ story.artstyle_id → art_styles.description)

## Story Context
- Title: {{storyContext.title}}
- Genre: {{storyContext.genre}}
- Target Age: {{storyContext.target_audience}}
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

## Flow
```
1. Validate input parameters (storyId, mentionName, targetLength, language)
2. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value)
3. Lấy prop info từ snapshot.props[] bằng mentionName
4. Lấy category info từ bảng asset_categories bằng prop.category_id
   → Lấy name, type, description của category
5. Lấy art_style description từ bảng art_styles
6. Lấy existing visual descriptions của các props khác để đảm bảo consistency
7. Call LLM
8. Return result
```

