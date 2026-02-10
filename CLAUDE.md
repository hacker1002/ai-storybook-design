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
├── CLAUDE.md                    # AI Guidelines
├── DATABASE-SCHEMA.md           # Database Schema
├── DB-CHANGELOG.md              # Database Schema Change Log
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
├── component/                   # Component Designs
│   ├── stores/                  # Zustand store designs
│   │   └── snapshot-store.md
│   └── editor-page/             # Editor page components
│       ├── README.md            # Page root (was 00-editor-page.md)
│       ├── 01-editor-header.md
│       ├── 02-icon-rail.md
│       ├── doc-creative-space/      # Creative space folder
│       │   ├── README.md            # Space root (was 00-doc-creative-space.md)
│       │   └── 01-doc-sidebar.md
│       ├── dummy-creative-space/
│       ├── sketch-creative-space/
│       ├── shared/                  # Shared components
│       │   └── manuscript-spread-view/
│       └── screenshots/
│
└── template-design/             # Design Templates
    ├── api-template.md          # Template cho API endpoint
    ├── app-template.md          # Template cho App feature
    └── component-template.md    # Template cho Component design
```

---

## Database Schema

👉 **Xem chi tiết:** [DATABASE-SCHEMA.md](./DATABASE-SCHEMA.md)

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

### Component Design

- Trong trường hợp component đơn giản (1 cấp), không chia làm các component con sâu hơn:

```bash
cp template-design/component-template.md component/{page-name}/{component-name}.md
```

- Nếu component phức tạp và phải chia làm nhiều cấp component con, tạo 1 thư mục cho component đó:

```bash
cp -r template-design/component-template.md component/{page-name}/{component-name}/README.md
cp template-design/component-template.md component/{page-name}/{component-name}/{01-child-xx}.md
cp template-design/component-template.md component/{page-name}/{component-name}/{02-child-yy}.md
...
```

#### Folder-Based Hierarchy Convention

| Type | Location | Format |
|------|----------|--------|
| Page root | `editor-page/` | `README.md` |
| Direct children of page | `editor-page/` | `01-editor-header.md`, `02-icon-rail.md` |
| Creative space | `editor-page/{component-name}/` | Folder name = component name |
| Creative space root | `{space path}/` | `README.md` |
| Creative space direct children | `{space path}/` | `01-`, `02-`, ... |
| Creative space nested children | `{space path}/{component}/` | `README.md`, `01-` `02-`, ... |
| Shared components | `editor-page/shared/{component}/` | Reusable across spaces |

**Quy tắc:**
- **Folder name** = component name (kebab-case), thay thế cho parent ID
- **Root file** trong folder luôn là `README.md`
- **Direct Children** trong folder: `01-`, `02-`, `03-`, ... (reset numbering per folder)
- **Nested children root**: `{parent-path}/{child-folder}/README.md`
- **Nested grant children**: `{parent-path}/{child-folder}/{xx-grant-child}.md`
- **Shared folder**: `shared/{component}/` cho components dùng chung

**Ví dụ cấu trúc:**
```
editor-page/
├── README.md                       # Page root
├── 01-editor-header.md             # Direct child of page
├── 02-icon-rail.md                 # Direct child of page
│
├── doc-creative-space/             # Creative space folder
│   ├── README.md                   # Space root
│   ├── 01-doc-sidebar.md           # Child of folder
│   └── 02-manuscript-doc-editor.md
│
├── shared/                         # Shared components
│   └── manuscript-spread-view/     # Reusable component group
│       ├── README.md
│       ├── 01-spread-view-header.md
│       ├── 02-spread-editor-panel.md
│       ├── 02-01-editable-image.md # Nested child of 02
│       └── 03-spread-thumbnail-list.md
│
└── screenshots/
```

**Lợi ích:**
- Folder name tự mô tả, dễ navigate
- README.md luôn hiển thị cho root component trong thư mục
- Numbering reset per folder, tránh số quá dài
- Shared folder tái sử dụng components
- Sort tự nhiên theo thứ tự trong mỗi folder

---

## Quy tắc chung
- **(IMPORTANT) No code in design docs** except: JSON structures, TypeScript interfaces, mapping constants, Store Integration section
- **(IMPORTANT)** không viết code css style vào doc thiết kế
- **Cố gắng** đặt tên component, api, feature 1 cách rõ ràng, dễ hiểu, tránh nhập nhằng
- Keep docs focused on specifications, not implementation details
- Định nghĩa TypeScript interfaces rõ ràng cho input/output

---

## Quy tắc khi thiết kế Component

### Phạm vi thiết kế
- **Chỉ thiết kế 2 tầng:** Component cha + các component con trực tiếp
- **KHÔNG thiết kế:** Sub-components của children (sẽ có file riêng)
- **Nếu đã có thiết kế component cha:** Tuân thủ theo thiết kế component cha khi thiết kế
- **BẮT BUỘC check các file store design** trước khi thiết kế component

### Nội dung bắt buộc

| Mục | Mô tả |
|-----|-------|
| **Mục đích** | Nhiệm vụ của component (1-2 câu) |
| **Interface** | Props, State, Callbacks, Store State Selectors & Actions (TypeScript) |
| **Visualization** | Diagrams (ASCII hoặc Mermaid), **Lưu ý:** nếu có screenshot mẫu thì thêm đường dẫn ảnh mẫu vào |
| **Child Components** | tập trung vào parent-child interaction |

### Được phép / Không được phép

| ✅ Được phép | ❌ Không được phép |
|-------------|-------------------|
| Pseudo code | Code implementation |
| TypeScript interfaces | JSX, CSS |
| JSON structures | Business logic chi tiết |
| Mapping constants | Sub-components details |
| Helper function signatures | Error handling details |

### State Location

| Loại State | Vị trí | Ví dụ |
|------------|--------|-------|
| Data state | Parent cao nhất | book, snapshot |
| UI state chung | Parent component | currentStep, activeCreativeSpace |
| UI state riêng | Local component | isMenuOpen, selectedItem |

### Naming Conventions

| Loại | Convention | Ví dụ |
|------|------------|-------|
| Data props | noun | `book`, `items` |
| Callback props | on + Verb | `onSave`, `onChange` |
| Boolean props | is/has + Adj | `isOpen`, `hasChanges` |
| Handler functions | handle + Verb | `handleSave` |

### Annotations

| Ký hiệu | Ý nghĩa |
|---------|---------|
| ⚡ | Bị ảnh hưởng bởi logic đặc biệt |
| ✅ | Có / Enabled |
| ❌ | Không / Disabled |

### Anti-patterns
- ❌ Prop drilling quá sâu (> 3 tầng)
- ❌ Component làm quá nhiều việc (God component)
- ❌ Intermediate component chỉ pass props mà không có logic riêng

### Quy tắc khi thiết kế Component Stores:

- **Zustand Stores:** Thiết kế store toàn cục cho app
- **Slice pattern**: Mỗi domain có slice riêng
- **Immer middleware**: Mutation syntax cho immutable updates
- **ID-based selectors**: Tối ưu re-render với stable ID
- **Actions-only hook**: `useXxxActions()` không trigger re-render

---

## Quy tắc khi thiết kế API

### Language Fallback
- Nếu `language` không được truyền vào parameter, **luôn lấy** `book.original_language`
- Áp dụng cho tất cả các generate-visual-description functions và translate-content

### Art Style
- **KHÔNG** truyền `artStyleId` vào parameter
- Art style được lấy từ `book.artstyle_id` → query bảng `art_styles` để lấy description
- Sử dụng art style description trong prompt để đảm bảo consistency

### Negative Prompt (Text Generation)
- Áp dụng cho các function `generate-visual-description-*` (Text Generation)
- **Luôn trả về** `negativePrompt` trong kết quả
- Không cần parameter `includeNegativePrompt`
- **Lưu ý:** Quy tắc này KHÔNG áp dụng cho các function Image Generation

### Consistency
- Khi generate visual description cho một entity, **nên lấy** visual descriptions của các entities cùng loại khác trong book
- Điều này đảm bảo style và tone nhất quán giữa các characters, props, stages

### Mention Name Convention
- Format: lowercase, underscore separated cho `key` prop (@key)
- Ví dụ: `@miu_cat`, `@red_bow`, `@forest_1`
- **KHÔNG** translate @key trong bất kỳ trường hợp nào

### Image Generation Model
- Model AI cho image generation được **fix cứng trong code**
- Không cần parameter `optimizeFor`

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
- Get extra context từ DB (book settings, related entities)
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

---

## Prompt Template System

### Variable Syntax
- **Pattern:** `{%variable_name%}` hoặc `{%request.variable_name%}`
- **Example:** `{%request.prompt%}`, `{%request.brief%}`, `{%request.draft%}`
- JSON trong content **không cần escape** - giữ nguyên `{}` bình thường

### Naming Convention

| Type | Pattern | Example | DB `type` |
|------|---------|---------|-----------|
| System prompt | `{API/FUNCTION}_SYSTEM` | `GENERATE_BRIEF_SYSTEM` | 1 |
| Skill prompt | `{SKILL_NAME}_SKILL` | `WRITING_BRIEF_SKILL` | 0 |

**Lưu ý:** Mỗi API endpoint chỉ có **1 system prompt**.

### Skill Reference

System prompt chứa marker để reference skill(s):
```
@@Skill sử dụng: WRITING_BRIEF_SKILL
```

Có thể reference nhiều skills (comma-separated):
```
@@Skill sử dụng: WRITING_SCRIPT_SKILL, PACING_SKILL
```

---

## Useful Commands

```bash
# Đọc file Excel với pandas
python3 -c "import pandas as pd; print(pd.read_excel('CoreDB.xlsx', 'Book').to_string())"

# Đọc tất cả sheets
python3 -c "import pandas as pd; xl = pd.ExcelFile('CoreDB.xlsx'); print(xl.sheet_names)"
```

---

## Liên hệ & Tài liệu

- Database Design excel: `CoreDB.xlsx`
- Database Schemas: `DATABASE-SCHEMA.md`
- Edge Functions Old Spec: `EDGE-FUNCTIONS-SPEC(OLD).md`
- Design Templates: `template-design/`
- App Features: `app/`
- API: `api/`
