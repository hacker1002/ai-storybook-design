# Component Designs

Thiết kế kiến trúc component cho các trang chính của AI Storybook Canvas.

## Quy tắc thiết kế

👉 **Xem chi tiết:** [CLAUDE.md](../CLAUDE.md#quy-tắc-khi-thiết-kế-component)

### Phạm vi
- **Chỉ thiết kế 2 tầng:** Component cha + children trực tiếp
- **KHÔNG thiết kế:** Sub-components của children (sẽ có file riêng)

### Nội dung bắt buộc
- **Mục đích:** Nhiệm vụ component (1-2 câu)
- **Interface:** Props, State, Callbacks (TypeScript)
- **Visualization:** Diagrams (ASCII/Mermaid)

### Tạo component design mới
```bash
cp template-design/component-template.md component/{page-name}/{nn}-{component-name}.md
```

## Danh sách Pages

| Page | Description | Status |
|------|-------------|--------|
| [editor-page](./editor-page/) | Editor page - workspace quản lý book | ✅ Active |

## Cấu trúc thư mục

```
component/
├── README.md
└── {page-name}/
    ├── 00-{root-component}.md    # Root component
    ├── 01-{child-component}.md   # Child components
    ├── ...
    └── screenshots/              # Reference screenshots
```
