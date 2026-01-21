# Text Generation Functions

Overview of all text generation functions for AI Storybook Canvas.

## Manuscript Generation (Multi-Step Pipeline)

| # | Function | Agent | Description | File |
|---|----------|-------|-------------|------|
| 0 | generate-manuscript | Orchestrator | Điều phối việc tạo manuscript hoàn chỉnh qua 5 bước tuần tự. | [00-generate-manuscript.md](./00-generate-manuscript.md) |
| 1 | generate-story-draft | Story Teller | Phân tích ý tưởng và tạo khung truyện ban đầu: character/prop/stage bible, phân tích nghệ thuật, phân chia nội dung từng spread. Tạo Story + Snapshot trong DB. | [01-generate-story-draft.md](./01-generate-story-draft.md) |
| 2 | generate-spread-visual-plan | Art Director | Tạo visual design cho characters, props, stages và kịch bản hình ảnh cơ bản cho từng spread. Chưa xử lý composition. | [02-generate-spread-visual-plan.md](./02-generate-spread-visual-plan.md) |
| 3 | generate-text-refinement | Word Smith | Biên tập và tinh chỉnh nội dung text trong textboxes[] của mỗi spread. Tập trung vào chất lượng ngôn ngữ. | [03-generate-text-refinement.md](./03-generate-text-refinement.md) |
| 4 | generate-spread-composition | Art Director | Thiết kế bố cục cho mỗi spread: xác định geometry cho images và textboxes, cập nhật visual_description với text zone notes. | [04-generate-spread-composition.md](./04-generate-spread-composition.md) |
| 5 | generate-quality-check | Tester Agents | Kiểm tra chất lượng nội dung qua nhiều góc nhìn: Story Consistency, Plot, và Age Appropriateness. | [05-generate-quality-check.md](./05-generate-quality-check.md) |

## Visual Description Generation

| # | Function | Description | File |
|---|----------|-------------|------|
| 6 | generate-visual-description-character | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character. | [06-generate-visual-description-character.md](./06-generate-visual-description-character.md) |
| 7 | generate-visual-description-prop | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop. | [07-generate-visual-description-prop.md](./07-generate-visual-description-prop.md) |
| 8 | generate-visual-description-stage | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background. | [08-generate-visual-description-stage.md](./08-generate-visual-description-stage.md) |
| 9 | generate-visual-description-spread | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage). | [09-generate-visual-description-spread.md](./09-generate-visual-description-spread.md) |

## Translation

| # | Function | Description | File |
|---|----------|-------------|------|
| 10 | translate-content | Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả. | [10-translate-content.md](./10-translate-content.md) |

## Pipeline Flow Diagram

```
User Input (Story Idea + Attributes)
        ↓
┌─────────────────────────────────────────────────┐
│ 0. generate-manuscript (ORCHESTRATOR)           │
│    - Validates input                             │
│    - Manages 5-step pipeline                     │
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
│   - manuscript, story_structure                  │
│   - artistic_imagery, moral_lesson               │
│ ✓ characters[] (basic info, personality)        │
│ ✓ props[] (name, type)                          │
│ ✓ stages[] (name, location)                     │
│ ✓ spreads[] (manuscript text, textboxes)        │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 2: generate-spread-visual-plan             │
│         (Art Director)                           │
│                                                  │
│ Updates:                                         │
│ ✓ characters[].visual_description                │
│ ✓ props[].visual_description                     │
│ ✓ stages[].visual_description                    │
│ ✓ spreads[].images[] (basic, NO geometry yet)   │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 3: generate-text-refinement                │
│         (Word Smith Agent)                       │
│                                                  │
│ Updates:                                         │
│ ✓ spreads[].textboxes[].language[].text         │
│   - Refined for reading level                    │
│   - Improved rhythm and flow                     │
│   - Economy of words                             │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 4: generate-spread-composition             │
│         (Art Director)                           │
│                                                  │
│ Updates:                                         │
│ ✓ spreads[].images[].geometry                   │
│ ✓ spreads[].images[].visual_description         │
│   + text zone notes appended                     │
│ ✓ spreads[].images[].text_zone                  │
│ ✓ spreads[].images[].composition_notes          │
│ ✓ spreads[].textboxes[].language[].geometry     │
└──────────────────────┬──────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────┐
│ Step 5: generate-quality-check                  │
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
│   - Preserves @key references                   │
│   - Maintains age-appropriate language          │
└─────────────────────────────────────────────────┘
```

## Key Concepts

### Multi-Step Pipeline
The manuscript generation follows a **5-step sequential pipeline**:
1. **Story Draft** - Creates story structure and content (Story Teller)
2. **Spread Visual Plan** - Adds visual descriptions for entities + basic image plans (Art Director)
3. **Text Refinement** - Polishes text content in textboxes (Word Smith)
4. **Spread Composition** - Designs layout with geometry + text zones (Art Director)
5. **Quality Check** - Validates content quality (Tester Agents)

Steps 2, 3, 4 (Art Direction) can be skipped together via `skipArtDirection` option.

### Error Handling Strategy
- **Step 1 fails** → Return error, no story created
- **Step 2 fails** → Return partial result, keep Step 1 data
- **Step 3 fails** → Return partial result, keep Step 1 + 2 data
- **Step 4 fails** → Return partial result, keep Step 1 + 2 + 3 data
- **Step 5 fails** → Return partial result, keep all previous data

### Visual Description Functions
Functions 6-9 can be used:
- **Independently** - To optimize existing visual descriptions
- **Within Pipeline** - Called by Art Director (Step 2) automatically

### Translation Support
Function 10 provides multi-language support:
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
