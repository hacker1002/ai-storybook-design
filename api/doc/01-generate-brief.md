# generate-brief

## Description
**Writing Brief Agent** - Tạo 3 brief tiềm năng từ ý tưởng truyện. Mỗi brief bao gồm: title, logline, plot_summary, main_character, story_question, theme_message, unique_angle.

**Entry point:** POST `/api/doc/generate-brief`

## DB Schema Dependencies

### Tables Referenced
- `eras`: Truy vấn era description (optional context)
- `locations`: Truy vấn location description (optional context)

## Parameters

```typescript
interface GenerateBriefRequest {
  // Required - User input
  prompt: string;                           // Ý tưởng truyện từ tác giả

  // Required - LLM Context
  llmContext: {
    // Story attributes (required)
    targetAudience: 1 | 2 | 3 | 4;          // 1: kindergarten, 2: preschool, 3: primary, 4: middle grade
    targetCoreValue: number;                 // 1-21: Core values
    formatGenre: 1 | 2 | 3 | 4 | 5 | 6;     // Narrative, Lullaby, Concept, Non-fiction, Early Reader, Wordless
    contentGenre: number;                    // 1-11: Mystery, Fantasy, etc.

    // Additional context (optional)
    era?: {
      id: string;
      name: string;
      description: string;
    };
    location?: {
      id: string;
      name: string;
      description: string;
    };
  };
}
```

### Target Audience Mapping
| Value | Name | Age Range | Reading Level |
|-------|------|-----------|---------------|
| 1 | Kindergarten | 2-3 tuổi | Từ rất đơn giản, câu 3-5 từ |
| 2 | Preschool | 4-5 tuổi | Từ đơn giản, câu ngắn |
| 3 | Primary | 6-8 tuổi | Câu phức, từ vựng đa dạng |
| 4 | Middle Grade | 9+ tuổi | Nội dung phức tạp, đa chủ đề |

## Result

```typescript
interface GenerateBriefResult {
  success: boolean;
  data: BriefItem[];                        // Array of 3 briefs
  meta?: {
    processingTime?: number;                // ms
    tokenUsage?: number;
  };
}

interface BriefItem {
  title: string;                            // Tiêu đề hấp dẫn, ngắn gọn
  logline: string;                          // 1 câu khiến tò mò ngay lập tức
  plot_summary: string;                     // 3-5 câu: mở đầu → xung đột → cao trào → kết thúc
  main_character: string;                   // Đặc điểm nổi bật phù hợp plot
  story_question: string;                   // Câu hỏi cốt lõi
  theme_message: string;                    // Thông điệp, bài học
  unique_angle: string;                     // Điều gì khiến truyện đặc biệt
}
```

## Prompt Templates

> **Template Names:**
> - Skill: `GENERATE_BRIEF_SKILL`
> - System: `GENERATE_BRIEF_SYSTEM`

### Skill Prompt (Role + Knowledge)

```
# ROLE
Bạn là chuyên gia sáng tạo nội dung sách tranh (picture book) với kinh nghiệm sâu rộng
trong ngành xuất bản trẻ em. Nhiệm vụ của bạn là biến ý tưởng thô thành các brief chuyên nghiệp,
mỗi brief phải khiến người duyệt thấy NGAY tại sao ý tưởng này xứng đáng trở thành sách.

# KIẾN THỨC NỀN TẢNG

## Story Question (Câu Hỏi Cốt Lõi)
Mỗi truyện tranh hay đều ẩn chứa MỘT câu hỏi lớn mà người đọc muốn tìm câu trả lời.
Ví dụ:
  "Where the Wild Things Are" → Làm sao kiểm soát cơn giận?
  "We Found a Hat" → Hai chú rùa có chia sẻ được chiếc mũ?
  "Chrysanthemum" → Liệu cô bé có tự hào về cái tên của mình?

Câu hỏi này KHÔNG được viết trực tiếp vào truyện, nhưng phải sáng rõ trong đầu người viết
và xuyên suốt mọi chi tiết.

## 7 Kiểu Cốt Truyện Cơ Bản (Christopher Booker)
1. Overcoming the Monster: Đối mặt và chiến thắng mối đe dọa
2. Rags to Riches: Từ bị xem thường đến được công nhận
3. The Quest: Hành trình tìm kiếm với chướng ngại
4. Voyage and Return: Đến thế giới mới → phiêu lưu → trở về đã thay đổi
5. Comedy: Hiểu lầm → rắc rối leo thang → sự thật được hé lộ
6. Tragedy (hiếm, thường mềm hóa): Sai lầm → hậu quả → bài học
7. Rebirth: Nhân vật bế tắc → xúc tác thay đổi → biến đổi

## 4 Loại Xung Đột
1. Character vs Self: Đấu tranh nội tâm (sợ hãi, tự ti, kiểm soát cảm xúc)
2. Character vs Others: Xung đột giữa các nhân vật (bạn bè, gia đình, kẻ bắt nạt)
3. Character vs Society: Đi ngược lại chuẩn mực (khác biệt, thách thức kỳ vọng)
4. Character vs Nature: Đối mặt với thiên nhiên/môi trường

## Yêu Cầu Theo Nhóm Tuổi
| Tuổi | Đặc điểm cốt truyện |
|------|---------------------|
| 2-4  | Rất đơn giản, cụ thể, ít nhân vật, nhân quả rõ, kết thúc vui |
| 4-6  | Tình huống xã hội, thử thách cảm xúc, hài hước, có sự trưởng thành |
| 6-8  | Cảm xúc phức tạp hơn, nhiều tầng ý nghĩa, kết thúc tinh tế |

## Tiêu Chí Đánh Giá Ý Tưởng
Mỗi brief phải đạt điểm cao trên 7 chiều:
1. Format Fit: Phù hợp format sách tranh
2. Plot Clarity: Cốt truyện rõ ràng
3. Conflict Strength: Xung đột hấp dẫn, liên quan (relatable)
4. Uniqueness: Góc nhìn mới mẻ, không "đã thấy 1000 lần"
5. Age Appropriateness: Phù hợp nhóm tuổi
6. Illustration Potential: Gợi mở không gian cho minh họa
7. Universal Appeal: Ai cũng thấy đồng cảm

# NHIỆM VỤ

Từ ý tưởng của user, tạo ra ĐÚNG 3 briefs. Mỗi brief phải:

1. Có TITLE hấp dẫn, ngắn gọn, trẻ có thể đọc được
2. Có LOGLINE 1 câu khiến người nghe tò mò ngay lập tức
3. Có PLOT SUMMARY ngắn gọn (3~5 câu) phác họa cốt truyện chính:
   mở đầu → xung đột → cao trào → kết thúc
4. Có MAIN CHARACTER với đặc điểm nổi bật phù hợp plot
5. Trả lời được STORY QUESTION: câu hỏi cốt lõi
6. Thể hiện rõ THEME/MESSAGE: thông điệp, bài học
7. Nêu được UNIQUE ANGLE: điều gì khiến truyện này đặc biệt

3 briefs phải KHÁC BIỆT RÕ RỆT về:
  - Góc tiếp cận (cùng ý tưởng nhưng kể khác nhau)
  - Kiểu cốt truyện (nếu có thể)
  - Tone/mood (vui nhộn vs trữ tình vs phiêu lưu...)
  - Nhân vật (đặc điểm, loài, tính cách)

# QUY TẮC

- KHÔNG viết cốt truyện chi tiết, chỉ phác thảo đủ để thấy tiềm năng
- KHÔNG chia trang, KHÔNG viết dialogue cụ thể
- MỖI brief phải đứng độc lập, tự nó đã hấp dẫn
- Ngôn ngữ brief phải chuyên nghiệp nhưng dễ hiểu
- Luôn kiểm tra: ý tưởng có "đã bị làm 1000 lần" không?
  Nếu có, phải tìm FRESH ANGLE thực sự mới

# OUTPUT FORMAT

Trả về JSON array gồm 3 objects, mỗi object có các trường:
title, logline, plot_summary, main_character, story_question, theme_message, unique_angle
```

### System Prompt (User Request)

```
Ý tưởng truyện từ tác giả:
---
{%request.prompt%}
---

Hãy tạo 3 briefs khác biệt rõ rệt cho ý tưởng này.
Trả về JSON array gồm 3 objects với các trường:
title, logline, plot_summary, main_character, story_question, theme_message, unique_angle

Tất cả giá trị phải là string.
```

## Flow

```
1. Validate input
   - prompt: required, non-empty string
   - llmContext: required object with targetAudience, targetCoreValue, formatGenre, contentGenre

2. Build prompts

   systemPrompt = GENERATE_BRIEF_SKILL

   IF llmContext.era:
     systemPrompt += "\n\n## Era Context\n" + era.name + ": " + era.description

   IF llmContext.location:
     systemPrompt += "\n\n## Location Context\n" + location.name + ": " + location.description

   systemPrompt += "\n\n## Story Attributes\n"
   systemPrompt += "- Target Audience: " + mapTargetAudience(llmContext.targetAudience)
   systemPrompt += "- Core Value: " + mapCoreValue(llmContext.targetCoreValue)
   systemPrompt += "- Format Genre: " + mapFormatGenre(llmContext.formatGenre)
   systemPrompt += "- Content Genre: " + mapContentGenre(llmContext.contentGenre)

   userPrompt = GENERATE_BRIEF_SYSTEM.replace("{%request.prompt%}", request.prompt)

3. Call LLM
   - model: gemini-3-pro
   - system: systemPrompt
   - user: userPrompt
   - thinking_level: high
   - temperature: 2
   - responseMimeType: text/plain

4. Parse response
   - Parse JSON array from LLM response
   - Validate array length = 3
   - Validate each item has required fields (all strings)

5. Return { success, data: BriefItem[], meta }
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **LLM Error**: Log error, return 500 with generic message
- **Parse Error**: If JSON invalid, retry once with explicit JSON instruction
- **Incomplete Response**: If < 3 briefs, return partial with warning

## Notes
- API is stateless - does not save briefs to DB
- Client stores selected brief locally for next step (generate-draft)
- 3 briefs must be distinctly different in approach, tone, character
