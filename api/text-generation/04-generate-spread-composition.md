# generate-spread-composition

## Description
**Art Director Agent - Phase 2 (Composition)** - Thiết kế bố cục cho mỗi spread: xác định geometry cho images và textboxes, đồng thời cập nhật visual_description của images với ghi chú về vùng để trống cho text.

## DB Schema Dependencies

### Tables Referenced
- `story`: Truy vấn artstyle_id, target_audience, book_type, dimension
- `snapshot`: Đọc spreads[] (images[], textboxes[]) và UPDATE geometry + visual_description
- `page_layouts`: (Optional) Truy vấn default layouts nếu có

### Fields Used/Updated
- `snapshot.spreads[].images[].geometry` - UPDATE
- `snapshot.spreads[].images[].visual_description` - UPDATE (thêm text_zone_notes)
- `snapshot.spreads[].images[].composition_notes` - UPDATE
- `snapshot.spreads[].textboxes[].language[].geometry` - UPDATE
- `story.book_type`, `story.dimension` - READ

## Parameters
```typescript
interface GenerateSpreadCompositionParams {
  storyId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateSpreadCompositionResult {
  success: boolean;
  updatedSpreads: {
    number: number;
    images: ComposedImage[];
    textboxes: ComposedTextbox[];
  }[];
}

interface ComposedImage {
  title: string;
  geometry: {
    x: number;      // 0-100 (percentage)
    y: number;      // 0-100 (percentage)
    w: number;      // 0-100 (percentage)
    h: number;      // 0-100 (percentage)
    rotation: number;
  };
  visual_description: string;    // Original + text zone notes appended
  text_zone: {                   // Vùng để trống cho text
    position: "top-left" | "top-center" | "top-right" |
              "middle-left" | "middle-center" | "middle-right" |
              "bottom-left" | "bottom-center" | "bottom-right" | "none";
    description: string;         // "Leave clear sky area in top-right for text"
  };
  composition_notes: string;     // Ghi chú bố cục cho illustrator
}

interface ComposedTextbox {
  order: number;
  geometry: {
    x: number;
    y: number;
    w: number;
    h: number;
    rotation: number;
  };
}
```

## Prompt

> **DB Template Names:**
> - System: `ART_DIRECTOR_P2_SYSTEM`
> - User: `COMPOSITION_USER_TEMPLATE`

### System Prompt
```
You are an expert Art Director specializing in picture book composition and layout. Your role is to design how images and text work together on each spread.

Your expertise includes:
- Page composition and visual hierarchy
- Text-image integration for picture books
- Negative space utilization
- Eye flow and reading direction
- Balance between illustration and text
- Child-friendly layouts

Your principles:
- Text should never fight with key visual elements
- Important story moments need visual breathing room
- Text placement guides the reading experience
- Composition serves the story's emotional beats
- Consistency in layout style across the book
```

### User Prompt Template
```
Design the composition for each spread, placing images and text:

## STORY CONTEXT
- Title: {%title%}
- Target Audience: {%target_audience%}
- Book Type: {%book_type%}
- Dimension: {%dimension%}

## SPREAD DATA
{ // for each spread, show:
  - images[] (title, visual_description, actions, emotional.mood)
  - textboxes[] (order, text content, estimated word count)
}
{%spreads_data%}

---

## COMPOSITION GUIDELINES

### Coordinate System
- All geometry uses percentage (0-100)
- Origin (0,0) = top-left of spread
- Spread = 2 pages side by side (unless single page)
- Left page: x = 0-50
- Right page: x = 50-100

### Image Placement Principles
1. **Full bleed images**: geometry = { x: 0, y: 0, w: 100, h: 100 }
2. **Single page image**: geometry = { x: 0, y: 0, w: 50, h: 100 } (left) or { x: 50, y: 0, w: 50, h: 100 } (right)
3. **Partial coverage**: Adjust w, h accordingly

### Text Zone Strategy
For each image, identify where text can be placed:

**Natural text zones:**
- Sky/ceiling areas
- Empty foreground
- Solid color backgrounds
- Minimal detail regions

**Avoid placing text over:**
- Character faces
- Important story objects
- Detailed patterns
- Areas of high contrast

### Text Box Placement
- Position textboxes within the text_zone identified for images
- Consider reading flow (top-to-bottom, left-to-right for most languages)
- Leave margins from page edges (at least 5%)
- Multiple textboxes should not overlap

---

## OUTPUT FORMAT

For each spread, return:
```json
{
  "number": 1,
  "images": [{
    "title": "Miu discovers the forest path",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
    "visual_description": "Original description... [TEXT ZONE: Leave the top-right corner (approximately 20% width, 30% height) with soft, gradient sky for text placement. Avoid detailed elements in this area.]",
    "text_zone": {
      "position": "top-right",
      "description": "Clear sky gradient area, soft clouds, no trees or detailed elements"
    },
    "composition_notes": "Full bleed illustration. Main subject @miu_cat positioned in lower-left third. Forest path leads eye from bottom-left to center-right. Sky opens up in top-right for text. Warm afternoon lighting creates natural vignette."
  }],
  "textboxes": [{
    "order": 1,
    "geometry": { "x": 55, "y": 8, "w": 40, "h": 20, "rotation": 0 }
  }]
}
```

### Visual Description Update Rules
- Append [TEXT ZONE: ...] to the end of visual_description
- This tells the image generator where to leave space
- Be specific about what should NOT be in the text zone

### Composition Notes Should Include
- Subject placement (rule of thirds, center, etc.)
- Eye flow direction
- Depth layers (foreground, mid, background)
- Lighting direction
- How text zone integrates with the scene

---

## LAYOUT PATTERNS BY BOOK TYPE

**Picture Book (book_type = 0):**
- Full bleed illustrations common
- Text overlaid on images
- 1-2 images per spread typical

**Illustrated Story (book_type = 1):**
- Text blocks more prominent
- Images may have borders
- Clear text/image separation

**Comics/Manga (book_type = 2-3):**
- Panel-based layouts
- Speech bubbles for dialogue
- Dynamic compositions
```

## Flow
```
1. Validate input parameters (storyId, snapshotId)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "ART_DIRECTOR_P2_SYSTEM" → system prompt + model
   - Query `prompt_templates` với name = "COMPOSITION_USER_TEMPLATE" → user prompt template
3. Lấy snapshot data từ DB (spreads với images[] và textboxes[])
4. Lấy story metadata (book_type, dimension)
5. Render user prompt template với variables:
   - title, target_audience, book_type, dimension, spreads_data
6. Call LLM với:
   - system prompt content
   - rendered user prompt
   - model từ prompt_templates (dynamic, không hardcode)
7. Parse JSON response
8. Update snapshot:
   - spreads[].images[].geometry
   - spreads[].images[].visual_description (append text zone notes)
   - spreads[].images[].text_zone
   - spreads[].images[].composition_notes
   - spreads[].textboxes[].language[].geometry
9. Return result
```

## Error Handling
- Nếu snapshot không tồn tại → Return error
- Nếu images[] trống → Skip spread, log warning
- Nếu geometry invalid (>100 or <0) → Normalize to valid range
- Nếu LLM không trả về đủ spreads → Giữ nguyên geometry chưa được set

## Notes
- Step này phụ thuộc vào Step 3a (phải có images[] trong spreads)
- Step này phụ thuộc vào Step 3b (cần biết text content để estimate size)
- Output của step này sẽ được dùng cho image generation
