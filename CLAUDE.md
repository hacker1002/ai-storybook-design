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
│   └── editor-page/             # Editor page components
│       ├── 00-editor-page.md
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
```bash
cp template-design/component-template.md component/{page-name}/{nn}-{component-name}.md
```
- `{page-name}`: tên page (editor-page, dashboard, ...)
- `{nn}`: số thứ tự 2 chữ số (00, 01, 02, ...)
- `{component-name}`: tên component (kebab-case)

---

## Quy tắc khi thiết kế Component

### Phạm vi thiết kế
- **Chỉ thiết kế 2 tầng:** Component cha + các component con trực tiếp
- **KHÔNG thiết kế:** Sub-components của children (sẽ có file riêng)
- **Nếu đã có thiết kế component cha:** Tuân thủ theo thiết kế component cha khi thiết kế

### Nội dung bắt buộc

| Mục | Mô tả |
|-----|-------|
| **Mục đích** | Nhiệm vụ của component (1-2 câu) |
| **Interface** | Props, State, Callbacks (TypeScript) |
| **Visualization** | Diagrams (ASCII hoặc Mermaid) |

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
| UI state chung | Parent component | currentStep, activeWorkspace |
| UI state riêng | Local component | isMenuOpen, selectedItem |

### Naming Conventions

| Loại | Convention | Ví dụ |
|------|------------|-------|
| Data props | noun | `book`, `items` |
| Callback props | on + Verb | `onSave`, `onChange` |
| Boolean props | is/has + Adj | `isOpen`, `hasChanges` |
| Handler functions | handle + Verb | `handleSave` |

**Cố gắng** đặt tên component 1 cách rõ ràng, dễ hiểu, tránh nhập nhằng

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
