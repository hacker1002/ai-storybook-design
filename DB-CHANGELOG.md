# Database Schema Changelog

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
