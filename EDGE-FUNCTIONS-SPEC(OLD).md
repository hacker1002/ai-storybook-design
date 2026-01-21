# Supabase Edge Functions Specification

## Tổng quan

Tài liệu này liệt kê các Edge Functions cần xây dựng cho AI Storybook Canvas, ngoài các CRUD operations cơ bản của Supabase.

---

## 1. AI IMAGE GENERATION

### 1.1 `generate-sketch`
**Mục đích:** Tạo sketch/line art từ visual description

**Sử dụng tại:**
- VisualsTab → Button "Generate"
- Character/Stage/Prop creation
- SpreadImage generation

**Input:**
```typescript
{
  entityType: 'character' | 'stage' | 'prop' | 'spread_image';
  entityId: string;
  visualDescription: string;
  sceneContext?: {
    stageId?: string;
    actions?: string;
    temporal?: { era, season, weather, timeOfDay };
    sensory?: { atmosphere, lighting, colorPalette, mood };
  };
  imageReferences?: string[];  // URLs of reference images
  artStyleId?: string;
  count?: number;  // Number of variations (default: 4)
}
```

**Output:**
```typescript
{
  sketches: Array<{
    mediaUrl: string;
    thumbnailUrl: string;
    promptUsed: string;
  }>;
  creditsUsed: number;
}
```

**AI Provider:** Stable Diffusion XL / DALL-E 3 / Midjourney API

---

### 1.2 `generate-illustration`
**Mục đích:** Tạo full-color illustration từ sketch hoặc description

**Sử dụng tại:**
- VisualsTab → sau khi chọn sketch
- Bulk generation cho toàn bộ spreads

**Input:**
```typescript
{
  entityType: 'character' | 'stage' | 'prop' | 'spread_image';
  entityId: string;
  sketchUrl?: string;  // If converting from sketch
  visualDescription: string;
  artStyleId: string;
  imageReferences?: string[];
  count?: number;
}
```

**Output:**
```typescript
{
  illustrations: Array<{
    mediaUrl: string;
    hiresUrl: string;
    thumbnailUrl: string;
  }>;
  creditsUsed: number;
}
```

---

### 1.3 `upscale-image`
**Mục đích:** Upscale ảnh lên độ phân giải cao cho print

**Sử dụng tại:**
- Export to print
- Final image approval

**Input:**
```typescript
{
  imageUrl: string;
  targetResolution: '2x' | '4x' | 'print_300dpi';
  spreadDimension?: string;  // e.g., "5760x2880"
}
```

**Output:**
```typescript
{
  hiresImageUrl: string;
  resolution: { width: number; height: number; dpi: number };
}
```

---

### 1.4 `generate-cover`
**Mục đích:** Tạo ảnh bìa sách

**Sử dụng tại:**
- Story settings → Cover generation

**Input:**
```typescript
{
  storyId: string;
  visualDescription: string;
  title: string;
  artStyleId: string;
  characters?: string[];  // Character IDs to include
  dimension: 'square' | 'portrait' | 'landscape';
}
```

---

### 1.5 `generate-storyboard`
**Mục đích:** Tạo storyboard thumbnails cho toàn bộ truyện

**Sử dụng tại:**
- Story overview
- Planning phase

**Input:**
```typescript
{
  storyId: string;
  spreads: Array<{
    spreadId: string;
    description: string;
  }>;
  artStyleId: string;
}
```

**Output:**
```typescript
{
  storyboardImages: Array<{
    spreadId: string;
    thumbnailUrl: string;
  }>;
}
```

---

## 2. AI TEXT GENERATION

### 2.1 `generate-manuscript`
**Mục đích:** Tạo hoặc expand manuscript từ outline

**Sử dụng tại:**
- ManuscriptEditor
- AI Chat Panel
- Story creation wizard

**Input:**
```typescript
{
  storyId: string;
  action: 'generate_full' | 'expand_scene' | 'rewrite' | 'continue';
  context: {
    title: string;
    description: string;
    targetAge: string;
    targetCoreValue: string;
    genre: number;
    characters: Array<{ name, personality, role }>;
    existingContent?: string;
    sceneNumber?: number;
  };
  style?: {
    tone: 'playful' | 'serious' | 'educational' | 'adventurous';
    complexity: 'simple' | 'moderate' | 'rich';
  };
  language: string;
}
```

**Output:**
```typescript
{
  manuscript: string;  // Markdown format
  scenes: Array<{
    number: number;
    title: string;
    content: string;
    visualSuggestions: string[];
  }>;
  wordCount: number;
}
```

**AI Provider:** Claude / GPT-4

---

### 2.2 `generate-visual-description`
**Mục đích:** Tạo visual description chi tiết cho AI image generation

**Sử dụng tại:**
- Character/Stage/Prop creation
- SpreadImage creation
- Auto-generate từ manuscript text

**Input:**
```typescript
{
  entityType: 'character' | 'stage' | 'prop' | 'spread_image';
  basicInfo: {
    name?: string;
    description?: string;
    // Character specific
    personality?: object;
    appearance?: object;
    // Stage specific
    location?: string;
    when?: string;
    // Scene specific
    manuscriptExcerpt?: string;
    characters?: string[];
    actions?: string;
  };
  artStyleId?: string;
  targetLength: 'short' | 'detailed';
}
```

**Output:**
```typescript
{
  visualDescription: string;
  keywords: string[];
  suggestedReferences: string[];
}
```

---

### 2.3 `translate-content`
**Mục đích:** Dịch nội dung sang ngôn ngữ khác

**Sử dụng tại:**
- Multi-language textboxes
- Manuscript translation
- Remix wizard (language change)

**Input:**
```typescript
{
  content: string | Array<{ id: string; text: string }>;
  sourceLanguage: string;
  targetLanguage: string;
  context?: {
    storyTitle: string;
    targetAge: string;
    characterNames?: Record<string, string>;  // Name mappings
  };
  preserveFormatting: boolean;
}
```

**Output:**
```typescript
{
  translations: Array<{
    id?: string;
    originalText: string;
    translatedText: string;
  }>;
}
```

---

### 2.4 `enhance-prompt`
**Mục đích:** Cải thiện prompt cho image generation

**Sử dụng tại:**
- Before calling generate-sketch/illustration
- Manual prompt enhancement

**Input:**
```typescript
{
  userPrompt: string;
  entityType: 'character' | 'stage' | 'prop' | 'scene';
  artStyle?: string;
  additionalContext?: {
    storyGenre: string;
    targetAge: string;
    existingElements: string[];
  };
}
```

**Output:**
```typescript
{
  enhancedPrompt: string;
  negativePrompt: string;
  suggestedParameters: {
    steps: number;
    cfgScale: number;
    sampler: string;
  };
}
```

---

## 3. AI AUDIO GENERATION

### 3.1 `generate-voice`
**Mục đích:** Text-to-Speech cho narration và character dialogue

**Sử dụng tại:**
- VoiceTab → Button "Generate"
- Textbox audio generation
- Character voice preview

**Input:**
```typescript
{
  text: string;
  voiceSettings: {
    voiceId?: string;  // System voice or custom
    customVoiceUrl?: string;  // User uploaded voice sample
    stability: number;  // 0-1
    clarity: number;  // 0-1
    similarity: number;  // 0-1
    styleExaggeration: number;  // 0-1
    speakerBoost: boolean;
  };
  emotion?: 'neutral' | 'happy' | 'sad' | 'excited' | 'angry' | 'whisper';
  speed?: number;  // 0.5 - 2.0
  language: string;
}
```

**Output:**
```typescript
{
  audioUrl: string;
  duration: number;  // seconds
  format: 'mp3' | 'wav';
}
```

**AI Provider:** ElevenLabs / Azure TTS / Google TTS

---

### 3.2 `clone-voice`
**Mục đích:** Tạo custom voice từ audio sample

**Sử dụng tại:**
- Character voice setup
- Remix wizard (voice mapping)

**Input:**
```typescript
{
  characterId: string;
  audioSamples: string[];  // URLs of audio files
  voiceName: string;
  description?: string;
}
```

**Output:**
```typescript
{
  voiceId: string;
  previewUrl: string;
  quality: 'low' | 'medium' | 'high';
}
```

---

### 3.3 `generate-sound-effect`
**Mục đích:** Tạo sound effect từ description

**Sử dụng tại:**
- SoundTab → Button "Generate"
- Stage/Prop sound effects
- Interaction sounds

**Input:**
```typescript
{
  description: string;
  duration?: number;  // seconds
  category?: 'ambient' | 'action' | 'ui' | 'nature' | 'voice';
}
```

**Output:**
```typescript
{
  audioUrl: string;
  duration: number;
  variations?: string[];  // Alternative versions
}
```

**AI Provider:** ElevenLabs Sound Effects / AudioGen

---

### 3.4 `generate-background-music`
**Mục đích:** Tạo background music cho story

**Sử dụng tại:**
- Story settings → Background music

**Input:**
```typescript
{
  storyId: string;
  mood: 'happy' | 'mysterious' | 'adventurous' | 'calm' | 'dramatic';
  genre: 'orchestral' | 'acoustic' | 'electronic' | 'lullaby';
  duration: number;  // seconds
  tempo?: 'slow' | 'medium' | 'fast';
}
```

**Output:**
```typescript
{
  audioUrl: string;
  loopable: boolean;
  bpm: number;
}
```

**AI Provider:** Suno AI / MusicGen

---

## 4. AI VIDEO GENERATION

### 4.1 `generate-animation`
**Mục đích:** Tạo animation từ static image

**Sử dụng tại:**
- SpreadVideo creation
- Character animation
- Interactive elements

**Input:**
```typescript
{
  imageUrl: string;
  animationType: 'pan' | 'zoom' | 'parallax' | 'character_motion' | 'ambient';
  duration: number;  // seconds
  parameters?: {
    direction?: 'left' | 'right' | 'up' | 'down';
    intensity?: number;
    loop?: boolean;
  };
}
```

**Output:**
```typescript
{
  videoUrl: string;
  thumbnailUrl: string;
  duration: number;
  format: 'mp4' | 'webm';
}
```

**AI Provider:** Runway ML / Pika Labs / Stable Video

---

### 4.2 `generate-lipsync`
**Mục đích:** Tạo lip-sync animation cho character

**Sử dụng tại:**
- Character dialogue scenes
- Interactive storybook playback

**Input:**
```typescript
{
  characterImageUrl: string;
  audioUrl: string;
  faceRegion?: { x, y, w, h };  // Face bounding box
}
```

**Output:**
```typescript
{
  videoUrl: string;
  duration: number;
}
```

**AI Provider:** D-ID / HeyGen / SadTalker

---

## 5. AI ASSISTANT / CHAT

### 5.1 `ai-chat`
**Mục đích:** AI assistant trong AIChatPanel

**Sử dụng tại:**
- AIChatPanel
- Contextual help throughout app

**Input:**
```typescript
{
  messages: Array<{
    role: 'user' | 'assistant' | 'system';
    content: string;
  }>;
  context: {
    currentView: string;
    activeEntity?: { type, id, data };
    storyContext?: {
      storyId: string;
      title: string;
      characters: string[];
      currentSpread?: number;
    };
  };
  capabilities: ('generate_image' | 'edit_text' | 'suggest_improvements')[];
}
```

**Output:**
```typescript
{
  message: string;
  images?: string[];
  clarificationQuestions?: Array<{
    id: string;
    question: string;
    options: Array<{ id, label, description }>;
    multiSelect: boolean;
  }>;
  suggestedActions?: Array<{
    action: string;
    params: object;
    label: string;
  }>;
}
```

**AI Provider:** Claude / GPT-4

---

### 5.2 `analyze-story`
**Mục đích:** Phân tích và đưa ra suggestions cho story

**Sử dụng tại:**
- WarningsView
- Story review
- AI Chat suggestions

**Input:**
```typescript
{
  storyId: string;
  analysisTypes: ('consistency' | 'pacing' | 'age_appropriateness' | 'visual_balance' | 'accessibility')[];
}
```

**Output:**
```typescript
{
  warnings: Array<{
    level: 'error' | 'warning' | 'info';
    category: string;
    title: string;
    description: string;
    location: { spreadId?, elementId?, pageNumber? };
    recommendation: string;
    autoFixAvailable: boolean;
    autoFixAction?: object;
  }>;
  score: {
    overall: number;
    consistency: number;
    readability: number;
    engagement: number;
  };
}
```

---

## 6. REMIX & PERSONALIZATION

### 6.1 `remix-story`
**Mục đích:** Tạo personalized version của story

**Sử dụng tại:**
- RemixWizardModal

**Input:**
```typescript
{
  sourceStoryId: string;
  targetStoryId: string;  // New story to create
  mappings: {
    characters: Record<string, string | null>;  // source -> target
    props: Record<string, string | null>;
    language: string;
    narrationVoice: string;
  };
  options: {
    regenerateImages: boolean;
    regenerateVoice: boolean;
    keepOriginalText: boolean;
  };
}
```

**Output:**
```typescript
{
  storyId: string;
  status: 'queued' | 'processing' | 'completed' | 'failed';
  progress?: {
    current: number;
    total: number;
    currentStep: string;
  };
}
```

---

### 6.2 `swap-character`
**Mục đích:** Thay thế character trong tất cả images

**Sử dụng tại:**
- Remix wizard
- Character replacement

**Input:**
```typescript
{
  storyId: string;
  sourceCharacterId: string;
  targetCharacter: {
    id?: string;
    visualDescription: string;
    imageReferences?: string[];
  };
  affectedSpreads?: string[];  // If null, apply to all
}
```

**Output:**
```typescript
{
  jobId: string;
  affectedImages: number;
  estimatedTime: number;  // seconds
}
```

**AI Provider:** IP-Adapter / InstantID / PhotoMaker

---

## 7. EXPORT & PUBLISHING

### 7.1 `export-pdf`
**Mục đích:** Export story thành PDF

**Sử dụng tại:**
- Export menu
- Print preparation

**Input:**
```typescript
{
  storyId: string;
  format: 'digital' | 'print';
  options: {
    resolution: '150dpi' | '300dpi';
    colorProfile: 'RGB' | 'CMYK';
    includeBleed: boolean;
    spreadLayout: 'single_page' | 'facing_pages';
    language: string;
  };
}
```

**Output:**
```typescript
{
  downloadUrl: string;
  fileSize: number;
  pageCount: number;
  expiresAt: string;
}
```

---

### 7.2 `export-epub`
**Mục đích:** Export thành ePub format

**Input:**
```typescript
{
  storyId: string;
  options: {
    includeAudio: boolean;
    includeInteractions: boolean;
    language: string;
    coverImageQuality: 'standard' | 'high';
  };
}
```

---

### 7.3 `export-video`
**Mục đích:** Export story thành video

**Sử dụng tại:**
- Video export for social media
- Animated storybook

**Input:**
```typescript
{
  storyId: string;
  options: {
    resolution: '720p' | '1080p' | '4k';
    includeNarration: boolean;
    includeBackgroundMusic: boolean;
    transitionStyle: 'fade' | 'slide' | 'zoom';
    pageDuration: number;  // seconds per spread
    language: string;
  };
}
```

**Output:**
```typescript
{
  jobId: string;
  status: 'queued' | 'processing' | 'completed';
  videoUrl?: string;
  duration?: number;
}
```

---

## 8. UTILITY FUNCTIONS

### 8.1 `upload-media`
**Mục đích:** Upload và process media files

**Input:**
```typescript
{
  file: File;
  type: 'image' | 'audio' | 'video';
  entityType: string;
  entityId: string;
  processOptions?: {
    resize?: { maxWidth, maxHeight };
    generateThumbnail?: boolean;
    extractColors?: boolean;  // For images
    transcribe?: boolean;  // For audio
  };
}
```

**Output:**
```typescript
{
  mediaUrl: string;
  thumbnailUrl?: string;
  metadata: {
    width?: number;
    height?: number;
    duration?: number;
    dominantColors?: string[];
    transcription?: string;
  };
}
```

---

### 8.2 `check-credits`
**Mục đích:** Kiểm tra và quản lý AI credits

**Input:**
```typescript
{
  userId: string;
  action?: 'check' | 'estimate' | 'deduct';
  operation?: string;
  amount?: number;
}
```

**Output:**
```typescript
{
  currentCredits: number;
  estimatedCost?: number;
  sufficient: boolean;
}
```

---

### 8.3 `process-job-status`
**Mục đích:** Kiểm tra trạng thái của async jobs

**Input:**
```typescript
{
  jobId: string;
  jobType: 'remix' | 'export' | 'bulk_generate';
}
```

**Output:**
```typescript
{
  status: 'queued' | 'processing' | 'completed' | 'failed';
  progress: number;  // 0-100
  currentStep?: string;
  result?: object;
  error?: string;
}
```

---

## Tóm tắt Edge Functions

| Category | Function | Priority | AI Provider |
|----------|----------|----------|-------------|
| **Image** | generate-sketch | P0 | SD XL / DALL-E |
| **Image** | generate-illustration | P0 | SD XL / DALL-E |
| **Image** | upscale-image | P1 | Real-ESRGAN |
| **Image** | generate-cover | P1 | SD XL / DALL-E |
| **Image** | generate-storyboard | P2 | SD XL |
| **Text** | generate-manuscript | P0 | Claude / GPT-4 |
| **Text** | generate-visual-description | P0 | Claude / GPT-4 |
| **Text** | translate-content | P1 | Claude / GPT-4 |
| **Text** | enhance-prompt | P1 | Claude / GPT-4 |
| **Audio** | generate-voice | P0 | ElevenLabs |
| **Audio** | clone-voice | P1 | ElevenLabs |
| **Audio** | generate-sound-effect | P2 | ElevenLabs |
| **Audio** | generate-background-music | P2 | Suno AI |
| **Video** | generate-animation | P2 | Runway ML |
| **Video** | generate-lipsync | P3 | D-ID |
| **Chat** | ai-chat | P0 | Claude / GPT-4 |
| **Chat** | analyze-story | P1 | Claude / GPT-4 |
| **Remix** | remix-story | P1 | Multiple |
| **Remix** | swap-character | P2 | IP-Adapter |
| **Export** | export-pdf | P0 | PDFKit |
| **Export** | export-epub | P1 | ePubJS |
| **Export** | export-video | P2 | FFmpeg |
| **Utility** | upload-media | P0 | Sharp / FFmpeg |
| **Utility** | check-credits | P0 | - |
| **Utility** | process-job-status | P0 | - |

**Priority Legend:**
- P0: MVP - Cần có cho launch
- P1: Important - Cần có trong 1-2 tháng sau launch
- P2: Nice to have - Roadmap Q2
- P3: Future - Roadmap Q3+

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (React)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Supabase Edge Functions                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Quick APIs  │  │ Async Jobs  │  │ Webhook Handlers    │  │
│  │ (<10s)      │  │ (Queue)     │  │ (AI Callbacks)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   AI Services   │ │ Supabase DB     │ │ Supabase Storage│
│                 │ │                 │ │                 │
│ - OpenAI        │ │ - Stories       │ │ - Images        │
│ - Anthropic     │ │ - Snapshots     │ │ - Audio         │
│ - ElevenLabs    │ │ - Jobs Queue    │ │ - Video         │
│ - Stability AI  │ │ - Credits       │ │ - Exports       │
│ - Runway ML     │ │                 │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Job Queue Pattern (for long-running tasks)

```typescript
// 1. Client calls edge function
const { jobId } = await supabase.functions.invoke('remix-story', { body: params });

// 2. Edge function creates job record and returns immediately
// INSERT INTO jobs (id, type, status, params) VALUES (...)

// 3. Background worker (another edge function on cron) processes jobs
// SELECT * FROM jobs WHERE status = 'queued' ORDER BY created_at LIMIT 5

// 4. Client polls for status or subscribes to realtime
const subscription = supabase
  .channel('job-updates')
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'jobs',
    filter: `id=eq.${jobId}`
  }, handleUpdate)
  .subscribe();
```
