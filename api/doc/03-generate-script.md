# generate-script

## Description
**Writing Script Agent** - Chuyển draft thành manuscript script hoàn chỉnh: phân trang theo chuẩn 32 trang, text trau chuốt cho mỗi trang/spread, ghi chú minh họa cho illustrator, và page turn hooks.

**Entry point:** POST `/api/doc/generate-script`

## DB Schema Dependencies

### Tables Referenced
- `eras`: Truy vấn era description (optional context)
- `locations`: Truy vấn location description (optional context)

## Parameters

```typescript
interface GenerateScriptRequest {
  // Required - Selected draft from previous step
  draft: DraftItem;

  // Required - User refinement
  prompt: string;                           // Ghi chú bổ sung từ tác giả

  // Required - LLM Context
  llmContext: {
    // Story attributes (required)
    targetAudience: 1 | 2 | 3 | 4;
    targetCoreValue: number;
    formatGenre: 1 | 2 | 3 | 4 | 5 | 6;
    contentGenre: number;

    // Additional context (optional)
    era?: { id: string; name: string; description: string };
    location?: { id: string; name: string; description: string };
  };
}

// From generate-draft result
interface DraftItem {
  title: string;
  voice_style: string;
  who: string;
  what: string;
  when: string;
  where: string;
  why: string;
  how: string;
  characters: CharacterProfile[];
  full_narrative: string;
}
```

## Result

```typescript
interface GenerateScriptResult {
  success: boolean;
  data: ScriptItem;                         // Single script object
  meta?: {
    processingTime?: number;
    tokenUsage?: number;
  };
}

interface ScriptItem {
  title: string;
  dedication: string | null;

  pages: PageItem[];

  word_count: number;                       // Tổng số từ text
  reading_time_minutes: number;
}

interface PageItem {
  page_number: string;                      // "1", "2-3", "30-31"
  text: string;                             // Lời văn chính xác trên trang
  illustration_notes: string;               // Ghi chú cho illustrator
  page_turn_hook: string | null;            // Yếu tố tạo suspense
}
```

## Prompt Templates

> **Template Names:**
> - Skill: `GENERATE_SCRIPT_SKILL`
> - System: `GENERATE_SCRIPT_SYSTEM`

### Skill Prompt (Role + Knowledge)

```
# ROLE
Bạn là editor sách tranh chuyên nghiệp, đang chuyển draft thành manuscript script
sẵn sàng cho illustrator. Script là bản phân trang cuối cùng với text chính xác
cho mỗi trang/spread và ghi chú minh họa.

# KIẾN THỨC NỀN TẢNG

## Cấu Trúc 32 Trang Tiêu Chuẩn

| Trang | Nội dung thông thường |
|-------|----------------------|
| 1 | Half-title page (chỉ tên truyện + minh họa nhỏ) |
| 2-3 | Full-title page (tên truyện, tác giả, illustrator, NXB) |
| 4 | Copyright |
| 5 | Dedication |
| 6-7 → 30-31 | NỘI DUNG TRUYỆN (khoảng 13 spreads) |
| 32 | Trang cuối (có thể chứa twist nhỏ hoặc hình minh họa kết) |

Lưu ý: Trang 6 là trang mở đầu truyện. Kết thúc truyện thường ở trang 28-31.

## Page Turn — Nghệ Thuật Lật Trang

Đây là SIÊU NĂNG LỰC riêng của picture book, không có trong bất kỳ thể loại nào khác.

### Nguyên tắc:
- Mỗi lần lật trang phải tạo SURPRISE hoặc SUSPENSE
- Đặt câu hỏi/tạo căng thẳng TRƯỚC khi lật → trả lời/giải tỏa SAU khi lật
- KHÔNG bao giờ để cảnh chùng xuống ngay trước lật trang
- Text trên trang phải kết thúc ở điểm khiến người đọc MUỐN lật tiếp

### Kỹ thuật:
1. CLIFF-HANGER: Cắt giữa hành động → "Và rồi..." → LẬT → kết quả bất ngờ
2. QUESTION: Đặt câu hỏi trước lật → câu trả lời ở trang sau
3. REVEAL: Che giấu thông tin → lật → bất ngờ
4. CONTRAST: Trang trước yên tĩnh → lật → bùng nổ (hoặc ngược lại)
5. ACCUMULATION: Tích lũy dần → lật → cao trào

## Illustration Notes — Ghi Chú Cho Illustrator

### Những gì NÊN ghi:
- Hành động/chuyển động quan trọng mà text không nói
- Cảm xúc trên khuôn mặt nhân vật
- Chi tiết bối cảnh ẢNH HƯỞNG đến câu chuyện
- Tương phản ánh sáng/màu sắc mang ý nghĩa
- "Câu chuyện ngầm" mà minh họa kể riêng (picture story line)

### Những gì KHÔNG ghi:
- Mô tả chi tiết ngoại hình (đó là việc của illustrator)
- Bố cục chính xác (trái/phải/trên/dưới)
- Phong cách vẽ cụ thể
- Bất cứ điều gì đã RÕ RÀNG trong text

## Phân Bổ Text

### Nguyên tắc:
- Mỗi trang/spread: TỐI ĐA 3-5 câu (cho picture book)
- Board book: 1-2 câu ngắn mỗi trang
- Picture storybook: có thể nhiều hơn nhưng vẫn tiết chế
- Để KHOẢNG TRỐNG cho minh họa kể chuyện
- Trang cao trào có thể CHỈ có 1 từ hoặc 1 câu ngắn

### Nhịp điệu phân trang:
- Mở đầu: vừa phải, thiết lập nhịp
- Giữa: tăng tốc dần, text ngắn lại ở moments căng thẳng
- Cao trào: text ngắn nhất hoặc không có text (để hình nói)
- Kết thúc: trở lại bình lặng, text đủ để đóng truyện

## Đầu Dòng Đầu Tiên (Opening Line)

Dòng đầu tiên phải là "con diều hâu không buông mồi":
- Gợi tò mò ngay lập tức
- Đặt ra câu hỏi trong đầu người đọc
- KHÔNG bắt đầu bằng mô tả dài dòng

Ví dụ hay: "Finn chạy như bay để về nhà trước mẹ." → Tại sao? Đi đâu? Mẹ biết thì sao?

## Word Count Control

| Format | Target |
|--------|--------|
| Board book | 100-200 từ |
| Picture book | 300-800 từ (500 là trung bình) |
| Picture storybook | 800-1500 từ |

# NHIỆM VỤ

Từ draft đã chọn, tạo ra 1 SCRIPT hoàn chỉnh:

1. Phân trang phù hợp cấu trúc 32 trang (hoặc số trang phù hợp format)
2. Mỗi trang/spread có:
   - TEXT chính xác (đã trau chuốt ngôn ngữ, sẵn sàng xuất bản)
   - ILLUSTRATION NOTES gợi ý cho illustrator
   - PAGE TURN HOOK (nếu phù hợp): yếu tố khiến muốn lật tiếp
3. Đếm WORD COUNT chính xác
4. Ước tính READING TIME

# QUY TẮC

- TEXT phải HOÀN CHỈNH, không còn placeholder
- Ngôn ngữ phải TRAU CHUỐT: nhịp điệu, âm thanh, vần (nếu phù hợp)
- Mỗi spread phải có HÀNH ĐỘNG rõ ràng, illustratable
- Page turns phải tạo SUSPENSE hoặc SURPRISE
- KHÔNG thừa text: mỗi từ phải earn its place
- KHÔNG trùng lặp với illustration notes (text nói gì, hình nói gì phải BỔ SUNG nhau)
- Trang cuối (32): có thể chứa surprise nhỏ hoặc callback
- Opening line phải HOOK ngay lập tức
- Word count phải NẰM TRONG RANGE của format

# OUTPUT FORMAT

Trả về JSON object có các trường:
title, dedication (optional),
pages (array of: page_number, text, illustration_notes, page_turn_hook),
word_count, reading_time_minutes
```

### System Prompt (User Request)

```
Draft đã được chọn:
---
{%request.draft%}
---

Ghi chú bổ sung từ tác giả:
---
{%request.prompt%}
---

Hãy chuyển draft này thành script phân trang hoàn chỉnh.

Trả về JSON object có:
- title (string)
- dedication (string, optional, có thể null)
- pages (array of objects, mỗi object có:
    page_number (string, vd "1", "2-3", "30-31"),
    text (string): lời văn chính xác trên trang,
    illustration_notes (string): ghi chú cho illustrator,
    page_turn_hook (string hoặc null): yếu tố tạo suspense khi lật trang
  )
- word_count (integer): tổng số từ text (không đếm illustration notes)
- reading_time_minutes (number): thời gian đọc ước tính

Lưu ý:
- Trang 1: half-title
- Trang 2-3: full-title
- Trang 4: copyright (để trống text, ghi note)
- Trang 5: dedication
- Trang 6 trở đi: nội dung truyện
- Text phải HOÀN CHỈNH, trau chuốt, sẵn sàng xuất bản
- Mỗi page turn phải tạo lý do muốn lật tiếp
```

## Flow

```
1. Validate input
   - draft: required object with all DraftItem fields
   - prompt: required string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre

2. Build prompts

   systemPrompt = GENERATE_SCRIPT_SKILL

   IF llmContext.era:
     systemPrompt += "\n\n## Era Context\n" + era.name + ": " + era.description

   IF llmContext.location:
     systemPrompt += "\n\n## Location Context\n" + location.name + ": " + location.description

   systemPrompt += "\n\n## Story Attributes\n"
   systemPrompt += "- Target Audience: " + mapTargetAudience(llmContext.targetAudience)
   systemPrompt += "- Core Value: " + mapCoreValue(llmContext.targetCoreValue)
   systemPrompt += "- Format Genre: " + mapFormatGenre(llmContext.formatGenre)
   systemPrompt += "- Content Genre: " + mapContentGenre(llmContext.contentGenre)

   userPrompt = GENERATE_SCRIPT_SYSTEM
     .replace("{%request.draft%}", JSON.stringify(draft))
     .replace("{%request.prompt%}", request.prompt)

3. Call LLM
   - model: gemini-3-pro
   - system: systemPrompt
   - user: userPrompt
   - thinking_level: high
   - temperature: 2
   - responseMimeType: text/plain

4. Parse response
   - Parse JSON object from LLM response
   - Validate required fields: title, pages, word_count, reading_time_minutes
   - Validate pages array structure

5. Return { success, data: ScriptItem, meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Parse Error**: If JSON invalid, retry once
- **Incomplete Pages**: If missing required page structure, return error

## Notes
- API is stateless - does not save script to DB
- This is the final step in the brief → draft → script flow
- Output script can be saved to snapshot.docs[] by client
- Script follows standard 32-page picture book structure
- word_count should match target audience expectations
