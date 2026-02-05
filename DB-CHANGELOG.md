# Database Schema Changelog

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
