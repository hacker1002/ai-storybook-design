# CLAUDE.md - AI Storybook Canvas

Hướng dẫn cho Claude AI khi làm việc với project này.

---

## Tổng quan Project

**AI Storybook Canvas** - Ứng dụng tạo sách tranh trẻ em với AI, cho phép:
- Tạo manuscript từ ý tưởng
- Quản lý characters, props, stages
- Generate hình ảnh với AI
- Tạo audio/voice cho truyện
- Export ra PDF, ePub, Video

---

## Cấu trúc Files

```
docs/
├── README.md                    # Homepage (file này hiển thị trên docsify)
├── _sidebar.md                  # Navigation sidebar
├── CLAUDE.md                    # Database Schema & Guidelines
├── index.html                   # Docsify config
│
└── api/text-generation/         # API Text Generation Documentation
```

---

## Database Schema

### Bảng Story (các fields chính)
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tiêu đề truyện |
| `description` | TEXT | Mô tả truyện |
| `owner_id` | UUID | ID người sở hữu |
| `step` | SMALLINT | 1: manuscript, 2: sketch, 3: illustration |
| `type` | SMALLINT | 0: source story (thư viện assets), 1: normal story |
| `original_language` | VARCHAR | Ngôn ngữ gốc (vi, en) |
| `current_version` | UUID | Version hiện tại |
| `current_content` | JSONB | Autosave content (ghi đè mỗi khi user dừng 1p) |
| `cover` | JSONB | Thông tin bìa: `{ thumbnail_url, normal_url }` |
| `book_type` | SMALLINT | Sách tranh/Truyện chữ có hình/Comics/Manga *(required: general)* |
| `dimension` | SMALLINT | Square (20cm x 20cm) *(required: general)* |
| `target_audience` | SMALLINT | preschool (2-5), primary (6-8), (9-10) *(required: general)* |
| `target_core_value` | VARCHAR | Đạo đức, Trí tuệ, Nghị lực *(required: general)* |
| `genre` | SMALLINT | 1: fantasy, 2: scifi, 3: mystery, 4: romance, 5: horror *(required: creative)* |
| `writing_style` | SMALLINT | Phong cách viết *(required: creative)* |
| `era_id` | UUID | FK → `eras` *(required: creative)* |
| `location_id` | UUID | FK → `locations` *(required: creative)* |
| `artstyle_id` | UUID | FK → `art_styles` *(required: creative)* |
| `background_music` | JSONB | `{ title, media_url }` |
| `typography` | JSONB | Default typography settings cho truyện |
| `page_layout` | JSONB | Default layout cho trang mới |
| `remix` | JSONB | Thông tin remix (languages, price, access_resources) |
| `print & export` | JSONB | Thông tin in ấn & export |

**typography structure:**
```json
{
  "size": 16,
  "weight": 400,
  "style": "normal",
  "family": "...",
  "color": "#000000",
  "line_height": 1.5,
  "letter_spacing": 0,
  "decoration": "none",
  "text_align": "left",
  "text_transform": "none"
}
```

**page_layout structure:**
```json
{
  "textbox": {
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 50 },
    "z-index": 1,
    "fill": { "color": "#fff", "opacity": 1 },
    "outline": { "color": "#000", "width": 1, "radius": 0, "type": "solid" }
  },
  "image": {
    "geometry": { "x": 0, "y": 50, "w": 100, "h": 50 },
    "z-index": 0
  }
}
```

**remix structure:**
```json
{
  "languages[]": [{ "name": "English", "code": "en_US" }],
  "price": 0,
  "access_resources[]": ["manuscript", "sketch", "illustration"]
}
```

**print & export structure:**
```json
{
  "filename": "...",
  "cover_material": "...",
  "spread_material": "...",
  "isbn": "...",
  "bleed": 3,
  "color_profile": "CMYK",
  "type": "...",
  "resolution": "300dpi",
  "format": "pdf"
}
```

### Bảng Snapshot (các fields chính)
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `version` | VARCHAR | Version string (năm, tháng, ngày, giờ, phút) |
| `tag` | VARCHAR | Tag đánh dấu |
| `story_id` | UUID | Link đến Story |
| `save_type` | SMALLINT | manual save hay autobackup |
| `docs[]` | JSONB | Các tài liệu |
| `spreads[]` | JSONB | Các spread/trang truyện |
| `characters[]` | JSONB | Danh sách nhân vật |
| `props[]` | JSONB | Danh sách đạo cụ |
| `stages[]` | JSONB | Danh sách bối cảnh |

### Bảng Flags
Bảng lưu các vấn đề tồn đọng trong story.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | ID người tạo flag |
| `story_id` | UUID | Link đến Story |
| `title` | VARCHAR | Tiêu đề vấn đề |
| `content` | TEXT | Mô tả chi tiết vấn đề |
| `type` | SMALLINT | Loại vấn đề |
| `status` | SMALLINT | Trạng thái |

### Bảng Users & Profiles

#### users
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `email` | VARCHAR | Email đăng nhập |
| `password` | INT | Mật khẩu (hashed) |

#### profiles
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK → users |
| `name` | VARCHAR | Tên hiển thị |
| `avatar` | VARCHAR | URL avatar |

### Bảng Collaboration

#### collaborations
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK → users |
| `story_id` | UUID | FK → stories |
| `access_rights` | JSONB | Quyền truy cập chi tiết |

**access_rights structure:**
```json
{
  "new_spread": true,
  "delete_spread": false,
  "languages[]": [{ "name": "Vietnamese", "code": "vi_VN" }],
  "steps": { "manuscript": true, "sketch": true, "illustration": false },
  "spreads[]": [{ "spread_number": 1, "view": true, "edit": true }]
}
```

#### share_links
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK → users |
| `story_id` | UUID | FK → stories |
| `name` | VARCHAR | Tên link |
| `url` | VARCHAR | URL chia sẻ |
| `privacy` | SMALLINT | Mức độ riêng tư |
| `passcode` | VARCHAR | Mã truy cập (nếu có) |
| `access_steps` | JSONB | `{ manuscript, sketch, illustration }` |

### Bảng Resources (Lookup Tables)

#### asset_categories
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên category (Cat, Dog, Pig, Old man, Car...) |
| `type` | TINYINT | human, animal, plant, item |
| `description` | TEXT | Mô tả thêm |

#### art_styles
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên phong cách |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | `[{ title, media_url }]` |

#### locations
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên địa danh |
| `description` | TEXT | Mô tả chi tiết |
| `nation` | VARCHAR | Quốc gia |
| `city` | VARCHAR | Thành phố |
| `type` | SMALLINT | 0: địa điểm có thật, 1: địa điểm hư cấu/ngoài trái đất |
| `image_references[]` | JSONB | `[{ title, media_url }]` |

#### eras
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên thời đại |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | `[{ title, media_url }]` |

#### environments
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên môi trường |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | `[{ title, media_url }]` |

#### page_layouts
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên layout |
| `book_type` | SMALLINT | Loại sách áp dụng |
| `dimension` | SMALLINT | Kích thước |
| `aspect_ratio` | VARCHAR | Tỉ lệ khung hình |
| `content` | JSONB | Nội dung layout (textbox, image geometry) |

### Bảng AI

#### prompt_templates
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | **UNIQUE** key (UPPER_SNAKE_CASE) - dùng để query từ code |
| `content` | TEXT | Nội dung prompt với `{%variable%}` placeholders |
| `model` | VARCHAR | Model AI sử dụng (e.g., `gemini-3-flash-preview`) |
| `created_at` | TIMESTAMPTZ | Thời điểm tạo |
| `updated_at` | TIMESTAMPTZ | Thời điểm cập nhật |

**Xem chi tiết:** [Prompt Template System](#prompt-template-system)

#### prompt_versions
Lưu lịch sử thay đổi của prompt templates. **Auto-populated** bởi trigger khi UPDATE `prompt_templates.content`.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `template_id` | UUID | FK → `prompt_templates` (ON DELETE CASCADE) |
| `name` | VARCHAR | Tên template tại thời điểm thay đổi |
| `content` | TEXT | Nội dung **CŨ** trước khi bị overwrite |
| `model` | VARCHAR | Model tại thời điểm thay đổi |
| `version` | INT | Version number (auto-increment per template) |
| `changed_at` | TIMESTAMPTZ | Thời điểm thay đổi |

**Trigger:** `trigger_save_prompt_version` - fires BEFORE UPDATE on `prompt_templates` khi `content` thay đổi

#### agents
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên agent |
| `key` | VARCHAR | Key định danh agent |
| `description` | TEXT | Mô tả (để orchestrator đọc và gọi) |
| `knowledge` | TEXT | Kiến thức của agent |
| `instruction` | TEXT | Hướng dẫn cho agent |
| `lens` | TEXT | Lens/góc nhìn của agent |
| `model` | VARCHAR | Model AI sử dụng |
| `type` | SMALLINT | 1: story agents, 2: characters, 3: parents, 4: children |

#### ai_requests
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên request |
| `content` | TEXT | Nội dung |
| `model` | VARCHAR | Model sử dụng |
| `timestamp` | TIMESTAMP | Thời điểm tạo |
| `status` | SMALLINT | Trạng thái |
| `params` | JSON | Tham số |

### Chi tiết JSONB structures

#### docs[] structure
```json
{
  "type": "manuscript | story_structure | artistic_imagery | moral_lesson",
  "title": "...",
  "content": "...",
  "url": "..."
}
```

- `type`: 0 = hệ thống đọc/sửa, 1 = user đọc/sửa, 2 = user chỉ đọc
- 4 loại docs:

| Type | Mô tả |
|------|-------|
| `manuscript` | Cốt truyện hoàn chỉnh |
| `story_structure` | Cấu trúc truyện - three-act, conflicts, pacing |
| `artistic_imagery` | Hình tượng nghệ thuật - metaphors, symbols |
| `moral_lesson` | Bài học đạo đức - themes, emotional journey |

#### spreads[] structure
```json
{
  "number": 1,
  "left_page": { "number": 1, "type": "story" },
  "right_page": { "number": 2, "type": "story" },
  "manuscript": "cốt truyện của spread",
  "tiny_sketch_media_url": "...",
  "images[]": [{
    "title": "...",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
    "visual_description": "...",
    "stage": "@stage_mention",
    "actions": "@character doing something",
    "temporal": { "era": "...", "season": "...", "weather": "...", "time_of_day": "...", "duration": "..." },
    "sensory": { "atmosphere": "...", "soundscape": "...", "lighting": "...", "color_palette": "..." },
    "emotional": { "mood": "..." },
    "image_references[]": [{ "title": "...", "media_url": "..." }],
    "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
    "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
    "final_hires_media_url": "...",
    "interaction": { "sound_effect": "...", "target_url": "..." }
  }],
  "videos[]": [{
    "title": "...",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100 },
    "final_hires_image_url": "...",
    "z-index": 1,
    "interaction": { "sound_effect": "...", "target_url": "..." }
  }],
  "textboxes[]": [{
    "order": 1,
    "language[]": [{
      "text": "...",
      "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
      "typography": { "size": 16, "weight": 400, "style": "normal", "family": "...", "color": "#000", "line_height": 1.5, "letter_spacing": 0, "decoration": "none", "text_align": "left", "text_transform": "none" }
    }],
    "fill": { "color": "...", "opacity": 1 },
    "outline": { "color": "...", "width": 1, "radius": 0, "type": "solid" },
    "audio": { "script": "...", "media_url": "...", "speed": 1, "emotion": "...", "voice": "..." },
    "interaction": { "sound_effect": "...", "target_url": "..." }
  }],
  "shapes[]": [{
    "title": "...",
    "type": "rectangle",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
    "fill": { "is_fill": true, "color": "...", "opacity": 1 },
    "outline": { "color": "...", "width": 1, "radius": 0, "type": "solid" },
    "interaction": { "sound_effect": "...", "target_url": "..." }
  }]
}
```

#### characters[] structure
```json
{
  "order": 1,
  "key": "miu_cat",
  "name": "Miu",
  "basic_info": {
    "description": "...",
    "gender": "male",
    "age": "3 tuổi",
    "category_id": "category_1",
    "role": "main character"
  },
  "personality": {
    "core_essence": "...",
    "flaws": "...",
    "emotions": "...",
    "reactions": "...",
    "desires": "...",
    "likes": "...",
    "fears": "...",
    "contradictions": "..."
  },
  "appearance": {
    "height": 30,
    "hair": "...",
    "eyes": "...",
    "face": "...",
    "build": "..."
  },
  "visual_description": "...",
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }],
  "voice": {
    "stability": 0.5,
    "clarity": 0.5,
    "similarity": 0.5,
    "style_exaggeration": 0.5,
    "speaker_boost": false,
    "system_voice": "...",
    "media_url": "..."
  }
}
```

#### props[] structure
```json
{
  "order": 1,
  "name": "Chiếc nơ đỏ",
  "key": "red_bow",
  "category_id": "category_1",
  "visual_description": "...",
  "type": "narrative | anchor",
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }],
  "sounds[]": [{ "title": "...", "media_url": "..." }]
}
```

- `narrative props`: đồ vật dẫn chuyện, tương tác với character
- `anchor props`: đồ vật nằm trong stages, tạo sự nhất quán

#### stages[] structure
```json
{
  "order": 1,
  "name": "Khu rừng 1",
  "key": "forest_1",
  "visual_description": "...",
  "location_id": "uuid",  // FK → locations
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }],
  "sounds[]": [{ "title": "...", "media_url": "..." }]
}
```

---

## Quy tắc khi thiết kế API

### Language Fallback
- Nếu `language` không được truyền vào parameter, **luôn lấy** `story.original_language`
- Áp dụng cho tất cả các generate-visual-description functions và translate-content

### Art Style
- **KHÔNG** truyền `artStyleId` vào parameter
- Art style được lấy từ `story.artstyle_id` → query bảng `art_styles` để lấy description
- Sử dụng art style description trong prompt để đảm bảo consistency

### Negative Prompt (Text Generation)
- Áp dụng cho các function `generate-visual-description-*` (Text Generation)
- **Luôn trả về** `negativePrompt` trong kết quả
- Không cần parameter `includeNegativePrompt`
- **Lưu ý:** Quy tắc này KHÔNG áp dụng cho các function Image Generation

### Consistency
- Khi generate visual description cho một entity, **nên lấy** visual descriptions của các entities cùng loại khác trong story
- Điều này đảm bảo style và tone nhất quán giữa các characters, props, stages

### Mention Name Convention
- Format: lowercase, underscore separated cho `key` prop (@key)
- Ví dụ: `@miu_cat`, `@red_bow`, `@forest_1`
- **KHÔNG** translate @key trong bất kỳ trường hợp nào

### Image Generation Model
- Model AI cho image generation được **fix cứng trong code**
- Không cần parameter `optimizeFor`

### Documentation Standards
- **No code in design docs** except: JSON structures, TypeScript interfaces, mapping constants
- Keep docs focused on specifications, not implementation details
- Định nghĩa TypeScript interfaces rõ ràng cho input/output

### Error Handling
- Trả về error messages rõ ràng
- Log errors để debug
- Không expose sensitive information trong error messages

### Naming Convention
- Edge Functions: `kebab-case` (ví dụ: `generate-visual-description-character`)
- Variables/Functions: `camelCase`
- Types/Interfaces: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`

---

## Prompt Template System

### Variable Syntax
- **Pattern:** `{%variable_name%}`
- **Example:** `{%story_idea%}`, `{%language%}`, `{%categories_text%}`
- JSON trong content **không cần escape** - giữ nguyên `{}` bình thường

### Naming Convention
| Type | Pattern | Example |
|------|---------|---------|
| System prompt | `{AGENT}_SYSTEM` | `STORY_TELLER_SYSTEM` |
| User template | `{FUNCTION}_USER_TEMPLATE` | `STORY_DRAFT_USER_TEMPLATE` |

### Template Registry

| name | Mô tả | Variables |
|------|-------|-----------|
| `STORY_TELLER_SYSTEM` | System prompt - Story Teller agent | *(none)* |
| `STORY_DRAFT_USER_TEMPLATE` | User prompt - tạo story draft | `story_idea`, `dimension`, `target_audience`, `target_core_value`, `genre`, `writing_style`, `era_name`, `era_description`, `location_name`, `location_description`, `art_style_name`, `art_style_description`, `language`, `spreads`, `words_per_spread`, `categories_text`, `locations_text` |
| `ART_DIRECTOR_P1_SYSTEM` | System prompt - Art Director Phase 1 | *(none)* |
| `VISUAL_PLAN_USER_TEMPLATE` | User prompt - tạo visual plan | `title`, `target_audience`, `art_style_description`, `language`, `docs_text`, `characters_text`, `props_text`, `stages_text`, `spreads_text` |
| `WORD_SMITH_SYSTEM` | System prompt - Word Smith | *(none)* |
| `TEXT_REFINEMENT_USER_TEMPLATE` | User prompt - refine text | `title`, `audience`, `language`, `manuscript_content`, `spreads_text` |
| `ART_DIRECTOR_P2_SYSTEM` | System prompt - Art Director Phase 2 | *(none)* |
| `COMPOSITION_USER_TEMPLATE` | User prompt - spread composition | `title`, `target_audience`, `book_type`, `dimension`, `spreads_data` |
| `TESTER_SYSTEM` | System prompt - Quality Check | *(none)* |
| `QUALITY_CHECK_USER_TEMPLATE` | User prompt - quality check | `title`, `target_audience`, `art_style_description`, `target_core_value`, `docs_text`, `characters_json`, `props_json`, `stages_json`, `spreads_json` |
| `VISUAL_DESCRIPTOR_SYSTEM` | System prompt - Visual Descriptor (shared for character, prop, stage, spread) | *(none)* |
| `VISUAL_DESC_CHARACTER_USER_TEMPLATE` | User prompt - character visual description | `name`, `key`, `description`, `gender`, `age`, `category_name`, `category_type`, `category_description`, `role`, `personality_text`, `appearance_text`, `art_style_description`, `title`, `genre`, `target_audience`, `target_core_value`, `existing_visual_descriptions`, `target_length`, `language` |
| `VISUAL_DESC_PROP_USER_TEMPLATE` | User prompt - prop visual description | `name`, `key`, `current_description`, `category_name`, `category_type`, `category_description`, `type`, `sounds_text`, `art_style_description`, `title`, `genre`, `target_audience`, `target_core_value`, `existing_visual_descriptions`, `target_length`, `language` |
| `VISUAL_DESC_STAGE_USER_TEMPLATE` | User prompt - stage visual description | `name`, `key`, `current_description`, `location_name`, `location_description`, `location_nation`, `location_city`, `location_type`, `location_image_references`, `art_style_description`, `title`, `genre`, `target_audience`, `target_core_value`, `era_name`, `era_description`, `story_location_name`, `story_location_description`, `existing_visual_descriptions`, `target_length`, `language` |
| `VISUAL_DESC_SPREAD_USER_TEMPLATE` | User prompt - spread visual description | `spread_number`, `total_spreads`, `left_page_number`, `right_page_number`, `left_page_type`, `right_page_type`, `images_text`, `textboxes_text`, `characters_in_scene`, `stage_info`, `props_in_scene`, `art_style_description`, `title`, `genre`, `target_audience`, `target_core_value`, `previous_spread_description`, `target_length`, `language` |
| `TRANSLATOR_SYSTEM` | System prompt - Translator | *(none)* |
| `TRANSLATOR_USER_TEMPLATE` | User prompt - translate content | `source_language`, `target_language`, `title`, `target_audience`, `genre`, `character_name_mappings`, `content_type`, `content`, `preserve_formatting` |
| `POETRY_GENERATION_USER_TEMPLATE` | User prompt - generate poetry | `title`, `audience`, `language`, `scope`, `poetry_type`, `poetry_type_description`, `source_content`, `custom_instructions` |

### Content Example

```
Analyze the following story idea and create a complete story framework:

## STORY IDEA
{%story_idea%}

## ATTRIBUTES
- Story Types: {%story_types%}
- Target Audience: {%audience%}
- Length: {%length%}
- Art Style Reference: {%art_style_desc%}
- Language: {%language%}

## OUTPUT FORMAT
Return ONLY valid JSON:
{
  "metadata": {
    "title": "Story title",
    "summary": "Brief summary"
  }
}
```

---

## Useful Commands

```bash
# Đọc file Excel với pandas
python3 -c "import pandas as pd; print(pd.read_excel('CoreDB.xlsx', 'Story').to_string())"

# Đọc tất cả sheets
python3 -c "import pandas as pd; xl = pd.ExcelFile('CoreDB.xlsx'); print(xl.sheet_names)"
```

---

## Liên hệ & Tài liệu

- Database Schema: `CoreDB.xlsx`
- Edge Functions Old Spec: `EDGE-FUNCTIONS-SPEC(OLD).md`
