# Templates, CSS & Schema

## Base HTML Template

Every page uses `BaseLayout.astro`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{title} | Indiaesque</title>  <!-- Dedup: only appends if not already present -->
    <meta name="description" content="{meta_description}">
    <link rel="canonical" href="{canonical_url}">
    <!-- OG + Twitter tags -->
    <!-- JSON-LD schema -->
    <link rel="stylesheet" href="/styles/main.css">
</head>
<body>
    <header><!-- Logo + nav --></header>
    <main>
        <!-- Breadcrumbs (schema-enabled) -->
        <!-- Page content -->
    </main>
    <footer><!-- Data-driven: cities tier 1 + categories --></footer>
    <!-- GYG script loaded here (NOT in <head>) -->
</body>
</html>
```

## Page Layouts

| Page Type | Layout | Wrapper Class |
|-----------|--------|---------------|
| City hub (main) | `pages/[city].astro` | `.hub-content-inner` |
| City hub (sub) | `layouts/CityHub.astro` | `.article-content` |
| Category | `layouts/CategoryPage.astro` | `.article-content` |
| PAA article | `layouts/PAAPage.astro` | `.article-content` |
| Journal | `pages/journal/[slug].astro` | `.prose` |

## SEO: Single H1 Strategy

Hero titles are decorative — they use `<span>` with ARIA:
```html
<span class="hero-title" role="heading" aria-level="1">DELHI</span>
<h1>Things To Do In Delhi</h1>  <!-- This is the real H1 for SEO -->
```

Files using this pattern: all layouts, `[city].astro`, `about.astro`, `contact.astro`, `index.astro`, `destinations/index.astro`, `staycations/index.astro`, `staycations/[slug].astro`, `journal/index.astro`, `journal/[slug].astro`, `Hero.astro`.

## Breadcrumbs

Every page except homepage. HTML + schema markup:
```html
<nav aria-label="Breadcrumb">
    <ol itemscope itemtype="https://schema.org/BreadcrumbList">
        <li itemprop="itemListElement" itemscope itemtype="https://schema.org/ListItem">
            <a itemprop="item" href="/"><span itemprop="name">Home</span></a>
            <meta itemprop="position" content="1">
        </li>
        <!-- ... -->
    </ol>
</nav>
```

## CSS Architecture

### Fonts (Self-Hosted)

| Font | Role | Weights | CSS Variable |
|------|------|---------|--------------|
| **Muli** | Body text | 300, 400, 600, 700 | `--font-body` |
| **Oswald** | Headings | 300, 400, 500, 700 | `--font-heading` |
| **Benton Sans** | Accent text | 300, 400, 500, 700 | `--font-accent` |

Stored at `/public/fonts/{family}/`. **Never use unloaded fonts** (Inter, Playfair Display, etc.).

### Header Logo

"INDIA ESQUE" uses Benton Sans: 26px mobile / 32px desktop, weight 300, letter-spacing 0.3em, colour `#E9CDB0`.

### Global Reset

`src/styles/main.css` resets all margins and list styles. **All margins must be explicitly set** in component styles.

### Scoped Styles + `:global()`

**Critical:** Astro scoped styles don't reach markdown-rendered children (`<Content />`, `<HubContent />`).

```css
/* WRONG — scoped styles won't match markdown <p> tags */
.article-content p { margin: 0 0 20px; }

/* CORRECT — :global() bypasses scoping on child */
.article-content :global(p) { margin: 0 0 20px; }
```

All layouts must provide `:global()` styles for: `p`, `h2`, `h3`, `h4`, `ul`, `ol`, `li`, `strong`, `a`, `blockquote`.

### Colour Palette

| Token | Hex | Usage |
|-------|-----|-------|
| `--gold` | `#E9CDB0` | CTA buttons, accent, logo |
| `--gray-100` | `#f3f4f6` | Light backgrounds |
| `--gray-200` | `#e5e7eb` | Borders |
| `--gray-500` | `#6b7280` | Secondary text |
| `--gray-600` | `#4b5563` | Body text |
| `--gray-900` | `#111827` | Headings, primary text |
| Teal | `#0d9488` | Links (hub pages) |
| Coral | `#e07060` | Links (sub-page layouts) |
| Dark teal | `#2a9d8f` | Blockquote borders, best-time bars |

### Horizontal Scroll Pattern (Homepage)

Used by "Top Cities" and "Staycations" carousels:

```css
.wrapper { position: relative; display: flex; align-items: center; }
/* NEVER add overflow:hidden to wrapper — breaks scrollWidth */

.scroll { display: flex; gap: 16px; overflow-x: auto; flex: 1; min-width: 0;
           scrollbar-width: none; scroll-behavior: smooth; }

.card { flex: 0 0 auto; width: 500px; }
```

## Schema Markup (JSON-LD)

All schema in `<head>` as JSON-LD (not microdata, not RDFa).

### Schema Types by Page

| Page Type | Schema |
|-----------|--------|
| Every page | `BreadcrumbList` + `Article` |
| Pages with FAQ | + `FAQPage` |
| City hub | + `TouristDestination` |

Multiple schemas combined in an array:
```json
[
    { "@type": "Article", ... },
    { "@type": "FAQPage", ... },
    { "@type": "BreadcrumbList", ... }
]
```

### Schema Rules

- FAQ answer text **must match visible page content** — Google penalises mismatches
- `dateModified` must update when content refreshes
- Never use schema for content not visible on page (cloaking)
- All URLs use `https://indiaesque.in`
- Validate: https://search.google.com/test/rich-results

### Homepage Data Sources

| Section | Data File | Logic |
|---------|-----------|-------|
| Hero & Intro | `homepage.json` | Editable via admin |
| Features (3 icons) | `homepage.json` | Available icons: globe, layers, pin, heart, star, compass |
| Featured Experience | `homepage.json` | `featured` object |
| Top Cities | `cities.json` | First 8 by array position |
| Staycations | `staycations.json` | All |
| Experience Types | `experiences.json` | All |

## Footer

Data-driven — links auto-update when data changes.

| Section | Source | Filter |
|---------|--------|--------|
| Destinations | `cities.json` | `tier === 1` |
| Collections | `categories.json` | All, linked via `crossCitySlug` |

Typography: Oswald 12px (headings), Muli 14px (links, #6b7280, hover: #2a9d8f).
