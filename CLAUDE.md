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
cowork/
├── CLAUDE.md                           # File này - hướng dẫn cho Claude
├── CoreDB.xlsx                         # Database schema (Excel)
├── EDGE-FUNCTIONS-SPEC(OLD).md         # Spec tổng quan (cũ) các Edge Functions
├── EDGE-FUNCTIONS-TEXT-GENERATION.md   # Chi tiết Text Generation functions
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
| `art_style_id` | UUID | FK → `art_styles` - Phong cách nghệ thuật |
| `era_id` | UUID | FK → `eras` - Thời đại lịch sử |
| `location_id` | UUID | FK → `locations` - Địa danh cụ thể |
| `genre` | SMALLINT | Thể loại |
| `target_age` | VARCHAR | Nhóm độ tuổi (kids-3-6, kids-7-12, teens) |
| `target_core_value` | VARCHAR | Giá trị cốt lõi (Đạo đức, Trí tuệ, Nghị lực) |
| `original_language` | VARCHAR | Ngôn ngữ gốc (vi, en) |
| `current_version` | UUID | Version hiện tại |
| `current_content` | JSONB | Autosave content |
| `type` | SMALLINT | 0: source story (thư viện assets), 1: normal story |
| `cover` | JSONB | Thông tin bìa sách |
| `storyboard` | JSONB | Storyboard thumbnails |
| `background_music` | JSONB | Nhạc nền |
| `prints` | JSONB | Thông tin in ấn |

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

### Bảng Flags (các fields chính)
Bảng lưu các vấn đề tồn đọng trong story (issues/bugs mà Tester phát hiện).

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `user_id` | UUID | ID người tạo flag (tester hoặc system) |
| `story_id` | UUID | Link đến Story |
| `snapshot_id` | UUID | Link đến Snapshot được kiểm tra |
| `title` | VARCHAR | Tiêu đề vấn đề |
| `content` | TEXT | Mô tả chi tiết vấn đề |
| `suggestion` | TEXT | Gợi ý cách khắc phục |
| `type` | SMALLINT | Loại vấn đề (0: consistency, 1: plot, 2: age_inappropriate, 3: other) |
| `severity` | SMALLINT | Mức độ nghiêm trọng (0: suggestion, 1: minor, 2: major, 3: critical) |
| `target_type` | SMALLINT | Loại đối tượng (0: story, 1: character, 2: prop, 3: stage, 4: spread) |
| `target_id` | VARCHAR | ID đối tượng (mention_name hoặc spread number) |
| `tester` | VARCHAR | Tên tester phát hiện (Story Consistency, Plot, Age Appropriateness) |
| `status` | SMALLINT | Trạng thái (0: open, 1: in_progress, 2: resolved, 3: ignored) |

### Bảng Resources (Lookup Tables)

#### asset_categories
Phân loại các loại assets (nhân vật, đồ vật...).

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `name` | VARCHAR | Tên category (Cat, Dog, Pig, Old man, Car...) |
| `type` | TINYINT | Loại (human, animal, plant, item) |
| `description` | TEXT | Mô tả thêm |

#### art_styles
Các phong cách nghệ thuật cho truyện.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên phong cách |
| `description` | TEXT | Mô tả chi tiết phong cách |
| `image_references[]` | JSONB | Hình ảnh tham khảo `[{ title, media_url }]` |

#### locations
Các địa danh cụ thể (thành phố, địa điểm nổi tiếng...).

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên địa danh |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | Hình ảnh tham khảo `[{ title, media_url }]` |

#### eras
Các thời đại lịch sử.

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên thời đại |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | Hình ảnh tham khảo `[{ title, media_url }]` |

#### environments
Các môi trường/bối cảnh chung (rừng, biển, thành phố...).

| Field | Type | Mô tả |
|-------|------|-------|
| `id` | UUID | Primary key |
| `title` | VARCHAR | Tên môi trường |
| `description` | TEXT | Mô tả chi tiết |
| `image_references[]` | JSONB | Hình ảnh tham khảo `[{ title, media_url }]` |

### Chi tiết JSONB structures

#### spreads[] structure
```json
{
  "number": 1,
  "left_page": { "number": 1, "type": "story" },
  "right_page": { "number": 2, "type": "story" },
  "manuscript": "trang 1: <cốt truyện trang 1>, trang 2: <cốt truyện trang 2>. Nếu spread chỉ có 1 trang thì không cần chia",
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
    "final_hires_image_url": "...",
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
    "language[]": [{ "text": "...", "typography": {...} }],
    "geometry": { "x": 0, "y": 0, "w": 100, "h": 100, "rotation": 0 },
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
  "mention_name": "miu_cat",
  "name": "Miu",
  "basic_info": {
    "description": "...",
    "gender": "male",
    "age": "3 tuổi (tương đương trẻ 5 tuổi)",
    "category_id": "category_1",  // FK → asset_categories
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
    "build": "...",
    "outfit": "..."
  },
  "visual_description": "...",
  "inseparable_item": "...",
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }],
  "voice": {
    "stability": 0.5,
    "clarity": 0.5,
    "similarity": 0.5,
    "style_exaggeration": 0.5,
    "speaker_boost": false,
    "media_url": "...",
    "system_voice": "..."
  },
  "metadata": {
    "role_importance": "main",
    "is_replaceable": false
  }
}
```

#### props[] structure
```json
{
  "name": "Chiếc nơ đỏ",
  "mention_name": "red_bow",
  "category_id": "category_1",  // FK → asset_categories
  "visual_description": "...",
  "type": "narrative",
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }],
  "sounds[]": [{ "title": "...", "media_url": "..." }]
}
```

#### stages[] structure
```json
{
  "name": "Khu rừng 1",
  "mention_name": "forest_1",
  "visual_description": "...",
  "location_id": "location_1",  // FK → `locations`
  "sketch[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "illustration[]": [{ "media_url": "...", "is_active": true, "is_selected": true }],
  "image_references[]": [{ "title": "...", "media_url": "..." }]
}
```

#### docs[] structure
```json
{
  "type": "manuscript | story_structure | artistic_imagery | moral_lesson",
  "title": "...",
  "content": "..." // Markdown content
}
```

- 4 loại docs:

| Type | Mô tả |
|------|-------|
| `manuscript` | Cốt truyện hoàn chỉnh - nội dung truyện đầy đủ từ đầu đến cuối |
| `story_structure` | Cấu trúc truyện - three-act, conflicts, pacing, key plot points |
| `artistic_imagery` | Hình tượng nghệ thuật - metaphors, symbols, visual motifs |
| `moral_lesson` | Bài học đạo đức - themes, emotional journey, age-appropriate messaging |

---

## Quy tắc khi implement Edge Functions

### 1. Language Fallback
- Nếu `language` không được truyền vào parameter, **luôn lấy** `story.original_language`
- Áp dụng cho tất cả các generate-visual-description functions và translate-content

### 2. Art Style
- **KHÔNG** truyền `artStyleId` vào parameter
- Art style được lấy từ `story.art_style_id` → query bảng `art_style` để lấy description
- Sử dụng art style description trong prompt để đảm bảo consistency

### 3. Negative Prompt (Text Generation)
- Áp dụng cho các function `generate-visual-description-*` (Text Generation)
- **Luôn trả về** `negativePrompt` trong kết quả
- Không cần parameter `includeNegativePrompt`
- **Lưu ý:** Quy tắc này KHÔNG áp dụng cho các function Image Generation

### 4. Consistency
- Khi generate visual description cho một entity, **nên lấy** visual descriptions của các entities cùng loại khác trong story
- Điều này đảm bảo style và tone nhất quán giữa các characters, props, stages

### 5. Mention Name Convention
- Format: lowercase, underscore separated
- Ví dụ: `@miu_cat`, `@red_bow`, `@forest_1`
- **KHÔNG** translate @mention_name trong bất kỳ trường hợp nào

### 6. Image Generation Model
- Model AI cho image generation được **fix cứng trong code**
- Không cần parameter `optimizeFor`

---

## Coding Standards

### TypeScript
- Sử dụng TypeScript cho tất cả Edge Functions
- Định nghĩa interfaces rõ ràng cho input/output
- Sử dụng Zod hoặc tương tự để validate input

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
- Edge Functions Spec: `EDGE-FUNCTIONS-SPEC.md`
- Text Generation Details: `EDGE-FUNCTIONS-TEXT-GENERATION.md`
