# generate-text-refinement

## Description
**Word Smith Agent** - Biên tập và tinh chỉnh nội dung text trong textboxes[] của mỗi spread. Agent này tập trung vào chất lượng ngôn ngữ, rhythm, và sự phù hợp với target audience. KHÔNG xử lý vị trí/geometry.

## DB Schema Dependencies

### Tables Referenced
- `story`: Truy vấn target_audience, original_language
- `snapshot`: Đọc docs[], spreads[] và UPDATE spreads[].textboxes[].language[].text

### Fields Used/Updated
- `snapshot.docs[]` - READ (để hiểu context)
- `snapshot.spreads[].textboxes[].language[].text` - READ & UPDATE
- `story.target_audience` - READ
- `story.original_language` - READ

## Parameters
```typescript
interface GenerateTextRefinementParams {
  storyId: string;
  snapshotId: string;
}
```

## Result
```typescript
interface GenerateTextRefinementResult {
  success: boolean;
  updatedSpreads: {
    number: number;
    textboxes: {
      order: number;
      originalText: string;
      refinedText: string;
      changes: string[];        // List of changes made
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
- Target Audience: {%audience%}
- Language: {%language%}

## MANUSCRIPT REFERENCE
{ // from snapshot.docs[] where type = "manuscript":
- content: "Full manuscript content..."
}
{%manuscript_content%}

## CURRENT SPREADS TEXT
{ // for each spread, show current textboxes[].language[].text:
- spread 1: textbox 1: "...", textbox 2: "..."
- spread 2: textbox 1: "..."
- ...
}
{%spreads_text%}

---

## REFINEMENT GUIDELINES

### For Target Audience: {audience}

**kids-3-6 (Preschool):**
- Simple sentences (5-10 words max)
- Repetition and rhythm (great for read-aloud)
- Familiar vocabulary
- Onomatopoeia encouraged
- Total per spread: 20-40 words max

**kids-7-12 (Primary):**
- Varied sentence structure
- Richer vocabulary (with context clues)
- Can handle compound sentences
- Total per spread: 40-80 words max

**teens:**
- Complex sentences allowed
- Sophisticated vocabulary
- Subtext and nuance
- Total per spread: 60-100 words max

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
      "originalText": "Ngày xửa ngày xưa, có một chú mèo con tên là Miu sống trong một ngôi nhà nhỏ.",
      "refinedText": "Miu là một chú mèo con sống trong ngôi nhà nhỏ xinh.",
      "changes": [
        "Bỏ 'Ngày xửa ngày xưa' - cliché không cần thiết",
        "Rút gọn câu cho phù hợp lứa tuổi 3-6"
      ]
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
1. Validate input parameters (storyId, snapshotId)
2. Lấy prompt templates từ DB:
   - Query `prompt_templates` với name = "WORD_SMITH_SYSTEM" → system prompt
   - Query `prompt_templates` với name = "TEXT_REFINEMENT_USER_TEMPLATE" → user prompt template
3. Lấy snapshot data từ DB (docs, spreads)
4. Lấy story metadata (target_audience, original_language)
5. Render user prompt template với variables:
   - title, audience, language, manuscript_content, spreads_text
6. Call LLM với system prompt và rendered user prompt
7. Parse JSON response
8. Update snapshot.spreads[].textboxes[].language[].text
9. Return result với summary
```

## Error Handling
- Nếu snapshot không tồn tại → Return error
- Nếu không có textboxes → Return success với empty changes
- Nếu LLM không trả về đủ spreads → Log warning, giữ nguyên text chưa được refine
