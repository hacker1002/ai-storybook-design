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

> **DB Template Names:**
> - System: `VISUAL_DESC_CHARACTER_SYSTEM`
> - User: `VISUAL_DESC_CHARACTER_USER_TEMPLATE`

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
**Name:** {%name%}
**Mention Name:** @{%key%}
**Description:** {%description%}

## Character Details

### Basic Info
- Gender: {%gender%}
- Age: {%age%}
- Category: {%category_name%} - {%category_description%}
  { // Lấy từ basicInfo.category_id → query bảng asset_categories }
- Role: {%role%}

### Personality (for expression/pose guidance)
{ // from character.personality:
- core_essence, flaws, emotions, reactions, desires, likes, fears, contradictions
}
{%personality_text%}

### Appearance
{ // from character.appearance:
- height, hair, eyes, face, build
}
{%appearance_text%}

## Art Style
**Style Reference:** {%art_style_description%}
{ // Lấy từ story.artstyle_id → art_styles.description }

## Story Context
- Title: {%title%}
- Genre: {%genre%}
- Target Age: {%target_audience%}
- Core Value: {%target_core_value%}

### Existing Descriptions (for consistency)
{ // visual_description của các characters khác trong story:
- @other_character_key: "..."
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
   - Query `prompt_templates` với name = "VISUAL_DESC_CHARACTER_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_CHARACTER_USER_TEMPLATE" → user prompt template
3. Lấy story info từ DB (artstyle_id, original_language, target_audience, genre, target_core_value)
4. Lấy character info từ snapshot.characters[] bằng mentionName
5. Lấy category info từ bảng asset_categories bằng character.basic_info.category_id
   → Lấy name, type, description của category
6. Lấy art_style description từ bảng art_styles
7. Lấy existing visual descriptions của các characters khác để đảm bảo consistency
8. Render user prompt template với variables:
   - name, key, description, gender, age, category_name, category_description, role
   - personality_text, appearance_text, art_style_description
   - title, genre, target_audience, target_core_value
   - existing_visual_descriptions, target_length, language
9. Call LLM với system prompt và rendered user prompt
10. Return result
```

