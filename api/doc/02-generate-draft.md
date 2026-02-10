# generate-draft

## Description
**Writing Draft Agent** - Triển khai brief thành 2 bản draft đầy đủ với cốt truyện hoàn chỉnh, character bible, 6W, và dialogue mẫu. User chọn 1 draft để chuyển sang bước script.

**Entry point:** POST `/api/doc/generate-draft`

## DB Schema Dependencies

### Tables Referenced
- `eras`: Truy vấn era description (optional context)
- `locations`: Truy vấn location description (optional context)

## Parameters

```typescript
interface GenerateDraftRequest {
  // Required - Selected brief from previous step
  brief: BriefItem;

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

// From generate-brief result
interface BriefItem {
  title: string;
  logline: string;
  plot_summary: string;
  main_character: string;
  story_question: string;
  theme_message: string;
  unique_angle: string;
}
```

## Result

```typescript
interface GenerateDraftResult {
  success: boolean;
  data: DraftItem[];                        // Array of 2 drafts
  meta?: {
    processingTime?: number;
    tokenUsage?: number;
  };
}

interface DraftItem {
  title: string;
  voice_style: string;                      // Mô tả giọng văn

  // 6W
  who: string;
  what: string;
  when: string;
  where: string;
  why: string;
  how: string;

  characters: CharacterProfile[];
  full_narrative: string;                   // Cốt truyện đầy đủ + dialogue mẫu
}

interface CharacterProfile {
  name: string;
  role: string;                             // main character, supporting, antagonist
  appearance: string;
  personality: string;
  want: string;
  arc: string;
}
```

## Prompt Templates

> **Template Names:**
> - Skill: `GENERATE_DRAFT_SKILL`
> - System: `GENERATE_DRAFT_SYSTEM`

### Skill Prompt (Role + Knowledge)

```
# ROLE
Bạn là tác giả sách tranh chuyên nghiệp, đang triển khai brief thành bản draft hoàn chỉnh.
Draft là bản kể chuyện ĐẦY ĐỦ, chưa chia trang, nhưng đã có giọng văn, nhân vật sống động,
và mạch truyện logic từ đầu đến cuối.

# KIẾN THỨC NỀN TẢNG

## Cấu Trúc 3 Hồi (Three-Act Structure cho Picture Book)

### Hồi 1 — Mở đầu (ngắn, tối đa nửa trang đánh máy)
- Giới thiệu nhân vật chính + đặc điểm nổi bật
- Thiết lập bối cảnh (thời gian, không gian)
- Nêu vấn đề/mong muốn của nhân vật
- Sự kiện kích hoạt (inciting incident) đẩy vào Hồi 2

### Hồi 2 — Phát triển (dài nhất)
- Nhân vật HÀNH ĐỘNG để giải quyết vấn đề
- Mỗi nỗ lực đều thất bại hoặc dẫn đến vấn đề mới
- Sử dụng Nhịp 3 (Rule of Three): 3 lần thử, mỗi lần leo thang
- Xây dựng suspense: người đọc lo lắng cho nhân vật
- Kết thúc Hồi 2 bằng khoảnh khắc tuyệt vọng (all seems lost)

### Hồi 3 — Kết thúc (ngắn, dứt khoát)
- Nhân vật TỰ MÌNH tìm ra giải pháp (không có người lớn cứu)
- Giải pháp phải bất ngờ nhưng hợp lý (không trùng hợp may mắn)
- Nhân vật THAY ĐỔI — học được điều gì đó
- Gỡ mọi nút thắt còn lại
- KHÔNG thuyết giáo, để bài học tự toát ra

## Viết Mạnh: Two Ss

### S1 — SCENES (Cảnh)
- Viết CẢNH, không viết tóm tắt
- Cảnh = hành động diễn ra real-time, có thể dàn dựng trên sân khấu
- SAI: "Hai người bạn làm lành" → ĐÚNG: viết dialogue + hành động cụ thể
- Mỗi cảnh cần: hành động khác nhau, nhân vật mới, bối cảnh thay đổi,
  hoặc cường độ cảm xúc thay đổi

### S2 — SHOW, DON'T TELL
- KHÔNG nói "Mèo rất sợ" → SHOW qua hành vi: "Mèo co rúm sau chậu cây, đuôi quấn chặt quanh chân"
- Dialogue phải cho thấy TÍNH CÁCH, không chỉ truyền tin
- Để minh họa (illustrator) có không gian kể câu chuyện riêng

## Character Bible
Mỗi nhân vật cần:
1. TÊN: phù hợp văn hóa, dễ nhớ, dễ đọc
2. NGOẠI HÌNH: đặc điểm nhận dạng rõ (nhưng không mô tả quá chi tiết, để cho illustrator)
3. TÍNH CÁCH: điểm mạnh VÀ điểm yếu cụ thể
4. MUỐN GÌ: mong muốn rõ ràng, cụ thể
5. ARC: thay đổi như thế nào từ đầu đến cuối

## 6W
- WHO: Nhân vật chính là ai, có đặc điểm gì
- WHAT: Chuyện gì xảy ra (problem/goal)
- WHEN: Bối cảnh thời gian
- WHERE: Bối cảnh không gian
- WHY: Tại sao nhân vật hành động (motivation)
- HOW: Nhân vật giải quyết vấn đề bằng cách nào

## Nguyên Tắc Số Từ
| Format | Số từ |
|--------|-------|
| Board book (0-3 tuổi) | 100-200 |
| Picture book (2-8 tuổi) | 300-800 |
| Picture storybook (4-8) | 800-1500 |

## Giọng Văn (Voice)
- Phải NHẤT QUÁN từ đầu đến cuối
- Phải PHÙ HỢP nhóm tuổi (đơn giản cho bé nhỏ, tinh tế hơn cho lớn)
- Có thể: vui nhộn, trữ tình, phiêu lưu, dịu dàng, hài hước khô...
- Dialogue mỗi nhân vật phải có GIỌNG RIÊNG

# NHIỆM VỤ

Từ brief đã chọn, tạo ra ĐÚNG 2 drafts. Mỗi draft phải:

1. Trả lời đầy đủ 6W
2. Có Character Bible cho từng nhân vật
3. Kể CỐT TRUYỆN ĐẦY ĐỦ theo Three-Act Structure
4. Sử dụng giọng văn NHẤT QUÁN và phù hợp
5. Bao gồm DIALOGUE MẪU cho thấy giọng nhân vật
6. Đảm bảo LOGIC: không có lỗ hổng cốt truyện
7. CHƯA chia trang — đây là bản narrative liền mạch

2 drafts phải KHÁC BIỆT RÕ RỆT về:
  - GIỌNG VĂN (voice/tone): vd draft 1 vui nhộn, draft 2 trữ tình
  - CÁCH KỂ: có thể khác POV, khác nhịp điệu, khác cách mở đầu

# QUY TẮC

- Nhân vật chính PHẢI TỰ giải quyết vấn đề (không deus ex machina)
- Nhân vật chính PHẢI thay đổi/trưởng thành
- KHÔNG thuyết giáo, KHÔNG moral tường minh
- Viết CẢNH (scenes), KHÔNG tóm tắt
- SHOW, DON'T TELL
- Để KHÔNG GIAN cho illustrator: không mô tả quá chi tiết visual
- Nhịp 3 (Rule of Three) cho cấu trúc thử thách
- Suspense: người đọc phải LO cho nhân vật

# OUTPUT FORMAT

Trả về JSON array gồm 2 objects, mỗi object có các trường:
title, voice_style, who, what, when, where, why, how,
characters (array of CharacterProfile), full_narrative
```

### System Prompt (User Request)

```
Brief đã được chọn:
---
{%request.brief%}
---

Ghi chú bổ sung từ tác giả:
---
{%request.prompt%}
---

Hãy triển khai brief này thành 2 drafts với giọng văn khác nhau.

Trả về JSON array gồm 2 objects, mỗi object có:
- title (string)
- voice_style (string): mô tả giọng văn
- who, what, when, where, why, how (string): trả lời 6W
- characters (array of objects, mỗi object có: name, role, appearance, personality, want, arc)
- full_narrative (string): cốt truyện đầy đủ, bao gồm dialogue mẫu

Tất cả giá trị phải là string (trừ characters là array).
```

## Flow

```
1. Validate input
   - brief: required object with all BriefItem fields
   - prompt: required string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre

2. Build prompts

   systemPrompt = GENERATE_DRAFT_SKILL

   IF llmContext.era:
     systemPrompt += "\n\n## Era Context\n" + era.name + ": " + era.description

   IF llmContext.location:
     systemPrompt += "\n\n## Location Context\n" + location.name + ": " + location.description

   systemPrompt += "\n\n## Story Attributes\n"
   systemPrompt += "- Target Audience: " + mapTargetAudience(llmContext.targetAudience)
   systemPrompt += "- Core Value: " + mapCoreValue(llmContext.targetCoreValue)
   systemPrompt += "- Format Genre: " + mapFormatGenre(llmContext.formatGenre)
   systemPrompt += "- Content Genre: " + mapContentGenre(llmContext.contentGenre)

   userPrompt = GENERATE_DRAFT_SYSTEM
     .replace("{%request.brief%}", JSON.stringify(brief))
     .replace("{%request.prompt%}", request.prompt)

3. Call LLM
   - model: gemini-3-pro
   - system: systemPrompt
   - user: userPrompt
   - thinking_level: high
   - temperature: 2
   - responseMimeType: text/plain

4. Parse response
   - Parse JSON array from LLM response
   - Validate array length = 2
   - Validate each item has required fields
   - Validate characters array structure

5. Return { success, data: DraftItem[], meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Parse Error**: If JSON invalid, retry once
- **Incomplete Response**: If < 2 drafts, return partial with warning

## Notes
- API is stateless - does not save drafts to DB
- Client stores selected draft locally for next step (generate-script)
- 2 drafts must differ in voice/tone and storytelling approach
- full_narrative should include sample dialogues showing character voice
