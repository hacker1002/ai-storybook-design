# generate-visual-description-spread

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (generate-spread-visual-plan) để tối ưu visual description.

## Description
**Visual Descriptor Agent** - Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage).

## DB Schema Dependencies

### Tables Used
- `books`: artstyle_id, original_language, target_audience, format_genre, content_genre, target_core_value, title
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
  - images[]: { title, geometry, visual_description, stage, actions, temporal, sensory, emotional, image_references[], sketch[], illustration[], final_hires_media_url }
  - objects[], animations[], textboxes[]

- `characters[]`, `props[]`, `stages[]` structures (referenced via @key)

## Parameters
```typescript
interface GenerateVisualDescriptionSpreadParams {
  bookId: string;          // ID của book trong DB
  snapshotId: string;      // ID của snapshot trong DB
  spreadNumber: number;    // Spread number (1-based)
  targetLength: "short" | "medium" | "detailed";  // short: 50-80, medium: 80-120, detailed: 120-200 words
  language?: string;       // Ngôn ngữ output - fallback: book.original_language
}
```

## Result
```typescript
interface GenerateVisualDescriptionSpreadResult {
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
> - User: `VISUAL_DESC_SPREAD_USER_TEMPLATE`

### System Prompt
*(Reuse `VISUAL_DESCRIPTOR_SYSTEM` - shared với character, prop, stage)*

### User Prompt Template (VISUAL_DESC_SPREAD_USER_TEMPLATE)
```
Generate a visual description for a spread scene:

## Spread Information
**Spread Number:** {%spread_number%} of {%total_spreads%}
**Left Page:** {%left_page_number%} (type: {%left_page_type%})
**Right Page:** {%right_page_number%} (type: {%right_page_type%})

## Scene Composition

### Image Details
{%images_text%}

### Text Content (for context)
{%textboxes_text%}

### Characters in Scene
{%characters_in_scene%}

### Stage/Background
{%stage_info%}

### Props in Scene
{%props_in_scene%}

## Art Style
**Style Reference:** {%art_style_description%}

## Story Context
- Title: {%title%}
- Format Genre: {%format_genre%}
- Content Genre: {%content_genre%}
- Target Audience: {%target_audience%}
- Core Value: {%target_core_value%}

### Previous Spread (for continuity)
{%previous_spread_description%}

## Output Requirements
- **Target Length:** {%target_length%}
- **Language:** {%language%}

---

Guidelines for spread descriptions:
- Describe the scene as a single cohesive illustration
- Focus on the key moment/action of the scene
- Include character positions, expressions, and interactions
- Maintain consistency with existing character and stage descriptions
- Consider temporal and sensory elements

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
1. Validate input parameters (bookId, snapshotId, spreadNumber, targetLength)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "VISUAL_DESCRIPTOR_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_SPREAD_USER_TEMPLATE" → user prompt
3. Lấy book info từ DB:
   - SELECT artstyle_id, original_language, target_audience, format_genre, content_genre, target_core_value, title
   - FROM books WHERE id = bookId
4. Lấy spread từ snapshot.spreads[] WHERE number = spreadNumber
5. Parse @key trong spread.images[].stage và spread.images[].actions
   → Lấy danh sách character keys, prop keys, stage keys liên quan
6. Lấy visual descriptions của entities từ snapshot:
   - characters[].visual_description WHERE key IN character_keys
   - props[].visual_description WHERE key IN prop_keys
   - stages[].visual_description WHERE key IN stage_keys
7. Với mỗi stage liên quan, lấy location info từ locations WHERE id = stage.location_id
   → name, description
8. Lấy art_style từ art_styles WHERE id = book.artstyle_id
   → description
9. Lấy previous spread description nếu spreadNumber > 1
   → spreads[spreadNumber - 2].images[0].visual_description (để đảm bảo continuity)
10. Determine language: params.language || book.original_language
11. Format all variables:
    - images_text: từ spread.images[]
    - textboxes_text: từ spread.textboxes[].language[].text
    - characters_in_scene: từ characters liên quan
    - stage_info: từ stages liên quan + location info
    - props_in_scene: từ props liên quan
12. Render user prompt template với variables
13. Call LLM với system prompt và rendered user prompt
14. Parse JSON response
15. Return result
```

## Error Handling
- Nếu book/snapshot không tồn tại → Return error
- Nếu spread với spreadNumber không tìm thấy → Return error với valid range
- Nếu referenced entities không tìm thấy → Log warning, tiếp tục với available data
- Nếu art_style không tìm thấy → Return error (art_style bắt buộc)
