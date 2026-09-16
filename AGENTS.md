# AGENTS.md

Use this file when adding or editing blog posts in this repo.

## Blog Rules

- Stack: Jekyll + `jekyll-theme-chirpy`
- Posts live in `_posts/`
- Prefer file names like `YYYY-MM-DD-slug.md`
- Keep the file date and front matter date aligned
- Do not use a future date unless the post is intentionally scheduled

## Front Matter

Use this shape for normal posts:

```yaml
---
layout: post
title: "Post Title"
date: 2026-03-27 10:00:00 -0700
categories: observability
description: "One-sentence summary."
tags: [observability, monitoring]
comments: false
---
```

Notes:

- `last_modified_at` is optional
- `toc` is already enabled by config, so headings matter

## Writing Pattern

- Start with `## Executive Summary`
- Lead with the answer, not suspense
- Use clear `##` sections
- Keep a simple arc: context -> distinction -> implications -> recommendation
- Keep tone practical and direct
- Use precise claims and avoid hedging
- Prefer concise descriptions and useful tags
- Keep repetition low and make every section earn its place
- Put images under `assets/img/` and reference them as `/assets/img/...`

## Before Finishing

- Run `bundle exec jekyll build`
- Do not change `_config.yml` for a normal post
- Leave unrelated files alone
