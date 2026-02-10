# Doc API

API endpoints cho DocCreativeSpace - hỗ trợ flow sáng tạo: Brief → Draft → Script.

## Endpoints

| # | Endpoint | Description | Input | Output |
|---|----------|-------------|-------|--------|
| 01 | [generate-brief](./01-generate-brief.md) | Tạo 3 brief từ ý tưởng | prompt + llmContext | 3 briefs |
| 02 | [generate-draft](./02-generate-draft.md) | Triển khai brief thành draft | brief + prompt + llmContext | 2 drafts |
| 03 | [generate-script](./03-generate-script.md) | Chuyển draft thành script phân trang | draft + prompt + llmContext | 1 script |

## Flow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  generate-brief │ ──▶ │  generate-draft │ ──▶ │ generate-script │
│    (3 briefs)   │     │    (2 drafts)   │     │   (1 script)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
   User chọn 1            User chọn 1            Lưu vào
   brief → draft          draft → script         snapshot.docs[]
```

## Shared Parameters

```typescript
interface LLMContext {
  // Required - Story attributes
  targetAudience: 1 | 2 | 3 | 4;       // Nhóm tuổi
  targetCoreValue: number;              // Giá trị cốt lõi (1-21)
  formatGenre: 1 | 2 | 3 | 4 | 5 | 6;  // Format sách
  contentGenre: number;                 // Thể loại (1-11)

  // Optional - Additional context
  era?: { id: string; name: string; description: string };
  location?: { id: string; name: string; description: string };
}
```

## Design Principles

1. **Stateless APIs** - Không lưu DB, client tự quản lý flow
2. **Parsed JSON Response** - API validates structure trước khi trả về
3. **Progressive Refinement** - Mỗi bước cho user chọn lựa và refine
4. **Rich Context** - Skill prompts chứa domain knowledge sâu
5. **llmContext consolidation** - Tất cả context cho LLM gom vào 1 object
