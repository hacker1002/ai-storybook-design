# generate-visual-description-character

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (generate-spread-visual-plan) để tối ưu visual description.

## Description
**Visual Descriptor Agent** - Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character.

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value, title
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
```typescript
interface GenerateVisualDescriptionCharacterParams {
  storyId: string;         // ID của story trong DB
  snapshotId: string;      // ID của snapshot trong DB
  key: string;             // Character key (e.g., "miu_cat"), dùng để lấy thông tin từ snapshot.characters[]
  targetLength: "short" | "medium" | "detailed";  // short: 50-80, medium: 80-120, detailed: 120-200 words
  language?: string;       // Ngôn ngữ output - fallback: story.original_language
}
```

## Result
```typescript
interface GenerateVisualDescriptionCharacterResult {
  success: boolean;
  visualDescription: string;      // Mô tả đã tối ưu cho image generation
  keywords: string[];             // Tags cho search/tagging (5-10 từ khóa)
  negativePrompt: string;         // Những gì cần tránh (luôn trả về)
  suggestedReferences: string[];  // Gợi ý tìm ảnh reference (2-3 gợi ý)
}
```

## Prompt

> **DB Template Names:**
> - System: `VISUAL_DESCRIPTOR_SYSTEM` (shared across all visual description functions)
> - User: `VISUAL_DESC_CHARACTER_USER_TEMPLATE`

### System Prompt (VISUAL_DESCRIPTOR_SYSTEM)
```
You are a visual description writer for children's picture book illustrations.
Transform entity information into vivid, detailed descriptions optimized for AI image generation.

Core Rules:
- Write in continuous prose with comma-separated descriptive phrases
- Be specific: avoid vague terms like "beautiful", "nice", "lovely"
- Include: colors, textures, proportions, lighting, mood, materials
- Keep child-friendly and age-appropriate
- Focus on distinctive visual features for consistency across images
- Consider how the entity will appear in various scenes
- Always provide a negative prompt to avoid unwanted elements

Output Format:
- visualDescription: Detailed prose description
- keywords: 5-10 searchable tags
- negativePrompt: What to avoid in generation
- suggestedReferences: 2-3 search terms for reference images
```

### User Prompt Template (VISUAL_DESC_CHARACTER_USER_TEMPLATE)
```
Generate a visual description for a character:

## Character Information
**Name:** {%name%}
**Key:** @{%key%}
**Description:** {%description%}

### Basic Info
- Gender: {%gender%}
- Age: {%age%}
- Category: {%category_name%} ({%category_type%}) - {%category_description%}
- Role: {%role%}

### Personality (for expression/pose guidance)
{%personality_text%}

### Appearance
{%appearance_text%}

## Art Style
**Style Reference:** {%art_style_description%}

## Story Context
- Title: {%title%}
- Genre: {%genre%}
- Target Audience: {%target_audience%}
- Core Value: {%target_core_value%}

### Existing Character Descriptions (for consistency)
{%existing_visual_descriptions%}

## Output Requirements
- **Target Length:** {%target_length%}
- **Language:** {%language%}

---

Generate JSON response:
{
  "visualDescription": "...",
  "keywords": ["...", "..."],
  "negativePrompt": "...",
  "suggestedReferences": ["...", "..."]
}
```

## Flow
```
1. Validate input parameters (storyId, snapshotId, key, targetLength)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "VISUAL_DESCRIPTOR_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_CHARACTER_USER_TEMPLATE" → user prompt
3. Lấy story info từ DB:
   - SELECT artstyle_id, original_language, target_audience, genre, target_core_value, title
   - FROM stories WHERE id = storyId
4. Lấy character từ snapshot.characters[] WHERE key = params.key
5. Lấy category từ asset_categories WHERE id = character.basic_info.category_id
   → name, type, description
6. Lấy art_style từ art_styles WHERE id = story.artstyle_id
   → description
7. Lấy existing visual descriptions từ các characters khác trong snapshot
   → Để đảm bảo consistency
8. Determine language: params.language || story.original_language
9. Render user prompt template với variables
10. Call LLM với system prompt và rendered user prompt
11. Parse JSON response
12. Return result
```

## Error Handling
- Nếu story/snapshot không tồn tại → Return error
- Nếu character với key không tìm thấy → Return error với available keys
- Nếu category_id invalid → Log warning, tiếp tục với empty category info
- Nếu art_style không tìm thấy → Return error (art_style bắt buộc)
