# Architecture Reference — Index

> Read the section relevant to your task. Don't load everything.

## Sub-Documents

| Document | Covers | Read when... |
|----------|--------|-------------|
| [site-structure.md](site-structure.md) | URL hierarchy, content types, city-first routing, file/folder layout, content schema | Working on routing, adding pages, changing URL patterns |
| [templates-css.md](templates-css.md) | Page templates, HTML structure, CSS architecture, fonts, colours, schema markup | Editing layouts, styling, fixing visual issues |
| [integrations.md](integrations.md) | Viator API, GYG widgets, affiliate revenue, destination IDs | Working on tours, booking widgets, API calls |
| [admin-content.md](admin-content.md) | Admin panel, AI generation, versioning, PAA research, prompt builder | Working on admin features, content generation |
| [deployment-seo.md](deployment-seo.md) | Cloudflare Workers, deployment, SEO meta tags, sitemap, robots.txt, analytics, performance | Deployment issues, SEO changes, performance work |
| [staycations.md](staycations.md) | Staycation data model, images, admin panel, AI import | Working on staycation pages or admin |

### D1 Migration (new)

| Document | Covers | Read when... |
|----------|--------|-------------|
| [d1-schema.md](d1-schema.md) | D1 binding, full SQL schema (8 tables), frontmatter JSON structure, migrations | Setting up D1, writing queries, adding tables |
| [content-pipeline.md](content-pipeline.md) | 6-stage pipeline (Ideate-Write-Humanize-Detect-Review-Publish), CLI commands, config | Building pipeline stages, debugging runs, adding CLI commands |
| [ideas-style.md](ideas-style.md) | Idea sources, approval flow, seasonal calendar, style references, guide extraction | Building idea sources, style UI, or integrating style guides |
| [migration-plan.md](migration-plan.md) | 5 migration phases, what stays in GitHub, env vars, dependencies, file structure | Planning migration PRs, understanding the rollout sequence |

## Core Principle

**Everything in the HTML. No client-side rendering. Every page is a complete, self-contained HTML document.**

```
Markdown/JSON  ──→  Astro Build  ──→  HTML files (Cloudflare Workers)
```

Why: fastest load, near-perfect PageSpeed, free hosting, instant indexing by Google and AI crawlers.

## Banned Phrases

Filter from all machine-generated content. These signal AI authorship:

```
"in conclusion", "it's worth noting", "delve into", "vibrant tapestry",
"bustling metropolis", "hidden gem", "kaleidoscope of", "rich tapestry",
"feast for the senses", "a must-visit", "nestled in",
"whether you're a ... or a ...", "embark on a journey", "immerse yourself",
"plethora of", "myriad of", "a testament to", "unparalleled",
"breathtaking", "awe-inspiring", "unforgettable experience",
"perfect blend of", "seamlessly blends", "caters to every taste"
```

Replace with specific, concrete language. Instead of "a hidden gem nestled in the heart of Old Delhi" → "a 40-seat restaurant on the second floor above a spice shop in Chandni Chowk."

## Pre-Launch Checklist

- [ ] DNS + Cloudflare Workers connected to GitHub org
- [ ] HTTPS, robots.txt, llms.txt deployed
- [ ] XML sitemap generating, submitted to Search Console
- [ ] Analytics installed (Cloudflare Web Analytics minimum)
- [ ] Favicon and og:image set, 404 page created
- [ ] Schema validated with Google Rich Results Test
- [ ] PageSpeed 90+ on mobile
- [ ] No broken internal links
- [ ] Content validation running pre-commit
