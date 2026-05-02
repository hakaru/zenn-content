# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A [Zenn](https://zenn.dev) content repository. Articles and books are written in Markdown with YAML frontmatter and published by pushing to GitHub (Zenn is connected to this repo).

## Content structure

- `articles/` — individual articles, one `.md` file each. Filename becomes the article slug.
- `books/<slug>/` — book directories, each with a `config.yaml` and one `.md` per chapter.
- `qiita-draft-*.md` — Qiita drafts kept at the root (not published to Zenn).

## Article frontmatter

```yaml
---
title: "記事タイトル"
emoji: "🎹"          # single emoji, shown as article thumbnail
type: "tech"          # "tech" (技術記事) or "idea" (アイデア)
topics: ["swift", "ios"]  # 1–5 tags
published: true       # false = draft (not visible on Zenn)
---
```

## Book structure

`books/<slug>/config.yaml`:
```yaml
title: "..."
summary: "..."
topics: [...]
published: true
price: 0
chapters:
  - introduction     # filenames without .md, in display order
  - chapter-two
```

Each chapter file has frontmatter with just `title`.

## Zenn CLI (if needed)

Zenn CLI is not installed in this repo (no `package.json`). Use `npx zenn` directly or install globally with `npm install -g zenn-cli`. Key commands:

```bash
npx zenn new:article --slug <slug> --title "..." --type tech
npx zenn new:book --slug <slug>
npx zenn preview    # local preview at http://localhost:8000
```

## Publishing

Merging/pushing to `main` triggers Zenn to pick up changes automatically. Setting `published: true` makes an article or book live; `published: false` keeps it as a draft visible only to the author.

## Content conventions

- Primary language is Japanese. English articles exist (e.g. `m2dx-testflight-beta-release-en.md`) for international audiences.
- Article slugs follow the pattern `<product>-<topic>-<descriptor>` (e.g. `m2dx-core-dx7-fm-synth-swift`).
- Topics covered: Swift/iOS development, audio/DSP (DX7/FM synthesis, CoreAudio), MIDI 2.0, SwiftUI, indie app development.

## Writing style

When writing or editing articles, use the `hakaru-voice` skill to match the author's established voice: casual-but-technical Japanese, conclusion-first structure, italics for internal reactions, and specific numbers over vague claims. Reference article: `articles/m2dx-local-llm-audit-zero-true-positives.md`.
