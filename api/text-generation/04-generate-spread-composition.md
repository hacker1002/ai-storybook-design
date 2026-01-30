# generate-spread-composition

## Description
**Art Director Agent - Phase 2 (Composition)** - Thiết kế bố cục cho mỗi spread: xác định geometry cho images và textboxes, đồng thời cập nhật visual_description của images với ghi chú về vùng để trống cho text.

## DB Schema Dependencies

### Tables Referenced
- `stories`: Truy vấn target_audience, book_type
- `snapshots`: Đọc spreads[] (images[], textboxes[]) và UPDATE geometry + visual_description

### Fields Used/Updated
- `snapshots.spreads[].images[].geometry` - UPDATE
- `snapshots.spreads[].images[].visual_description` - UPDATE (append text zone notes for image gen)
- `snapshots.spreads[].textboxes[].language[0].geometry` - UPDATE (original_language only)
- `stories.title`, `stories.book_type`, `stories.target_audience` - READ

## Parameters
```typescript
interface GenerateSpreadCompositionInput {
  storyId: string;
  snapshotId: string;
}
```

## Book Type Mapping
| Value | Name | Layout Style |
|-------|------|--------------|
| 1 | Picture Book (Sách tranh) | Full bleed illustrations, text overlaid |
| 2 | Illustrated Story (Truyện chữ có hình) | Text blocks prominent, images with borders |
| 3 | Comics | Panel-based layouts, speech bubbles |
| 4 | Manga | Right-to-left reading, manga panels |

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
    y: number;
    w: number;
    h: number;
  };
  visual_description: string;    // Original + [TEXT ZONE: ...] appended
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

## SPREAD DATA
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

**Note:** Images don't have `rotation` - only textboxes support rotation.

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
- Consider reading flow (top-to-bottom, left-to-right)
- Leave margins from page edges (at least 5%)
- Multiple textboxes should not overlap

---

## OUTPUT FORMAT

Return a JSON object with `spreads` array containing all composed spreads:
{
  "spreads": [
    {
      "number": 1,
      "images": [{
        "title": "Miu discovers the forest path",
        "geometry": { "x": 0, "y": 0, "w": 100, "h": 100 },
        "visual_description": "Original description... [TEXT ZONE: Leave the top-right corner (approximately 20% width, 30% height) with soft, gradient sky for text placement.]"
      }],
      "textboxes": [{
        "order": 1,
        "geometry": { "x": 55, "y": 8, "w": 40, "h": 20, "rotation": 0 }
      }]
    }
  ]
}

### Visual Description Update Rules
- Append [TEXT ZONE: ...] to the end of visual_description
- This tells the image generator where to leave space
- Be specific about what should NOT be in the text zone

---

## LAYOUT PATTERNS BY BOOK TYPE

**Picture Book (book_type = 1):**
- Full bleed illustrations common
- Text overlaid on images
- 1-2 images per spread typical

**Illustrated Story (book_type = 2):**
- Text blocks more prominent
- Images may have borders
- Clear text/image separation

**Comics (book_type = 3):**
- Panel-based layouts
- Speech bubbles for dialogue
- Dynamic compositions

**Manga (book_type = 4):**
- Right-to-left reading
- Manga panel conventions
- Dynamic action panels
```

## Flow
```
1. Validate input parameters (storyId, snapshotId)

2. Lấy prompt templates từ DB:
   - ART_DIRECTOR_P2_SYSTEM → system prompt + model
   - COMPOSITION_USER_TEMPLATE → user prompt template

3. Lấy snapshot data từ DB (spreads với images[] và textboxes[])

4. Lấy story metadata (title, book_type, target_audience)

5. Render user prompt template với variables:
   - title, target_audience, book_type, spreads_data

6. Call LLM

7. Parse JSON response

8. Update snapshot:
   - spreads[].images[].geometry
   - spreads[].images[].visual_description (append [TEXT ZONE: ...])
   - spreads[].textboxes[].language[0].geometry (original_language only)

9. Return result
```

## Error Handling
- Snapshot không tồn tại → Return error
- images[] trống → Skip spread, log warning
- Geometry invalid (>100 or <0) → Normalize to valid range
- LLM không trả về đủ spreads → Giữ nguyên geometry chưa được set
