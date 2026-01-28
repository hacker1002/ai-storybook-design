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
ai-storybook-design/
├── README.md                    # Homepage (docsify)
├── _sidebar.md                  # Navigation sidebar
├── CLAUDE.md                    # Database Schema & Guidelines
├── index.html                   # Docsify config
│
├── api/
│   ├── chat/                    # Chat API
│   │   └── 00-story-brainstorming.md
│   └── text-generation/         # Text Generation API
│       ├── 00-generate-manuscript.md
│       └── ...
│
├── app/                         # App Features Design
│   └── ai-assistant/
│       └── story-idea-brainstorming.md
│
└── template-design/             # Design Templates
    ├── api-template.md          # Template cho API endpoint
    └── app-template.md          # Template cho App feature
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
| `dimension` | SMALLINT | 1: Square 8.5x8.5 (216x216mm), 2: Portrait 8x10 (203x254mm), 3: Portrait 6x9 (152x229mm), 4: Portrait 8.5x11 (216x279mm), 5: Portrait A4 (210x297mm), 6: Square 8.25x8.25 (210x210mm), 7: Square 8x8 (203x203mm) *(required: general)* |
| `target_audience` | SMALLINT | 1: kindergarten (2-3), 2: preschool (4-5), 3: primary (6-8), 4: middle grade (9+) *(required: general)* |
| `target_core_value` | SMALLINT | 1: Dũng cảm, 2: Quan tâm, 3: Trung thực, 4: Kiên trì, 5: Biết ơn, 6: Bản lĩnh, 7: Thấu cảm, 8: Chính trực, 9: Vị tha, 10: Tự thức, 11: Tình bạn, 12: Hợp tác, 13: Chấp nhận sự khác biệt, 14: Tử tế, 15: Tò mò, 16: Tự lập, 17: Xử lý nỗi sợ, 18: Quản lý cảm xúc, 19: Chuyển giao, 20: Bảo vệ môi trường, 21: Trí tưởng tượng *(required: general)* |
| `format_genre` | SMALLINT | 1: Narrative Picture Books, 2: Lullaby/Bedtime Books, 3: Concept Books, 4: Non-fiction Picture Books, 5: Early Reader, 6: Wordless Picture Books *(required: creative)* |
| `content_genre` | SMALLINT | 1: Mystery, 2: Fantasy, 3: Realistic Fiction, 4: Historical Fiction, 5: Science Fiction, 6: Folklore/Fairy Tales, 7: Humor, 8: Horror/Scary, 9: Biography, 10: Informational, 11: Memoir *(required: creative)* |
| `writing_style` | SMALLINT | 1: Narrative, 2: Rhyming, 3: Humorous Fiction *(required: creative)* |
| `era_id` | UUID | FK → `eras` *(required: creative)* |
| `location_id` | UUID | FK → `locations` *(required: creative)* |
| `artstyle_id` | UUID | FK → `art_styles` *(required: creative)* |
| `background_music` | JSONB | `{ title, media_url }` |
| `typography` | JSONB | Default typography settings cho truyện |
| `template_layout` | JSONB | Default layout cho trang mới `{ spread: id, left_page: id, right_page: id }` - FK → `template_layouts` |
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

**template_layout structure:**
```json
{
  "spread": "uuid",
  "left_page": "uuid",
  "right_page": "uuid"
}
```
*Note: IDs reference `template_layouts` table*

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
| `type` | SMALLINT | 0: consistency, 1: plot, 2: age_inappropriate, 3: other |
| `status` | SMALLINT | 0: open, 1: in_progress, 2: resolved, 3: ignored |

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

#### template_layouts
| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên layout |
| `thumbnail_url` | VARCHAR | URL ảnh thumbnail |
| `book_type` | SMALLINT | Loại sách áp dụng |
| `type` | SMALLINT | 1: spread, 2: left_page, 3: right_page, 4: anyside |
| `slots` | JSONB | Nội dung layout (textboxes[], images[]) |

**slots structure:**
```json
{
  "textboxes[]": [{
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 50 },
    "z-index": 1,
    "fill": { "color": "#fff", "opacity": 1 },
    "outline": { "color": "#000", "width": 1, "radius": 0, "type": "solid" }
  }],
  "images[]": [{
    "geometry": { "x": 0, "y": 50, "w": 100, "h": 50 },
    "z-index": 0,
    "edge_treatment": "spot | vignette | crop"
  }]
}
```

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
| `type` | SMALLINT | 0: orchestrator, 1: creator, 2: artist, 3: parent, 4: children |

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

#### ai_conversations
Chat sessions giữa user và AI assistants.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK → auth.users |
| `story_id` | UUID | FK → stories (nullable) |
| `step` | VARCHAR(50) | `brainstorming`, `story_editing`, `complete`, etc. |
| `created_at` | TIMESTAMPTZ | Thời điểm tạo |
| `updated_at` | TIMESTAMPTZ | Thời điểm cập nhật |
| `deleted_at` | TIMESTAMPTZ | Soft delete |

#### ai_messages
Messages trong conversation.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `conversation_id` | UUID | FK → ai_conversations (CASCADE) |
| `role` | VARCHAR(20) | `user`, `assistant`, `system` |
| `content` | TEXT | Nội dung message |
| `created_at` | TIMESTAMPTZ | Thời điểm tạo |

#### background_jobs
Background jobs cho các task chạy async (generate manuscript, export, etc.)

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `type` | VARCHAR(50) | Job type: `generate_manuscript`, `export_pdf`, etc. |
| `story_id` | UUID | FK → stories (nullable) |
| `user_id` | UUID | FK → auth.users |
| `status` | VARCHAR(20) | `queued`, `running`, `completed`, `failed` |
| `current_step` | SMALLINT | Current step number (default 0) |
| `total_steps` | SMALLINT | Total steps in job |
| `step_details` | JSONB | Detailed status per step |
| `params` | JSONB | Job parameters |
| `result` | JSONB | Job result data |
| `error_message` | TEXT | Error message if failed |
| `created_at` | TIMESTAMPTZ | Thời điểm tạo |
| `updated_at` | TIMESTAMPTZ | Thời điểm cập nhật |

**step_details structure:**
```json
{
  "step1_storyDraft": "pending | running | completed | failed | skipped",
  "step2_visualPlan": "pending",
  "step3_textRefinement": "pending",
  "step4_composition": "pending",
  "step5_qualityCheck": "pending"
}
```

### Chi tiết JSONB structures

#### docs[] structure
```json
{
  "type": 0,
  "title": "manuscript | story_structure | artistic_imagery | moral_lesson",
  "content": "...",
  "url": "..."
}
```

- `type`: 0 = hệ thống đọc/sửa, 1 = user đọc/sửa, 2 = user chỉ đọc
- 4 loại docs theo title:

| Type | Mô tả |
|------|-------|
| `manuscript` | Cốt truyện hoàn chỉnh |
| `story_structure` | Cấu trúc truyện - three-act, conflicts, pacing |
| `artistic_imagery` | Hình tượng nghệ thuật - metaphors, symbols |
| `moral_lesson` | Bài học đạo đức - themes, emotional journey |

#### spreads[] structure
```json
{
  "type": 1,
  "number": 1,
  "left_page": { "number": 1, "type": "story" },
  "right_page": { "number": 2, "type": "story" },
  "manuscript": "cốt truyện của spread",
  "tiny_sketch_media_url": "...",
  "background": { "color": "#fff", "texture": "..." },
  "images[]": [{
    "title": "...",
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100 },
    "setting": "@stage_key/setting_key",
    "visual_description": "...",
    "negative_prompt": "...",
    "image_references[]": [{ "title": "...", "media_url": "..." }],
    "sketches[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "illustrations[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
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

- `type`: 1 = dps (double page spread), 0 = non-dps (2 trang nhỏ)
- `background`: màu nền và texture cho spread
- `setting`: reference tới stage setting (format: `@<stage_key>/<setting_key>`)

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
  "variants[]": [{
    "name": "Default",
    "key": "default",
    "type": 0,
    "appearance": {
      "height": 30,
      "hair": "...",
      "eyes": "...",
      "face": "...",
      "build": "..."
    },
    "visual_description": "...",
    "negative_prompt": "...",
    "sketches[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "illustrations[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "image_references[]": [{ "title": "...", "media_url": "..." }]
  }],
  "voices[]": [{
    "name": "...",
    "key": "...",
    "stability": 0.5,
    "clarity": 0.5,
    "similarity": 0.5,
    "style_exaggeration": 0.5,
    "speaker_boost": false,
    "system_voice": "...",
    "media_url": "..."
  }]
}
```

- `variants[]`: Các biến thể của nhân vật (trang phục, trạng thái khác nhau)
  - `type`: 0 = default, 1 = variant
- `voices[]`: Nhiều giọng nói cho mỗi nhân vật

#### props[] structure
```json
{
  "order": 1,
  "name": "Chiếc nơ đỏ",
  "key": "red_bow",
  "category_id": "category_1",
  "type": "narrative | anchor",
  "states[]": [{
    "name": "Default",
    "key": "default",
    "type": 0,
    "visual_description": "...",
    "negative_prompt": "...",
    "sketches[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "illustrations[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "image_references[]": [{ "title": "...", "media_url": "..." }]
  }],
  "sounds[]": [{ "name": "...", "key": "...", "media_url": "..." }]
}
```

- `type`: `narrative` = đồ vật dẫn chuyện, tương tác với character | `anchor` = đồ vật nằm trong stages, tạo sự nhất quán
- `states[]`: Các trạng thái của đạo cụ (mới, cũ, hỏng, ...)
  - `type`: 0 = default, 1 = state

#### stages[] structure
```json
{
  "order": 1,
  "name": "Khu rừng 1",
  "key": "forest_1",
  "location_id": "uuid",
  "settings[]": [{
    "name": "Default",
    "key": "default",
    "type": 0,
    "visual_description": "...",
    "negative_prompt": "...",
    "temporal": {
      "era": "...",
      "season": "...",
      "weather": "...",
      "time_of_day": "...",
      "duration": "..."
    },
    "sensory": {
      "atmosphere": "...",
      "soundscape": "...",
      "lighting": "...",
      "color_palette": "..."
    },
    "emotional": {
      "mood": "..."
    },
    "sketches[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "illustrations[]": [{ "media_url": "...", "created_time": "...", "is_selected": true }],
    "image_references[]": [{ "title": "...", "media_url": "..." }]
  }],
  "sounds[]": [{ "name": "...", "key": "...", "media_url": "..." }]
}
```

- `location_id`: FK → locations
- `settings[]`: Các cài đặt môi trường (mùa, thời tiết, thời gian trong ngày, ...)
  - `type`: 0 = default, 1 = setting
  - `temporal`: Thông tin thời gian (era, season, weather, time_of_day, duration)
  - `sensory`: Thông tin cảm quan (atmosphere, soundscape, lighting, color_palette)
  - `emotional`: Cảm xúc (mood)

---

## Design Templates

Khi thiết kế API endpoint hoặc App feature mới, **BẮT BUỘC** sử dụng template tương ứng.

### API Endpoint
```bash
cp template-design/api-template.md api/{category}/{nn}-{endpoint-name}.md
```
- `{category}`: thư mục API (chat, text-generation, ...)
- `{nn}`: số thứ tự 2 chữ số (00, 01, 02, ...)
- `{endpoint-name}`: tên endpoint (kebab-case)

### App Feature
```bash
cp template-design/app-template.md app/{feature-group}/{feature-name}.md
```
- `{feature-group}`: nhóm tính năng (ai-assistant, editor, ...)
- `{feature-name}`: tên tính năng (kebab-case)

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

## Quy tắc khi thiết kế App Feature

### Layers (Các tầng)

| Layer | Trách nhiệm |
|-------|-------------|
| **CLIENT** | UI state, validation, user interaction, display data |
| **DB** | Supabase - data storage, RLS policies, realtime subscriptions |
| **API** | Business logic, build prompts, parse responses, background tasks |
| **AI** | AI Provider (Gemini, OpenAI, etc.) - generate content |

### Trách nhiệm chi tiết từng Layer

#### CLIENT
- Validate required fields trước khi gọi API/DB
- Hiển thị form/UI để thu thập thông tin còn thiếu
- Local state management (loading, error, success)
- Display và format data
- **CÓ THỂ** gọi trực tiếp DB (Supabase client) cho CRUD đơn giản
- **KHÔNG** gọi trực tiếp AI Provider

#### DB (Supabase)
- Data storage và CRUD operations
- RLS policies để authorize access
- Realtime subscriptions
- **Có thể gọi từ CLIENT** (qua Supabase client SDK)
- **Có thể gọi từ API** (qua Supabase service role)

#### API
- Business logic phức tạp
- Get extra context từ DB (story settings, related entities)
- Build prompts từ template + context
- Gọi AI Provider
- Parse và validate AI response
- Return formatted response cho client
- **Background tasks:** async jobs, queues, long-running tasks (image gen, export)
- **KHÔNG** chứa UI logic

#### AI (AI Provider)
- Receive prompt từ API
- Generate content (text, image, audio)
- Return raw response cho API
- **Chỉ được gọi từ API layer**

### Quy tắc chung
- Mỗi phase trong feature design **BẮT BUỘC** ghi rõ flow pattern
- **KHÔNG** mix responsibilities giữa các layers
- Client **KHÔNG** gọi trực tiếp AI Provider - luôn qua API
- Client **CÓ THỂ** gọi trực tiếp DB cho simple CRUD (RLS bảo vệ)
- Background tasks **PHẢI** idempotent và có job status tracking
- **No code in design docs** except: JSON structures, TypeScript interfaces, mapping constants
- Keep docs focused on specifications, not implementation details
- Định nghĩa TypeScript interfaces rõ ràng cho input/output

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
| `STORY_CONSULTANT_SYSTEM` | System prompt - Story Consultant (brainstorming) | *(none)* |
| `STORY_CONSULTANT_USER_TEMPLATE` | User prompt - brainstorming chat | `conversation_history`, `current_user_message`, `current_story_idea`, `current_params`, `available_eras`, `available_locations`, `available_art_styles`, `is_finalize` |
| `STORY_TELLER_SYSTEM` | System prompt - Story Teller agent | *(none)* |
| `STORY_DRAFT_USER_TEMPLATE` | User prompt - tạo story draft | `story_idea`, `story_idea_explanation`, `target_audience`, `target_core_value`, `format_genre`, `content_genre`, `writing_style`, `era_name`, `era_description`, `location_name`, `location_description`, `language`, `spreads`, `words_per_spread`, `categories_text`, `locations_text` |
| `ART_DIRECTOR_P1_SYSTEM` | System prompt - Art Director Phase 1 | *(none)* |
| `VISUAL_PLAN_USER_TEMPLATE` | User prompt - tạo visual plan | `title`, `target_audience`, `format_genre`, `content_genre`, `language`, `docs_text`, `characters_text`, `props_text`, `stages_text`, `spreads_text` |
| `WORD_SMITH_SYSTEM` | System prompt - Word Smith | *(none)* |
| `TEXT_REFINEMENT_USER_TEMPLATE` | User prompt - refine text | `title`, `target_audience`, `format_genre`, `writing_style`, `language`, `manuscript_content`, `spreads_text` |
| `ART_DIRECTOR_P2_SYSTEM` | System prompt - Art Director Phase 2 | *(none)* |
| `COMPOSITION_USER_TEMPLATE` | User prompt - spread composition | `title`, `target_audience`, `book_type`, `spreads_data` |
| `TESTER_SYSTEM` | System prompt - Quality Check | *(none)* |
| `QUALITY_CHECK_USER_TEMPLATE` | User prompt - quality check | `title`, `target_audience`, `target_core_value`, `docs_text`, `characters_json`, `props_json`, `stages_json`, `spreads_json` |
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
- Design Templates: `template-design/`
- App Features: `app/`
- API: `api/`
