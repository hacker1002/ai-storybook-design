# generate-visual-description-stage

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (generate-spread-visual-plan) để tối ưu visual description.

## Description
**Visual Descriptor Agent** - Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background.

## DB Schema Dependencies

### Tables Used
- `stories`: artstyle_id, original_language, target_audience, genre, target_core_value, title, era_id, location_id
- `snapshots`: stages[]
- `locations`: id, name, description, nation, city, type, image_references[]
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
```typescript
interface GenerateVisualDescriptionStageParams {
  storyId: string;         // ID của story trong DB
  snapshotId: string;      // ID của snapshot trong DB
  key: string;             // Stage key (e.g., "forest_1"), dùng để lấy thông tin từ snapshot.stages[]
  targetLength: "short" | "medium" | "detailed";  // short: 50-80, medium: 80-120, detailed: 120-200 words
  language?: string;       // Ngôn ngữ output - fallback: story.original_language
}
```

## Result
```typescript
interface GenerateVisualDescriptionStageResult {
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
> - User: `VISUAL_DESC_STAGE_USER_TEMPLATE`

### System Prompt
*(Reuse `VISUAL_DESCRIPTOR_SYSTEM` - shared với character, prop, spread)*

### User Prompt Template (VISUAL_DESC_STAGE_USER_TEMPLATE)
```
Generate a visual description for a stage/background:

## Stage Information
**Name:** {%name%}
**Key:** @{%key%}
**Current Description:** {%current_description%}

### Location Details
- Location: {%location_name%} - {%location_description%}
- Nation: {%location_nation%}
- City: {%location_city%}
- Type: {%location_type%} (0: địa điểm thật, 1: địa điểm hư cấu)
- Location References:
{%location_image_references%}

## Art Style
**Style Reference:** {%art_style_description%}

## Story Context
- Title: {%title%}
- Genre: {%genre%}
- Target Audience: {%target_audience%}
- Core Value: {%target_core_value%}
- Era: {%era_name%} - {%era_description%}
- Story Location Setting: {%story_location_name%} - {%story_location_description%}

### Existing Stage Descriptions (for consistency)
{%existing_visual_descriptions%}

## Output Requirements
- **Target Length:** {%target_length%}
- **Language:** {%language%}

---

Guidelines for stage descriptions:
- Describe foreground, midground, and background elements
- Include lighting conditions and atmosphere
- Consider how characters will interact with the environment
- Maintain consistency with story's era and location setting

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
   - Query `prompt_templates` với name = "VISUAL_DESC_STAGE_USER_TEMPLATE" → user prompt
3. Lấy story info từ DB:
   - SELECT artstyle_id, original_language, target_audience, genre, target_core_value, title, era_id, location_id
   - FROM stories WHERE id = storyId
4. Lấy stage từ snapshot.stages[] WHERE key = params.key
5. Lấy location từ locations WHERE id = stage.location_id
   → name, description, nation, city, type, image_references
6. Lấy era từ eras WHERE id = story.era_id
   → name, description
7. Lấy story location từ locations WHERE id = story.location_id
   → name, description (story-level setting)
8. Lấy art_style từ art_styles WHERE id = story.artstyle_id
   → description
9. Lấy existing visual descriptions từ các stages khác trong snapshot
   → Để đảm bảo consistency
10. Determine language: params.language || story.original_language
11. Render user prompt template với variables
12. Call LLM với system prompt và rendered user prompt
13. Parse JSON response
14. Return result
```

## Error Handling
- Nếu story/snapshot không tồn tại → Return error
- Nếu stage với key không tìm thấy → Return error với available keys
- Nếu location_id invalid → Log warning, tiếp tục với empty location info
- Nếu era_id invalid → Log warning, tiếp tục với empty era info
- Nếu art_style không tìm thấy → Return error (art_style bắt buộc)
