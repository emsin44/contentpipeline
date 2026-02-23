# Site Structure

## URL Hierarchy

All content is **city-first**. No separate `/experiences/` tree.

```
/                                    → Homepage
/{city}/                             → City hub (overview + links to sub-pages)
/{city}/food-tours/                  → Category page (Viator listings)
/{city}/heritage-walks/              → Category page
/{city}/best-street-food/            → PAA page + contextual tours
/{city}/is-delhi-safe/               → PAA page (info only)
/staycations/                        → Staycations index
/staycations/{slug}/                 → Individual staycation
/blog/                               → Blog index
/blog/{slug}/                        → Blog post
/food-tours-india/                   → Cross-city category hub
/destinations/                       → All destinations
/about/                              → About page
/contact/                            → Contact page
/journal/                            → Journal index
```

### URL Rules

- All lowercase, hyphens between words (never underscores)
- No dates in URLs (content is evergreen)
- Trailing slashes always (configured in `astro.config.mjs`)
- City slug is always the first path segment for city content
- Max 3 levels deep: `/{city}/{slug}/`
- Every page has a canonical tag pointing to itself

## Page Types

| Type | Example | Content | Tours | Layout |
|------|---------|---------|-------|--------|
| **Hub** | `/delhi/` | Overview + links to all sub-pages | Featured (6) | `[city].astro` |
| **Category** | `/delhi/food-tours/` | Experience intro + Viator listings | Yes (6, filtered by tag) | `CategoryPage.astro` |
| **PAA** | `/delhi/best-street-food/` | Answer to specific question | Contextual (2–4) | `PAAPage.astro` |
| **PAA** | `/delhi/is-delhi-safe/` | Answer (no tours relevant) | None | `PAAPage.astro` |

## Routing

Content is handled by `pages/[...slug].astro` (catch-all route):
- Content files at `content/{city}/{slug}.md` become `/{city}/{slug}/`
- Route detects page type from frontmatter (`type: category` or `type: paa`)
- Renders appropriate layout

City hub pages: `pages/[city].astro`

## Content Frontmatter

**Category page** (`food-tours.md`):
```yaml
title: "Best Food Tours In Delhi — 2026"
type: category
city: delhi
category: food
viatorTagId: 21911  # Optional override
```

**PAA page** (`is-delhi-safe.md`):
```yaml
title: "Is Delhi Safe for Tourists?"
type: paa
city: delhi
category: practical  # Maps to tour type (or none)
```

**Full schema** (TypeScript at `src/content/config.ts`):
```typescript
z.object({
    title: z.string().max(60),
    description: z.string().min(120).max(155),
    city: z.string(),
    category: z.string(),
    type: z.enum(['hub', 'category', 'paa', 'blog']),
    datePublished: z.string(),
    dateModified: z.string(),
    status: z.enum(['machine-draft', 'published', 'human-edited']),
    schema: z.array(z.string()),
    relatedPages: z.array(z.string()),
    parentPage: z.string(),
    faq: z.array(z.object({ question: z.string(), answer: z.string() })).optional()
})
```

## File & Folder Layout

```
india-experiences/
├── src/
│   ├── pages/
│   │   ├── index.astro              # Homepage
│   │   ├── [city].astro             # City hub pages
│   │   ├── [...slug].astro          # Catch-all (category + PAA pages)
│   │   ├── staycations/
│   │   │   ├── index.astro
│   │   │   └── [slug].astro
│   │   ├── journal/
│   │   │   ├── index.astro
│   │   │   └── [slug].astro
│   │   └── api/viator/tours.ts      # Dynamic API route
│   ├── layouts/
│   │   ├── BaseLayout.astro         # Base HTML + GYG script at end of <body>
│   │   ├── CategoryPage.astro
│   │   ├── CityHub.astro
│   │   └── PAAPage.astro
│   ├── components/
│   │   ├── Header.astro
│   │   ├── Footer.astro
│   │   ├── Hero.astro
│   │   └── GygWidget.astro
│   ├── lib/viator.ts                # Viator API client
│   ├── data/                        # JSON data files
│   ├── content/{city}/*.md          # Markdown content
│   └── styles/main.css              # Global CSS + @font-face
├── public/
│   ├── fonts/                       # Self-hosted fonts
│   ├── images/                      # All images
│   ├── robots.txt
│   └── llms.txt
├── scripts/
│   └── seo-validate.mjs
└── wrangler.toml                    # Cloudflare Workers config
```

## Data Files

| File | Controls |
|------|----------|
| `data/cities.json` | City metadata, hero images, homepage order (first 8 shown) |
| `data/staycations.json` | Staycation properties + GYG tour IDs |
| `data/experiences.json` | Experience types + Viator tag IDs (30 types) |
| `data/homepage.json` | Hero, intro, features, featured experience |
| `data/categories.json` | Footer category links |
| `data/about.json` | About page content |
| `data/content/{city}/hub.json` | Generated city hub content + versions |

## City Data Structure

```json
{
  "name": "Delhi",
  "slug": "delhi",
  "tier": 1,
  "lat": "28.6139",
  "lng": "77.2090",
  "description": "India's capital city...",
  "heroImage": "/images/cities/delhi-hero.jpg",
  "cardImage": "/images/cities/delhi-card.jpg",
  "bestMonths": [10, 11, 12, 1, 2, 3],
  "goodMonths": [4, 9]
}
```

- `tier`: metadata only (1=major, 2=secondary) — does NOT affect homepage display
- Homepage shows first 8 cities by array position in `cities.json`
- Content pages inherit city hero image via fallback: `heroImage ?? cityData?.heroImage`

## Content Injection Workflow

```
GENERATE → VALIDATE → COMMIT → BUILD → DEPLOY → INDEX → MONITOR → HUMAN EDIT
```

Validation checks (run via `npm run validate`):
- Title < 60 chars, description 120–155 chars
- Valid city and category slugs
- Minimum 3 internal links, no broken internal links
- FAQ answers match visible content
- No duplicate title or URL
- Minimum word count met
- No AI-giveaway phrases
