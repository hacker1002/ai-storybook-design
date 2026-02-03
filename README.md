# AI Storybook Canvas Documentation

> Comprehensive documentation for AI Storybook Canvas API

## 📖 Overview

**AI Storybook Canvas** is a platform for creating children's picture books with AI assistance. This documentation covers the complete API reference, database schema, and implementation guides.

## 🎯 Key Features

- **Manuscript Generation** - Create complete story manuscripts from ideas
- **Character Management** - Design and manage characters, props, and stages
- **AI Image Generation** - Generate illustrations with AI
- **Audio/Voice Synthesis** - Create narration and sound effects
- **Multi-format Export** - Export to PDF, ePub, and Video

## 📚 Documentation Structure

### App Features
User-facing features và flows.

| Feature | Description |
|---------|-------------|
| [Story Idea Brainstorming](app/ai-assistant/story-idea-brainstorming.md) | Tạo truyện từ ý tưởng qua AI chat |
| [Generate Manuscript](app/generate-manuscript/generate-manuscript.md) | Background job tạo manuscript hoàn chỉnh |

### API - Chat
Realtime chat endpoints với AI.

| Endpoint | Description |
|----------|-------------|
| [Story Brainstorming Initial](api/chat/00-story-brainstorming-initial.md) | Extract story idea + params từ prompt |
| [Story Brainstorming Draft](api/chat/01-story-brainstorming-draft.md) | Generate/refine docs qua chat |

### API - Text Generation
Pipeline tạo manuscript từ story idea.

| Endpoint | Description |
|----------|-------------|
| [Generate Manuscript](api/text-generation/00-generate-manuscript.md) | Orchestrator - điều phối 5 steps |
| [Story Draft](api/text-generation/01-generate-story-draft.md) | Step 1: Tạo khung truyện |
| [Spread Visual Plan](api/text-generation/02-generate-spread-visual-plan.md) | Step 2: Visual descriptions |
| [Text Refinement](api/text-generation/03-generate-text-refinement.md) | Step 3: Refine text |
| [Spread Composition](api/text-generation/04-generate-spread-composition.md) | Step 4: Layout geometry |
| [Quality Check](api/text-generation/05-generate-quality-check.md) | Step 5: Validate story |

### API - Visual Description
Generate visual descriptions cho entities.

| Endpoint | Description |
|----------|-------------|
| [Character](api/text-generation/06-generate-visual-description-character.md) | Character visual desc |
| [Prop](api/text-generation/07-generate-visual-description-prop.md) | Prop visual desc |
| [Stage](api/text-generation/08-generate-visual-description-stage.md) | Stage visual desc |
| [Spread](api/text-generation/09-generate-visual-description-spread.md) | Spread visual desc |

### API - Translation & Poetry
| Endpoint | Description |
|----------|-------------|
| [Translate Content](api/text-generation/10-translate-content.md) | Dịch nội dung |
| [Generate Poetry](api/text-generation/11-generate-poetry.md) | Tạo thơ/vần điệu |

### Database Schema
👉 [View Database Schema](DATABASE-SCHEMA.md)

## 🚀 Deploy

### Local Development
```bash
npx serve .
# Open http://localhost:3000
```

### Deploy to Netlify
```bash
npm install -g netlify-cli
netlify login
netlify deploy --prod
```

**Settings:**
- Publish directory: `.`
- Enable password protection: Netlify Dashboard → Settings → Visitor access

---

**Last Updated:** January 28, 2026
