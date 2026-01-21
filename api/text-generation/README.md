# Text Generation Edge Functions

Overview of all text generation functions for AI Storybook Canvas.

## Manuscript Generation (Multi-Step Pipeline)

| # | Function | Agent | Description | File |
|---|----------|-------|-------------|------|
| 1 | generate-manuscript | Orchestrator | Điều phối việc tạo manuscript hoàn chỉnh qua 3 bước tuần tự. Function này gọi lần lượt 3 step functions và quản lý flow tổng thể. | [01-generate-manuscript.md](./01-generate-manuscript.md) |
| 2 | generate-story-draft | Story Teller | Phân tích ý tưởng và tạo khung truyện ban đầu bao gồm: character bible, prop bible, stage bible, phân tích nghệ thuật, và phân chia nội dung từng spread. Function này cũng tạo Story và Snapshot record trong DB. | [02-generate-story-draft.md](./02-generate-story-draft.md) |
| 3 | generate-art-direction | Art Director | Tạo hình (visual design) cho characters, props, stages và kịch bản hình ảnh chi tiết cho từng trang. | [03-generate-art-direction.md](./03-generate-art-direction.md) |
| 4 | generate-quality-check | Tester Agents | Kiểm tra chất lượng nội dung qua nhiều góc nhìn: Story Consistency, Plot, và Age Appropriateness. | [04-generate-quality-check.md](./04-generate-quality-check.md) |

## Visual Description Generation

| # | Function | Description | File |
|---|----------|-------------|------|
| 5 | generate-visual-description-character | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character. Có thể được sử dụng độc lập hoặc được gọi bởi Step 2 (Art Direction) để tối ưu visual description. | [05-generate-visual-description-character.md](./05-generate-visual-description-character.md) |
| 6 | generate-visual-description-prop | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop. | [06-generate-visual-description-prop.md](./06-generate-visual-description-prop.md) |
| 7 | generate-visual-description-stage | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background. | [07-generate-visual-description-stage.md](./07-generate-visual-description-stage.md) |
| 8 | generate-visual-description-spread | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage). | [08-generate-visual-description-spread.md](./08-generate-visual-description-spread.md) |

## Translation

| # | Function | Description | File |
|---|----------|-------------|------|
| 9 | translate-content | Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả. | [09-translate-content.md](./09-translate-content.md) |

## Pipeline Flow Diagram

```
User Input (Story Idea + Attributes)
        ↓
┌─────────────────────────────────────────────────┐
│ 1. generate-manuscript (ORCHESTRATOR)           │
│    - Validates input                             │
│    - Manages 3-step pipeline                     │
│    - Handles errors gracefully                   │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 1: generate-story-draft                    │
│         (Story Teller Agent)                     │
│                                                  │
│ Output:                                          │
│ ✓ Story + Snapshot in DB                        │
│ ✓ docs[] (4 documents)                          │
│   - manuscript (full text)                       │
│   - story_structure                              │
│   - artistic_imagery                             │
│   - moral_lesson                                 │
│ ✓ characters[] (basic info, personality)        │
│ ✓ props[] (name, type)                          │
│ ✓ stages[] (name, location)                     │
│ ✓ spreads[] (manuscript text, textboxes)        │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 2: generate-art-direction                  │
│         (Art Director Agent)                     │
│                                                  │
│ Enhances:                                        │
│ ✓ characters[].visual_description                │
│ ✓ props[].visual_description                     │
│ ✓ stages[].visual_description                    │
│ ✓ spreads[].images[] (detailed image scripts)   │
│                                                  │
│ Optional: Calls generate-visual-description-*   │
│           for optimization                       │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 3: generate-quality-check                  │
│         (Tester Agents)                          │
│                                                  │
│ Creates:                                         │
│ ✓ flags[] (if issues found)                     │
│   - Story consistency issues                     │
│   - Plot problems                                │
│   - Age appropriateness concerns                 │
└──────────────────────┬──────────────────────────┘
                       ↓
        Final Manuscript Ready
                       ↓
┌─────────────────────────────────────────────────┐
│ Optional: Visual Description Optimization       │
│                                                  │
│ Can be called independently:                     │
│ • generate-visual-description-character          │
│ • generate-visual-description-prop               │
│ • generate-visual-description-stage              │
│ • generate-visual-description-spread             │
└─────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Optional: Content Translation                   │
│                                                  │
│ • translate-content                              │
│   - Translates manuscript, textboxes, etc.      │
│   - Preserves @key references          │
│   - Maintains age-appropriate language          │
└─────────────────────────────────────────────────┘
```

## Key Concepts

### Multi-Step Pipeline
The manuscript generation follows a **3-step sequential pipeline**:
1. **Story Analysis** - Creates story structure and content
2. **Art Direction** - Adds visual descriptions
3. **Quality Check** - Validates content quality

Each step can be skipped if needed via options parameters.

### Error Handling Strategy
- **Step 1 fails** → Return error, no story created
- **Step 2 fails** → Return partial result, keep Step 1 data
- **Step 3 fails** → Return partial result, keep Step 1 + Step 2 data

### Visual Description Functions
Functions 5-8 can be used:
- **Independently** - To optimize existing visual descriptions
- **Within Pipeline** - Called by Art Director (Step 2) automatically

### Translation Support
Function 9 provides multi-language support:
- Translates any content type (manuscript, textbox, visual_description, metadata)
- Preserves @key references
- Maintains age-appropriate vocabulary
- Keeps formatting intact

## DB Schema References

All functions interact with these primary tables:
- **stories** - Main story data with metadata
- **snapshots** - Versioned story content (docs[], characters[], props[], stages[], spreads[])
- **flags** - Quality issues identified during testing
- **art_styles** - Visual style guidelines
- **locations** - Geographic/setting references
- **asset_categories** - Entity type classifications

## Common Parameters

### Language
- **Default**: Falls back to `story.original_language` from DB
- **Supported**: vi (Vietnamese), en (English)
- **Rule**: Always use `story.original_language` if not explicitly provided

### Art Style
- **Source**: Always from `story.artstyle_id` → `art_styles.description`
- **Never** pass as parameter
- **Purpose**: Ensures visual consistency across all generations

### Mention Names
- **Format**: lowercase with underscores (e.g., `@miu_cat`, `@forest_1`)
- **Rule**: NEVER translate @key references
- **Usage**: Links entities across characters[], props[], stages[], and spreads[]

## Enum Mappings & Helper Functions

### target_audience Mapping

The `target_audience` field in the `stories` table uses SMALLINT values that map to age ranges.

```typescript
// DB Schema: target_audience SMALLINT
// Values: 0=preschool (2-5), 1=primary (6-8), 2=tweens (9-10)

const TARGET_AUDIENCE_MAP = {
  0: { key: 'preschool', label: 'Preschool (2-5 years)', ageRange: '2-5' },
  1: { key: 'primary', label: 'Primary (6-8 years)', ageRange: '6-8' },
  2: { key: 'tweens', label: 'Tweens (9-10 years)', ageRange: '9-10' }
};

function mapTargetAudienceToText(value: number): string {
  return TARGET_AUDIENCE_MAP[value]?.label || 'Unknown';
}

function mapTextToTargetAudience(text: string): number {
  const entry = Object.entries(TARGET_AUDIENCE_MAP)
    .find(([_, v]) => v.key === text || v.label === text);
  return entry ? parseInt(entry[0]) : 0;
}
```

### genre Mapping

The `genre` field uses SMALLINT values to represent different story genres.

```typescript
// DB Schema: genre SMALLINT
// Values: 1=fantasy, 2=scifi, 3=mystery, 4=romance, 5=horror

const GENRE_MAP = {
  1: { key: 'fantasy', label: 'Fantasy' },
  2: { key: 'scifi', label: 'Science Fiction' },
  3: { key: 'mystery', label: 'Mystery' },
  4: { key: 'romance', label: 'Romance' },
  5: { key: 'horror', label: 'Horror' }
};

function mapGenreToText(value: number): string {
  return GENRE_MAP[value]?.label || 'Unknown';
}

function mapTextToGenre(text: string): number {
  const entry = Object.entries(GENRE_MAP)
    .find(([_, v]) => v.key === text.toLowerCase() || v.label === text);
  return entry ? parseInt(entry[0]) : 1;
}
```

### book_type Mapping

The `book_type` field uses SMALLINT values for different book formats.

```typescript
// DB Schema: book_type SMALLINT
// Values: TBD based on schema

const BOOK_TYPE_MAP = {
  0: { key: 'picture_book', label: 'Picture Book' },
  1: { key: 'illustrated_story', label: 'Illustrated Story' },
  2: { key: 'comic', label: 'Comic' },
  3: { key: 'manga', label: 'Manga' }
};

function mapBookTypeToText(value: number): string {
  return BOOK_TYPE_MAP[value]?.label || 'Unknown';
}

function mapTextToBookType(text: string): number {
  const entry = Object.entries(BOOK_TYPE_MAP)
    .find(([_, v]) => v.key === text.toLowerCase() || v.label === text);
  return entry ? parseInt(entry[0]) : 0;
}
```

### Usage Notes

These mapping functions should be used when:

1. **Sending data to LLM prompts** (enum → text)
   - Convert DB enum values to human-readable text for better LLM understanding
   - Example: `mapTargetAudienceToText(story.target_audience)` before sending to prompt

2. **Parsing LLM responses** (text → enum)
   - Convert LLM text outputs back to enum values for DB storage
   - Example: `mapTextToGenre(llmResponse.genre)` before saving to DB

3. **Displaying to users** (enum → label)
   - Show user-friendly labels in UI instead of numeric values
   - Example: `mapGenreToText(story.genre)` for display in story card

4. **API Documentation**
   - Include these mappings in API documentation for clarity
   - Helps frontend developers understand enum values

**Important:**
- Always use these mapping functions for consistency across the codebase
- Never hardcode enum values directly in prompts or UI
- Update mapping functions if DB schema enum values change

## Development Notes

### TypeScript
- Use interfaces for all input/output types
- Validate input with Zod or similar
- Follow kebab-case for function names

### Error Handling
- Return clear error messages
- Log errors for debugging
- Never expose sensitive data in errors

### Consistency
- When generating visual descriptions, fetch other entities of same type for consistency
- Use art style description in all prompts
- Maintain tone and vocabulary appropriate for target audience
