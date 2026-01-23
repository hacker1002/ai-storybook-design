# API Documentation

Overview of all API endpoints for AI Storybook Canvas.

## Chat API

| # | Endpoint | Agent | Description | File |
|---|----------|-------|-------------|------|
| 0 | chat/story-brainstorming | Story Consultant | Chat với AI để brainstorm ý tưởng truyện. Tự động extract parameters từ conversation. | [chat/00-story-brainstorming.md](./chat/00-story-brainstorming.md) |

## Text Generation API

### Manuscript Generation (Multi-Step Pipeline)

| # | Endpoint | Agent | Description | File |
|---|----------|-------|-------------|------|
| 0 | generate-manuscript | Orchestrator | Điều phối việc tạo manuscript hoàn chỉnh qua 5 bước tuần tự. | [text-generation/00-generate-manuscript.md](./text-generation/00-generate-manuscript.md) |
| 1 | generate-story-draft | Story Teller | Phân tích ý tưởng và tạo khung truyện ban đầu: character/prop/stage bible, phân tích nghệ thuật, phân chia nội dung từng spread. | [text-generation/01-generate-story-draft.md](./text-generation/01-generate-story-draft.md) |
| 2 | generate-spread-visual-plan | Art Director | Tạo visual design cho characters, props, stages và kịch bản hình ảnh cơ bản cho từng spread. | [text-generation/02-generate-spread-visual-plan.md](./text-generation/02-generate-spread-visual-plan.md) |
| 3 | generate-text-refinement | Word Smith | Biên tập và tinh chỉnh nội dung text trong textboxes[] của mỗi spread. | [text-generation/03-generate-text-refinement.md](./text-generation/03-generate-text-refinement.md) |
| 4 | generate-spread-composition | Art Director | Thiết kế bố cục cho mỗi spread: xác định geometry cho images và textboxes. | [text-generation/04-generate-spread-composition.md](./text-generation/04-generate-spread-composition.md) |
| 5 | generate-quality-check | Tester Agents | Kiểm tra chất lượng nội dung qua nhiều góc nhìn. | [text-generation/05-generate-quality-check.md](./text-generation/05-generate-quality-check.md) |

### Visual Description Generation

| # | Endpoint | Agent | Description | File |
|---|----------|-------|-------------|------|
| 6 | generate-visual-description-character | Visual Descriptor | Tối ưu mô tả hình ảnh cho AI sinh ảnh character. | [text-generation/06-generate-visual-description-character.md](./text-generation/06-generate-visual-description-character.md) |
| 7 | generate-visual-description-prop | Visual Descriptor | Tối ưu mô tả hình ảnh cho AI sinh ảnh prop. | [text-generation/07-generate-visual-description-prop.md](./text-generation/07-generate-visual-description-prop.md) |
| 8 | generate-visual-description-stage | Visual Descriptor | Tối ưu mô tả hình ảnh cho AI sinh ảnh stage/background. | [text-generation/08-generate-visual-description-stage.md](./text-generation/08-generate-visual-description-stage.md) |
| 9 | generate-visual-description-spread | Visual Descriptor | Tối ưu mô tả hình ảnh cho AI sinh ảnh spread. | [text-generation/09-generate-visual-description-spread.md](./text-generation/09-generate-visual-description-spread.md) |

### Translation & Poetry

| # | Endpoint | Agent | Description | File |
|---|----------|-------|-------------|------|
| 10 | translate-content | Translator | Dịch nội dung truyện sang ngôn ngữ khác. | [text-generation/10-translate-content.md](./text-generation/10-translate-content.md) |
| 11 | generate-poetry | Word Smith | Tạo thơ từ nội dung văn xuôi. Hỗ trợ thể loại Việt Nam và quốc tế. | [text-generation/11-generate-poetry.md](./text-generation/11-generate-poetry.md) |

---

## Enum Mappings

### target_audience
```typescript
const TARGET_AUDIENCE_MAP = {
  1: { key: 'preschool', label: 'Preschool (2-5 years)' },
  2: { key: 'primary', label: 'Primary (6-8 years)' },
  3: { key: 'tweens', label: 'Tweens (9-10 years)' }
};
```

### genre
```typescript
const GENRE_MAP = {
  1: { key: 'fantasy', label: 'Fantasy' },
  2: { key: 'scifi', label: 'Science Fiction' },
  3: { key: 'mystery', label: 'Mystery' },
  4: { key: 'romance', label: 'Romance' },
  5: { key: 'horror', label: 'Horror' }
};
```

### dimension
```typescript
const DIMENSION_MAP = {
  1: { key: 'square', label: 'Square (20x20cm)' },
  2: { key: 'landscape', label: 'A4 Landscape (29.7x21cm)' },
  3: { key: 'portrait', label: 'A4 Portrait (21x29.7cm)' }
};
```

### writing_style
```typescript
const WRITING_STYLE_MAP = {
  1: { key: 'narrative', label: 'Narrative' },
  2: { key: 'rhyming', label: 'Rhyming' },
  3: { key: 'humorous', label: 'Humorous Fiction' }
};
```

### book_type
```typescript
const BOOK_TYPE_MAP = {
  0: { key: 'picture_book', label: 'Picture Book' },
  1: { key: 'illustrated_story', label: 'Illustrated Story' },
  2: { key: 'comic', label: 'Comic' },
  3: { key: 'manga', label: 'Manga' }
};
```

**Usage:** enum → text (prompts), text → enum (parsing), enum → label (UI)
