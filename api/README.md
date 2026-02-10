# API Documentation

Overview of all API endpoints for AI Storybook Canvas.

## Endpoints

| Category | Description | Folder |
|----------|-------------|--------|
| Chat API | Chat-based AI interactions (brainstorming, assistant) | [chat/](./chat/) |
| Doc API | Document generation and manipulation | [doc/](./doc/) |
| Text Generation API | Multi-step manuscript pipeline, visual descriptions, translation | [text-generation/](./text-generation/) |

---

## Prompt Template System

### Naming Convention
| Type | Pattern | Example | DB `type` |
|------|---------|---------|-----------|
| System prompt | `{API}_SYSTEM` | `GENERATE_BRIEF_SYSTEM` | 1 |
| Skill prompt | `{SKILL}_SKILL` | `WRITING_BRIEF_SKILL` | 0 |

### Skill Reference
System prompt chứa marker: `@@Skill sử dụng: SKILL_NAME` hoặc `@@Skill sử dụng: SKILL_1, SKILL_2`

### Build Flow
```
1. Fetch System Prompt ({API}_SYSTEM)
2. Extract Skill Names (parse "@@Skill sử dụng: ...")
3. Fetch Skill Prompt(s)
4. Build Final = Skill(s) + System
5. Render Variables ({%request.prompt%}, ...)
```

### Template Variables
- `{%request.prompt%}` - User input prompt
- `{%request.brief%}` - Selected brief JSON (for draft)
- `{%request.draft%}` - Selected draft JSON (for script)

---

## Enum Mappings

### book_type
```typescript
const BOOK_TYPE_MAP = {
  1: 'Picture Book',        // Sách tranh
  2: 'Illustrated Story',   // Truyện chữ có hình
  3: 'Comic',
  4: 'Manga'
};
```

### dimension
```typescript
const DIMENSION_MAP = {
  1: 'Square 8.5×8.5 (216×216mm)',
  2: 'Portrait 8×10 (203×254mm)',
  3: 'Portrait 6×9 (152×229mm)',
  4: 'Portrait 8.5×11 (216×279mm)',
  5: 'Portrait A4 (210×297mm)',
  6: 'Square 8.25×8.25 (210×210mm)',
  7: 'Square 8×8 (203×203mm)'
};
```

### target_audience
```typescript
const TARGET_AUDIENCE_MAP = {
  1: 'Kindergarten (2-3)',
  2: 'Preschool (4-5)',
  3: 'Primary (6-8)',
  4: 'Middle Grade (9+)'
};
```

### target_core_value
```typescript
const TARGET_CORE_VALUE_MAP = {
  1: 'Dũng cảm',
  2: 'Quan tâm',
  3: 'Trung thực',
  4: 'Kiên trì',
  5: 'Biết ơn',
  6: 'Bản lĩnh',
  7: 'Thấu cảm',
  8: 'Chính trực',
  9: 'Vị tha',
  10: 'Tự thức',
  11: 'Tình bạn',
  12: 'Hợp tác',
  13: 'Chấp nhận sự khác biệt',
  14: 'Tử tế',
  15: 'Tò mò',
  16: 'Tự lập',
  17: 'Xử lý nỗi sợ',
  18: 'Quản lý cảm xúc',
  19: 'Chuyển giao',
  20: 'Bảo vệ môi trường',
  21: 'Trí tưởng tượng'
};
```

### format_genre
```typescript
const FORMAT_GENRE_MAP = {
  1: 'Narrative Picture Books',
  2: 'Lullaby/Bedtime Books',
  3: 'Concept Books',
  4: 'Non-fiction Picture Books',
  5: 'Early Reader',
  6: 'Wordless Picture Books'
};
```

### content_genre
```typescript
const CONTENT_GENRE_MAP = {
  1: 'Mystery',
  2: 'Fantasy',
  3: 'Realistic Fiction',
  4: 'Historical Fiction',
  5: 'Science Fiction',
  6: 'Folklore/Fairy Tales',
  7: 'Humor',
  8: 'Horror/Scary',
  9: 'Biography',
  10: 'Informational',
  11: 'Memoir'
};
```

### writing_style
```typescript
const WRITING_STYLE_MAP = {
  1: 'Narrative',       // Văn xuôi
  2: 'Rhyming',         // Thơ/Vần điệu
  3: 'Humorous Fiction' // Hài hước
};
```

**Usage:** enum → text (prompts), text → enum (parsing), enum → label (UI)
