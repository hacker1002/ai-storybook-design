# Component Designs

Thiết kế kiến trúc component cho các trang chính của AI Storybook Canvas.

## Quy tắc thiết kế

👉 **Xem chi tiết:** [CLAUDE.md](CLAUDE.md#quy-tắc-khi-thiết-kế-component)

### Phạm vi
- **Chỉ thiết kế 2 tầng:** Component cha + children trực tiếp
- **KHÔNG thiết kế:** Sub-components của children (sẽ có file riêng)

### Nội dung bắt buộc
- **Mục đích:** Nhiệm vụ component (1-2 câu)
- **Interface:** Props, State, Callbacks (TypeScript)
- **Visualization:** Diagrams (ASCII/Mermaid)

### Tạo component design mới

**Simple component (1 file):**
```bash
cp template-design/component-template.md component/{page-name}/{nn}-{component-name}.md
```

**Complex component (folder with children):**
```bash
mkdir -p component/{page-name}/{component-name}
cp template-design/component-template.md component/{page-name}/{component-name}/README.md
cp template-design/component-template.md component/{page-name}/{component-name}/01-{child}.md
```

## Danh sách Pages

| Page | Description | Status |
|------|-------------|--------|
| [editor-page](./editor-page/) | Editor page - creativeSpace quản lý book | ✅ Active |

## Stores

| Store | Description | Status |
|-------|-------------|--------|
| [book-store](./stores/book-store.md) | Zustand store cho book metadata và settings | ✅ Designed |
| [snapshot-store](./stores/snapshot-store.md) | Zustand store cho snapshot data (manuscript, spreads, characters, props, stages) | ✅ Designed |
| [editor-settings-store](./stores/editor-settings-store.md) | Zustand store cho editor UI state (currentLanguage, viewMode, zoom) | ✅ Designed |

## Cấu trúc thư mục

```
component/
├── README.md
├── stores/                       # Zustand store designs
│   ├── README.md
│   └── snapshot-store.md
└── {page-name}/
    ├── README.md                 # Page root component
    ├── 01-{child-component}.md   # Direct children
    ├── 02-{child-component}.md
    ├── {complex-component}/      # Complex component folder
    │   ├── README.md             # Component root
    │   ├── 01-{child}.md
    │   └── 02-{child}.md
    ├── shared/                   # Shared/reusable components
    │   └── {component}/
    │       └── README.md
    └── screenshots/              # Reference screenshots
```
