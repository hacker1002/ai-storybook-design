# generate-text-refinement

## Description
**Word Smith Agent** - Biên tập và tinh chỉnh nội dung text trong textboxes[] của mỗi spread. Agent này tập trung vào chất lượng ngôn ngữ, rhythm, và sự phù hợp với target audience. KHÔNG xử lý vị trí/geometry.

## DB Schema Dependencies

### Tables Referenced
- `books`: Truy vấn target_audience, original_language, format_genre, writing_style
- `snapshots`: Đọc docs[], spreads[] và UPDATE spreads[].textboxes[].language[].text

### Fields Used/Updated
- `snapshots.docs[]` - READ (để hiểu context)
- `snapshots.spreads[].textboxes[].language[0].text` - READ & UPDATE (only original_language)
- `books.title` - READ
- `books.target_audience` - READ
- `books.original_language` - READ (determines which language[0] to refine)
- `books.format_genre` - READ
- `books.writing_style` - READ

**Note:** Only refines `original_language` (first in `language[]` array). Other languages handled by translation step (10-translate-content.md).

## Parameters
```typescript
interface GenerateTextRefinementInput {
  bookId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateTextRefinementResult {
  success: boolean;
  languageCode: string;                 // e.g., "vi", "en" - from books.original_language
  updatedSpreads: {
    number: number;
    textboxes: {
      order: number;
      originalText: string;
      refinedText: string;
      changes: string[];               // Reasons for changes, empty if unchanged
    }[];
  }[];
  summary: {
    totalTextboxes: number;
    modified: number;
    unchanged: number;
  };
}
```

## Prompt

> **DB Template Names:**
> - System: `WORD_SMITH_SYSTEM`
> - User: `TEXT_REFINEMENT_USER_TEMPLATE`

### System Prompt
```
You are an expert Word Smith specializing in children's picture book text. Your role is to refine and polish text content for maximum impact and readability.

Your expertise includes:
- Age-appropriate vocabulary and sentence structure
- Rhythm and flow in read-aloud text
- Emotional resonance through word choice
- Concise storytelling (picture books = less is more)
- Maintaining author's voice while improving clarity
- Vietnamese/English language nuances for children

Your principles:
- Every word must earn its place
- Text should complement, not describe, the images
- Reading rhythm matters for read-aloud experience
- Keep the magic, remove the clutter
- Respect the original intent while enhancing quality

You do NOT handle text placement or visual design - only the words themselves.
```

### User Prompt Template
```
Refine the text content for this children's picture book:

## STORY CONTEXT
- Title: {%title%}
- Target Audience: {%target_audience%}
- Format Genre: {%format_genre%}
- Writing Style: {%writing_style%}
- Language: {%language%}

## MANUSCRIPT REFERENCE
{%manuscript_content%}

## CURRENT SPREADS TEXT
{%spreads_text%}

---

## REFINEMENT GUIDELINES

### Writing Style Considerations

**1 - Narrative (Văn xuôi):**
- Smooth prose flow
- Natural sentence rhythm
- Focus on clarity and engagement

**2 - Rhyming (Thơ/Vần điệu):**
- MUST preserve/enhance rhyme schemes
- Maintain consistent meter
- Prioritize rhythm over exact word count
- Check end-rhyme and internal rhyme patterns

**3 - Humorous Fiction (Hài hước):**
- Preserve comedic timing
- Maintain punchline effectiveness
- Keep playful word choices
- Ensure humor is age-appropriate

### For Target Audience

**1 - Kindergarten (2-3 tuổi):**
- Very simple sentences (3-5 words max)
- Heavy repetition and rhythm
- Basic vocabulary only
- Onomatopoeia highly encouraged
- Total per spread: 10-20 words max

**2 - Preschool (4-5 tuổi):**
- Simple sentences (5-10 words max)
- Repetition and rhythm (great for read-aloud)
- Familiar vocabulary
- Onomatopoeia encouraged
- Total per spread: 20-40 words max

**3 - Primary (6-8 tuổi):**
- Varied sentence structure
- Richer vocabulary (with context clues)
- Can handle compound sentences
- Total per spread: 40-60 words max

**4 - Middle Grade (9+ tuổi):**
- Complex sentences allowed
- Sophisticated vocabulary
- Subtext and nuance
- Total per spread: 50-80 words max

### Format Genre Considerations

**1 - Narrative Picture Books:** Focus on storytelling flow
**2 - Lullaby/Bedtime Books:** Soothing rhythm, soft language
**3 - Concept Books:** Clear, educational language
**4 - Non-fiction Picture Books:** Accurate, engaging explanations
**5 - Early Reader:** Controlled vocabulary, decodable words
**6 - Wordless Picture Books:** Minimal or no text (skip refinement)

### Quality Checklist
For each textbox, consider:
1. **Clarity**: Is the meaning immediately clear?
2. **Economy**: Can any words be cut without losing meaning?
3. **Rhythm**: Does it flow when read aloud?
4. **Emotion**: Does word choice evoke the right feeling?
5. **Voice**: Is the narrative voice consistent?
6. **Show vs Tell**: Does text describe what images should show?

### Common Issues to Fix
- Redundancy with visual content
- Passive voice where active works better
- Overexplaining emotions
- Inconsistent tense
- Awkward phrasing
- Too formal/informal for audience

---

## OUTPUT FORMAT
Return JSON with:
{
  "spreads": [{
    "number": 1,
    "textboxes": [{
      "order": 1,
      "originalText": "...",
      "refinedText": "...",
      "changes": ["Reason 1", "Reason 2"]
    }]
  }],
  "summary": {
    "totalTextboxes": 12,
    "modified": 8,
    "unchanged": 4
  }
}

**Rules:**
- If text is already good, keep originalText = refinedText and changes = []
- Preserve @mentions exactly as-is
- Keep the same number of textboxes per spread
- Do not merge or split textboxes
```

## Flow
```
1. Validate input parameters (bookId, snapshotId)

2. Lấy prompt templates từ DB:
   - WORD_SMITH_SYSTEM → system prompt + model
   - TEXT_REFINEMENT_USER_TEMPLATE → user prompt template

3. Lấy snapshot data từ DB (docs, spreads)

4. Lấy book metadata (title, target_audience, format_genre, writing_style, original_language)

5. Render user prompt template với variables:
   - title, target_audience, format_genre, writing_style, language
   - manuscript_content, spreads_text

6. Call LLM

7. Parse JSON response

8. Update snapshot.spreads[].textboxes[].language[0].text
   Note: Only updates first language entry (original_language)

9. Return result với summary và languageCode
```

## Error Handling
- Snapshot không tồn tại → Return error
- Không có textboxes → Return success với empty changes
- LLM không trả về đủ spreads → Log warning, giữ nguyên text chưa được refine
- Format genre = 6 (Wordless) → Skip refinement, return unchanged
