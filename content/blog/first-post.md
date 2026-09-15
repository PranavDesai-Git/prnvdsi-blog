---
title: "My First Terminal Post"
date: 2026-09-15T17:30:00+05:30
tags: ["terminal", "minimal", "ssg"]
---

### Why I chose this stack

I wanted a blog that I could write from the comfort of my terminal.
No Javascript, no heavy frameworks, just pure markdown and HTML.

**Features of this setup:**
1. Zero javascript (0 bytes)
2. Perfect rendering in `w3m` and `lynx`
3. Instantly deployed to Vercel via Git
4. Fast enough to compile 10,000 posts in milliseconds

Here's an example of some code, which renders perfectly as text:

```bash
git add .
git commit -m "New post"
git push
```
