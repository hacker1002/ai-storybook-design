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

| # | Function | Agent | Description | File |
|---|----------|-------|-------------|------|
| 6 | generate-visual-description-character | Visual Descriptor | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh character. | [06-generate-visual-description-character.md](./06-generate-visual-description-character.md) |
| 7 | generate-visual-description-prop | Visual Descriptor | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh prop. | [07-generate-visual-description-prop.md](./07-generate-visual-description-prop.md) |
| 8 | generate-visual-description-stage | Visual Descriptor | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh stage/background. | [08-generate-visual-description-stage.md](./08-generate-visual-description-stage.md) |
| 9 | generate-visual-description-spread | Visual Descriptor | Tối ưu mô tả hình ảnh chi tiết cho AI sinh ảnh spread (scene composition với characters, props, và stage). | [09-generate-visual-description-spread.md](./09-generate-visual-description-spread.md) |

## Translation

| # | Function | Agent | Description | File |
|---|----------|-------|-------------|------|
| 10 | generate-translation | Translator | Dịch nội dung truyện sang ngôn ngữ khác, giữ nguyên formatting và context phù hợp với đối tượng độc giả. | [10-translate-content.md](./10-translate-content.md) |

## Poetry Generation

| # | Function | Agent | Description | File |
|---|----------|-------|-------------|------|
| 11 | generate-poetry | Word Smith | Tạo thơ từ nội dung văn xuôi cho page, spread, hoặc toàn bộ truyện. Hỗ trợ nhiều thể loại thơ Việt Nam (lục bát, tứ tuyệt) và quốc tế (haiku, quatrain, limerick). | [11-generate-poetry.md](./11-generate-poetry.md) |

## Enum Mappings

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
```

### book_type Mapping

The `book_type` field uses SMALLINT values for different book formats.

```typescript
// DB Schema: book_type SMALLINT

const BOOK_TYPE_MAP = {
  0: { key: 'picture_book', label: 'Picture Book' },
  1: { key: 'illustrated_story', label: 'Illustrated Story' },
  2: { key: 'comic', label: 'Comic' },
  3: { key: 'manga', label: 'Manga' }
};
```

### Usage Notes

Use these mappings when:
- Sending data to LLM prompts (enum → text)
- Parsing LLM responses (text → enum)
- Displaying to users (enum → label)

**Important:** Never hardcode enum values directly in prompts or UI.
