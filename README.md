# ken --verbose

Personal portfolio & blog built with [Astro](https://astro.build/).

## Development

```bash
npm install
npm run dev      # Start dev server at localhost:4321
npm run build    # Build for production
npm run preview  # Preview production build
```

## Adding Blog Posts

Create a new `.md` file in `src/content/blog/` with this frontmatter:

```markdown
---
title: "Your Post Title"
slug: "your-post-slug"
date: "2026-03-12"
description: "A short description of your post."
tags: ["tag1", "tag2"]
---

Your content here...
```

## Deployment

Pushes to `main` auto-deploy to GitHub Pages via the workflow in `.github/workflows/deploy.yml`.

**Setup:** In your GitHub repo, go to Settings → Pages → Source → select "GitHub Actions".

Live at: https://fearyj.github.io/ken-verbose/
