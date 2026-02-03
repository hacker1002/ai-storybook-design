# generate-visual-description-prop

> **Note:** Function này có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (generate-spread-visual-plan) để tối ưu visual description.

## Description
**Visual Descriptor Agent** - Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop.

## DB Schema Dependencies

### Tables Used
- `books`: artstyle_id, original_language, target_audience, format_genre, content_genre, target_core_value, title
- `snapshots`: props[]
- `asset_categories`: id, name, type, description
- `art_styles`: id, name, description, image_references[]

### JSONB Fields
- `props[]` structure (from snapshots):
  - order, name, key
  - category_id
  - visual_description
  - type: "narrative" | "anchor"
    - narrative: đồ vật dẫn chuyện, tương tác với character
    - anchor: đồ vật nằm trong stages, tạo sự nhất quán
  - sketch[], illustration[], image_references[]
  - sounds[]: [{ title, media_url }]

## Parameters
```typescript
interface GenerateVisualDescriptionPropParams {
  bookId: string;          // ID của book trong DB
  snapshotId: string;      // ID của snapshot trong DB
  key: string;             // Prop key (e.g., "red_bow"), dùng để lấy thông tin từ snapshot.props[]
  targetLength: "short" | "medium" | "detailed";  // short: 50-80, medium: 80-120, detailed: 120-200 words
  language?: string;       // Ngôn ngữ output - fallback: book.original_language
}
```

## Result
```typescript
interface GenerateVisualDescriptionPropResult {
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
> - User: `VISUAL_DESC_PROP_USER_TEMPLATE`

### System Prompt
*(Reuse `VISUAL_DESCRIPTOR_SYSTEM` - shared với character, stage, spread)*

### User Prompt Template (VISUAL_DESC_PROP_USER_TEMPLATE)
```
Generate a visual description for a prop:

## Prop Information
**Name:** {%name%}
**Key:** @{%key%}
**Current Description:** {%current_description%}

### Details
- Category: {%category_name%} ({%category_type%}) - {%category_description%}
- Type: {%type%}
  - narrative: item that drives story, interacts with characters
  - anchor: item in stages, creates consistency

### Associated Sounds
{%sounds_text%}

## Art Style
**Style Reference:** {%art_style_description%}

## Story Context
- Title: {%title%}
- Format Genre: {%format_genre%}
- Content Genre: {%content_genre%}
- Target Audience: {%target_audience%}
- Core Value: {%target_core_value%}

### Existing Prop Descriptions (for consistency)
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
1. Validate input parameters (bookId, snapshotId, key, targetLength)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "VISUAL_DESCRIPTOR_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "VISUAL_DESC_PROP_USER_TEMPLATE" → user prompt
3. Lấy book info từ DB:
   - SELECT artstyle_id, original_language, target_audience, format_genre, content_genre, target_core_value, title
   - FROM books WHERE id = bookId
4. Lấy prop từ snapshot.props[] WHERE key = params.key
5. Lấy category từ asset_categories WHERE id = prop.category_id
   → name, type, description
6. Lấy art_style từ art_styles WHERE id = book.artstyle_id
   → description
7. Format sounds_text từ prop.sounds[]
8. Lấy existing visual descriptions từ các props khác trong snapshot
   → Để đảm bảo consistency
9. Determine language: params.language || book.original_language
10. Render user prompt template với variables
11. Call LLM với system prompt và rendered user prompt
12. Parse JSON response
13. Return result
```

## Error Handling
- Nếu book/snapshot không tồn tại → Return error
- Nếu prop với key không tìm thấy → Return error với available keys
- Nếu category_id invalid → Log warning, tiếp tục với empty category info
- Nếu art_style không tìm thấy → Return error (art_style bắt buộc)
