# Database Schema Changelog

---

## [2026-02-09 10:55] Snapshot Sketch Refactor & Asset Cleanup

**Important:** Editor pipeline steps đổi từ `Idea > Sketch > Illustration > Retouch` -> `Manuscript > Illustration > Retouch`

### Snapshot Table
- **manuscript**: REMOVED - tách thành 3 fields riêng
- **docs[]**: NEW JSONB - manuscript documents (brief, draft, script)
- **dummies[]**: NEW JSONB - dummy spreads với text và art notes
- **sketch**: NEW JSONB - centralized sketch data
  ```json
  {
    "dummy_id": "uuid",
    "character_sheets[]": ["img1", "img2"],
    "prop_sheets[]": ["img1"]
  }
  ```

### Book Table
- **step (pipeline)**: manuscript, illustration, retouch
- **remix.access_resources[]**: string array → object array (`["manuscript", "sketch", "illustration"] -> [{ "type": "character | prop", "name": "...", "key": "asset_key" }]`)

### JSONB spreads[]
- **images[].id**: NEW UUID
- **objects[].original_image_id**: NEW - link to source image
- **objects[].media_url**: replaces `image_url`
- **objects[].media_type**: NEW - "image | video" (replaces `video_url`)

### JSONB characters[].variants[], props[].states[], stages[].settings[]
- **sketches[]**: REMOVED - moved to snapshot.sketch

**Rationale**: Centralize sketch management in `snapshot.sketch` instead of scattered across entities. Simplify asset structure.

Migration: `../supabase/migrations/20260209000001_snapshot_sketch_refactor.sql`

---

## [2026-02-07 10:20] Add spread ID for optimized rendering

- **JSONB manuscript.dummies[].spreads[]**: thêm `id` (UUID)
- **JSONB spreads[]**: thêm `id` (UUID)
- **Lý do**: Enable ID-based subscription trong Zustand store, tránh re-render toàn bộ list khi edit 1 spread
- **Generation**: Client-side UUID (e.g., `crypto.randomUUID()`) khi tạo spread mới

Migration: N/A (JSONB field, backward compatible - existing spreads sẽ được assign ID khi load/save)

---

## [2026-02-06 10:55] Rename flags to issues & Update animations

- **flags**: đổi tên thành `issues`
- **issues.type**: 1-15 (xem Issue Types reference table trong DATABASE-SCHEMA.md)
  - 1-4: Visual, 5-7: Logic & Thời gian, 8-10: Văn bản - Hình ảnh, 11-13: Nhân vật, 14-15: Kỹ thuật
- **issues.status**: 0=not_fixed, 1=fixed, 2=skipped
- **JSONB spreads[].animations[].effect**:
  - Di chuyển `delay`, `duration`, `loop` vào trong `effect`
  - Thêm: `amount`, `direction`
  - `effect.type`: 1-27 (xem Animation Effect Types reference table trong DATABASE-SCHEMA.md)

Migration: `../supabase/migrations/20260206000001_rename_flags_to_issues.sql`

---

## [2026-02-05 18:32] Rename manuscripts to manuscript

- **snapshots**: `manuscripts` → `manuscript` (column rename)
- **JSONB structure**: array `[]` → object `{}`
- **Data migration**: first array element extracted, empty arrays → empty object

Migration: `../supabase/migrations/20260205000002_rename_manuscripts_to_manuscript.sql`

---

## [2026-02-05 10:15] Snapshot Structure Refactor

- **snapshots**: `docs[]` → `manuscripts[]`
- **JSONB manuscripts[]**: NEW structure
  - `docs[]`: type (brief/draft/script), content
  - `dummies[]`: type (prose/poetry), spreads[] với layout info
- **JSONB spreads[]**:
  - Thêm: `layout` (FK → template_layouts)
  - Thêm: `left_page.layout`, `right_page.layout`
  - Thêm: `images[].art_note`
  - Đổi: `textboxes[].language[]` array → `textboxes[].[language_key]` object
- **JSONB spreads[].animations[]**:
  - Đổi: `action_type` → `effect.type`
  - Thêm: `effect.geometry` (x, y, w, h) cho moving animation
  - Thêm: `effect.type` = "moving"
  - Thêm: `loop` (số lần lặp)
- **Backward compat**: giữ `negative_prompt` trong images[], variants[], states[], settings[]

Migration: `../supabase/migrations/20260205000001_snapshot_structure_refactor.sql`

---

## [2026-02-03 14:50] Rename Story → Book

- **stories**: đổi tên thành `books`
- **snapshots**: `story_id` → `book_id`
- **flags**: `story_id` → `book_id`
- **collaborations**: `story_id` → `book_id`
- **share_links**: `story_id` → `book_id`
- **ai_conversations**: `story_id` → `book_id`
  - step: thêm `generating`, đổi `story_editing` → `book_editing`
- **background_jobs**: `story_id` → `book_id`
- **template_layouts**: type comment đổi thành "1: double page, 2: single page"
- **prompt_templates**: thêm `type` SMALLINT (1=storytelling, 2=pacing, 3=art_direction, 4=wordsmith, 5=draw)
- **JSONB spreads[]**:
  - Xoá: videos[], shapes[], interaction
  - Thêm: objects[], animations[], textboxes[].id, textboxes[].title
- **JSONB characters[]/props[]**: thêm crop_sheets[]
- **JSONB stages[].temporal**: xoá duration

Migration: `../supabase/migrations/20260203000001_rename_story_to_book.sql`

---

## [2026-01-29 14:39] Background Jobs

- **background_jobs**: tạo mới - async task processing

Migration: `../supabase/migrations/20260129000001_create_background_jobs_table.sql`

---

## [2026-01-28 11:58] Sync Schema

- **stories**: thêm `format_genre`, `content_genre`; xoá `genre`; đổi `page_layout` → `template_layout`
- **page_layouts**: đổi tên thành `template_layouts`; thêm `thumbnail_url`; đổi `content` → `slots`
- **prompt_templates**: thêm UNIQUE constraint cho `name`

Migration: `../supabase/migrations/20260128000001_sync_schema_with_design.sql`

---

## [2026-01-23 18:51] AI Chat Tables

- **ai_conversations**: tạo mới - chat sessions
- **ai_messages**: tạo mới - messages

Migration: `../supabase/migrations/20260123000001_create_ai_chat_tables.sql`
