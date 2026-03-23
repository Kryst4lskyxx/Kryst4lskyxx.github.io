# About Page Design

**Date:** 2026-03-23
**Scope:** `src/content/spec/about.md` + `src/config.ts`

## Goal

Replace the placeholder about page with a real personal introduction. Simultaneously update config fields that are still set to Chinese placeholder values.

## About Page Structure (Option A — Narrative flow)

Single flowing page, four beats:

1. **Opening intro** — Who I am, one punchy sentence.
2. **Professional** — Big data engineering focus: ClickHouse, pipelines, storage systems.
3. **This blog** — What topics to expect: big data, dev tooling, AI coding tools.
4. **Personal + closing** — Gaming (Steam), learning new tech; brief warm close.

## Final Content

```markdown
# About

Hi, I'm kryst4lskyxx — a big data engineer interested in the intersection of large-scale data systems and the tools that make engineering faster and less painful.

I work on big data infrastructure — designing and debugging ingestion pipelines, storage engines, and the systems that sit between raw data and actionable insight. ClickHouse is a recurring focus: its internals, performance characteristics, and how to integrate it effectively into production pipelines.

This blog is where I document what I'm learning and building. Expect write-ups on big data engineering (ClickHouse, storage engines, query optimization), developer tooling, and how AI coding tools are changing the way engineers work.

Outside work, I game on Steam and enjoy picking up new technologies as they emerge. If something on here is useful to you, that's a bonus.
```

## Config Changes (`src/config.ts`)

| Field | Before | After |
|---|---|---|
| `siteConfig.lang` | `zh_CN` | `en` |
| `siteConfig.title` | `"我的博客"` | `"kryst4lskyxx"` |
| `siteConfig.subtitle` | `"用 Astro 和 Fuwari 构建的个人博客"` | `"Big data, dev tooling, and the tech in between"` |
| `profileConfig.bio` | `"To be a good programmer"` | `"Big data engineer. ClickHouse, pipelines, and the tooling that makes it bearable."` |
