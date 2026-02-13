# generate-dummy

## Description
**Dummy Generator Agent** - Tạo dữ liệu cụ thể cho từng trang truyện dựa trên script. Agent này lựa chọn layout phù hợp cho tỷ lệ tranh và chữ cho mỗi spread/page, sau đó generate geometry (vị trí textbox, image), text, và art notes.

**Entry point:** POST `/api/dummy/generate-dummy`

## DB Schema Dependencies

### Tables Referenced
- `prompt_templates`: Lấy CHOOSING_DUMMY_LAYOUT_SKILL, GENERATE_DUMMY_SYSTEM prompts + model
- `template_layouts`: Query danh sách layouts để AI chọn phù hợp

## Parameters

```typescript
interface Attachment {
  filename: string;                          // Tên file gốc
  mimeType: string;                          // MIME type (image/png, application/pdf, etc.)
  base64Data: string;                        // Base64 encoded content (max 10MB per file)
}

interface GenerateDummyRequest {
  // Required - Script input
  script: string;                           // Kịch bản đã được phân trang từ generate-script

  // Required - User prompt (refinement/direction)
  prompt: string;                           // Hướng dẫn chi tiết từ tác giả

  // Optional - Attachments
  attachments?: Attachment[];               // File đính kèm (reference materials, images, etc.)

  // Required - LLM Context
  llmContext: {
    // Story attributes (required)
    targetAudience: 1 | 2 | 3 | 4;          // 1: kindergarten, 2: preschool, 3: primary, 4: middle grade
    targetCoreValue: number;                 // 1-21: Core values
    formatGenre: 1 | 2 | 3 | 4 | 5 | 6;     // Narrative, Lullaby, Concept, Non-fiction, Early Reader, Wordless
    contentGenre: number;                    // 1-11: Mystery, Fantasy, etc.
  };
}
```

## Result

```typescript
interface PageGeometry {
  x: number;                                // % position relative to page (0-100)
  y: number;                                // % position relative to page (0-100)
  w: number;                                // % width relative to page (0-100)
  h: number;                                // % height relative to page (0-100)
}

interface TextboxData {
  id: string;                               // UUID
  geometry: PageGeometry;
  text: string;                             // Text content cho textbox này
  font_size_pt: number;                     // Font size in points
  text_alignment: "left" | "center" | "right";
  line_height: number;                      // Line height multiplier
}

interface ImageData {
  id: string;                               // UUID
  geometry: PageGeometry;
  art_note: string;                         // Art description/instruction
  edge_treatment?: "bleed" | "spot" | "vignette" | "crop";
  shape?: "circle" | "oval";                // For crop edge treatment
}

interface SpreadData {
  id: string;                               // UUID
  type: 1 | 0;                              // 1: double page spread, 0: single pages
  number: number;                           // Spread number
  layout: string;                           // template_layout ID
  left_page?: {
    number: number;
    type: string;                           // "normal_page" | "front_matter" | ...
    layout?: string;                        // template_layout ID (optional)
  };
  right_page?: {
    number: number;
    type: string;
    layout?: string;                        // template_layout ID (optional)
  };
  textboxes: TextboxData[];                 // All textboxes on this spread
  images: ImageData[];                      // All images on this spread
  manuscript: string;                       // Original script text for this spread
}

interface GenerateDummyResult {
  success: boolean;
  data: {
    spreads: SpreadData[];                  // Array of spreads with layout + geometry
    total_pages: number;                    // Total page count
    type: "prose" | "poetry";               // Text format type
  };
  meta?: {
    processingTime?: number;                // ms
    tokenUsage?: number;
  };
}
```

## Prompt Templates

| Type | Name | Description |
|------|------|-------------|
| System | `GENERATE_DUMMY_SYSTEM` | User prompt + script context + output format |
| Skill | `CHOOSING_DUMMY_LAYOUT_SKILL` | Template catalog + geometry calculation logic |

**Skill Reference:** `@@Skill sử dụng: CHOOSING_DUMMY_LAYOUT_SKILL`

## Flow

```
1. Validate input
   - script: required, non-empty string
   - prompt: required, non-empty string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre
   - attachments: optional array

2. Fetch template layouts
   - Query template_layouts table for available layouts
   - Build layout catalog with IDs, titles, types (spread/single), slot structures
   - Include in AI context

3. Build prompts (Prompt Build Logic)
   a. Fetch system prompt: GENERATE_DUMMY_SYSTEM
   b. Extract skill names: parse "@@Skill sử dụng: ..." → ['CHOOSING_DUMMY_LAYOUT_SKILL']
   c. Fetch skill prompts from DB
   d. Combine: skillPrompts + systemPrompt

4. Enrich with context
   - Append Story Attributes:
     • targetAudience → TARGET_AUDIENCE_MAP
     • targetCoreValue → TARGET_CORE_VALUE_MAP
     • formatGenre → FORMAT_GENRE_MAP
     • contentGenre → CONTENT_GENRE_MAP
   - Append layout catalog from step 2

5. Render variables
   - Replace {%request.prompt%} with user input
   - Replace {%request.script%} with script content
   - Replace {%request.attachments%} with attachments metadata
   - IF attachments: Include as inlineData parts in LLM request

6. Call LLM
   - model: gemini-3-pro (from prompt_templates.model)
   - thinking_level: high
   - temperature: 1
   - responseMimeType: application/json

7. Parse AI response
   - Validate JSON structure matches SpreadData[] schema
   - Ensure all required fields present
   - Validate geometry values (0-100 range)

8. Return { success: true, data: { spreads, total_pages, type }, meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Invalid JSON**: If AI returns invalid JSON, retry once
- **Missing Layouts**: If template_layouts query fails, return 500

## References

- [Prompt Template System](../README.md#prompt-template-system) - Variable syntax, skill reference pattern
- [Enum Mappings](../README.md#enum-mappings) - TARGET_AUDIENCE_MAP, etc.
- [Template Layouts Database](../../DATABASE-SCHEMA.md#template_layouts) - Layout structure

## Notes
- API generates dummy data but does NOT save to DB
- Client stores dummy data in `snapshots.dummies[]` field
- Geometry uses percentage coordinates (0-100) for responsive design
- Layout selection based on text/image ratio and page content
- AI autonomously selects appropriate template_layout IDs from catalog
- Supports both single pages and double-page spreads
- Text can be prose or poetry format (AI determines based on script)
