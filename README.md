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

## 🚀 Quick Start

### Database Schema
Learn about the complete database structure, including stories, snapshots, characters, and resources.

👉 [View Database Schema](CLAUDE.md)

### API Functions
Explore the text generation pipeline and individual API functions.

👉 [API Functions Overview](api/text-generation/README.md)

## 🚀 Deploy

### Local Development
```bash
npx serve .
# Open http://localhost:3000
```

### Deploy to Netlify
```bash
# Install Netlify CLI
npm install -g netlify-cli

# Login & Deploy
netlify login
netlify deploy --prod
```

**Settings:**
- Publish directory: `.` (current directory)
- Enable password protection in Netlify Dashboard → Settings → Visitor access

## 🤝 Support

For questions or issues, please refer to the detailed documentation in each section.

---

**Last Updated:** January 22, 2026
