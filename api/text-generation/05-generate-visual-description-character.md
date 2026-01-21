# 5. generate-visual-description-character

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (Art Direction) để tối ưu visual description.

## Description
Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character.

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value
- `snapshots`: characters[]
- `asset_categories`: id, name, type, description
- `art_styles`: id, name, description, image_references[]

### JSONB Fields
- `characters[]` structure (from snapshots):
  - order, key, name
  - basic_info: { description, gender, age, category_id, role }
  - personality: { core_essence, flaws, emotions, reactions, desires, likes, fears, contradictions }
  - appearance: { height, hair, eyes, face, build }
  - visual_description
  - sketch[], illustration[], image_references[]
  - voice: { stability, clarity, similarity, style_exaggeration, speaker_boost, system_voice, media_url }

## Parameters
```
- storyId: string                // ID của story trong DB
- mentionName: string            // Character key, dùng để lấy thông tin character từ DB
- targetLength: "short" | "medium" | "detailed"  // Số lượng từ cho mô tả (short: 50-80, medium: 80-120, detailed: 120-200)
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
- Include: colors, textures, proportions, lighting, mood
- Keep child-friendly (ages 2-8)
- Focus on distinctive visual features that make the character recognizable
- Consider consistency across multiple images
- Always provide a negative prompt to avoid unwanted elements
```

### User Prompt Template
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
3. Lấy character info từ snapshot.characters[] bằng mentionName
4. Lấy category info từ bảng asset_categories bằng character.basic_info.category_id
   → Lấy name, type, description của category
5. Lấy art_style description từ bảng art_styles
6. Lấy existing visual descriptions của các characters khác để đảm bảo consistency
7. Call LLM
8. Return result
```

