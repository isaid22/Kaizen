---
title: "Setting Up This Site"
date: 2026-09-05T09:15:00-05:00
draft: false
tags: ["meta", "hugo"]
summary: "Quick note on the stack behind this site."
---

Stack: **Hugo** (static site generator) + **PaperMod** (theme) + **Cloudflare Pages** (free hosting, auto-deploys on git push).

Workflow for new content:

```bash
# a longer piece
hugo new content posts/some-title.md

# a quick note
hugo new content notes/some-title.md

# preview locally before pushing
hugo server -D
```

Once pushed to GitHub, Cloudflare Pages picks up the commit and rebuilds the live site within a minute or two.
