# D1 Database Schema

> Reference for implementing the Cloudflare D1 database layer.
> Read this when: setting up D1, writing migrations, or querying content tables.

---

## Wrangler Configuration

Add D1 binding to `india-experiences/wrangler.toml`:

```toml
[[d1_databases]]
binding = "DB"
database_name = "content"
database_id = "<created-via-wrangler>"
```

Create the database:

```bash
cd india-experiences
wrangler d1 create content
```

Add the type definition to `india-experiences/src/env.d.ts`:

```typescript
type Runtime = import('@astrojs/cloudflare').Runtime<{
  DB: D1Database;
  VIATOR_API_KEY: string;
  GYG_PARTNER_ID: string;
  D1_API_KEY: string;
}>;

declare namespace App {
  interface Locals extends Runtime {}
}
```

Astro on Cloudflare Workers accesses D1 via `Astro.locals.runtime.env.DB`. This is automatic when the D1 binding is configured in `wrangler.toml`.

---

## Schema (8 Tables)

Create `india-experiences/migrations/0001_initial.sql`:

```sql
-- Content pages (replaces markdown files in content/{city}/*.md)
CREATE TABLE pages (
  id TEXT PRIMARY KEY,
  city TEXT NOT NULL,
  slug TEXT NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('paa', 'category', 'hub', 'blog')),
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'review', 'published', 'archived')),
  title TEXT NOT NULL,
  meta_description TEXT,
  body_markdown TEXT,
  frontmatter JSON NOT NULL DEFAULT '{}',
  pangram_score REAL,
  generation_config JSON,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  published_at DATETIME,
  UNIQUE(city, slug)
);

CREATE INDEX idx_pages_city ON pages(city);
CREATE INDEX idx_pages_status ON pages(status);
CREATE INDEX idx_pages_type ON pages(type);
CREATE INDEX idx_pages_city_status ON pages(city, status);

-- Version history for every content change
CREATE TABLE page_versions (
  id TEXT PRIMARY KEY,
  page_id TEXT NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  version INTEGER NOT NULL,
  body_markdown TEXT NOT NULL,
  source TEXT NOT NULL CHECK (source IN ('ai', 'humanizer', 'review', 'manual', 'pipeline')),
  pangram_score REAL,
  generation_config JSON,
  note TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(page_id, version)
);

CREATE INDEX idx_versions_page ON page_versions(page_id);

-- PAA research data (replaces hub.json paaResearch field)
CREATE TABLE paa_research (
  id TEXT PRIMARY KEY,
  city TEXT NOT NULL,
  question TEXT NOT NULL,
  category TEXT,
  selected BOOLEAN DEFAULT 0,
  researched_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(city, question)
);

CREATE INDEX idx_paa_city ON paa_research(city);

-- Content ideas from all sources
CREATE TABLE ideas (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL CHECK (type IN ('blog', 'paa', 'category')),
  city TEXT,
  title TEXT NOT NULL,
  slug TEXT,
  source TEXT NOT NULL CHECK (source IN ('search_console', 'paa_gap', 'seasonal', 'competitor', 'claude', 'manual')),
  rationale TEXT,
  target_keyword TEXT,
  search_volume INTEGER,
  difficulty_score REAL,
  priority INTEGER DEFAULT 5 CHECK (priority BETWEEN 1 AND 10),
  status TEXT NOT NULL DEFAULT 'idea' CHECK (status IN ('idea', 'approved', 'in_pipeline', 'published', 'rejected')),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  approved_at DATETIME,
  page_id TEXT REFERENCES pages(id),
  metadata JSON DEFAULT '{}'
);

CREATE INDEX idx_ideas_status ON ideas(status);
CREATE INDEX idx_ideas_city ON ideas(city);
CREATE INDEX idx_ideas_source ON ideas(source);

-- Pipeline run tracking
CREATE TABLE pipeline_runs (
  id TEXT PRIMARY KEY,
  page_id TEXT REFERENCES pages(id),
  idea_id TEXT REFERENCES ideas(id),
  status TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'completed', 'failed', 'needs_review')),
  stages JSON NOT NULL DEFAULT '{}',
  started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  completed_at DATETIME,
  error TEXT,
  retry_count INTEGER DEFAULT 0
);

CREATE INDEX idx_pipeline_page ON pipeline_runs(page_id);

-- Style reference articles
CREATE TABLE style_references (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  source_url TEXT,
  body TEXT NOT NULL,
  notes TEXT,
  tags JSON DEFAULT '[]',
  page_type TEXT,
  added_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Distilled style guides (extracted from references)
CREATE TABLE style_guides (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  guide TEXT NOT NULL,
  page_types JSON DEFAULT '[]',
  source_article_ids JSON DEFAULT '[]',
  version INTEGER DEFAULT 1,
  is_active BOOLEAN DEFAULT 1,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- City facts (replaces hub.json facts field)
CREATE TABLE city_facts (
  id TEXT PRIMARY KEY,
  city TEXT NOT NULL,
  fact TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(city, fact)
);

CREATE INDEX idx_facts_city ON city_facts(city);
```

### Table Summary

| Table | Replaces | Purpose |
|-------|----------|---------|
| `pages` | `content/{city}/*.md` files | All content pages (PAA, category, hub, blog) |
| `page_versions` | Hub.json `versions[]` | Full version history for every content change |
| `paa_research` | Hub.json `paaResearch` | PAA question research data per city |
| `ideas` | Nothing (new) | Content ideas from all sources, with approval flow |
| `pipeline_runs` | Nothing (new) | Tracks each pipeline execution through 6 stages |
| `style_references` | Nothing (new) | Reference articles for tone/voice extraction |
| `style_guides` | `TONE_PROMPTS` in admin | Extracted writing style guides per page type |
| `city_facts` | Hub.json `facts[]` | City-specific facts used by the writer agent |

---

## Frontmatter JSON Structure

The `frontmatter` column on `pages` stores what was previously YAML frontmatter as JSON:

```json
{
  "category": "food",
  "schema": ["FAQPage", "Article", "BreadcrumbList"],
  "relatedPages": ["/delhi/", "/delhi/cooking-classes/"],
  "parentPage": "/delhi/",
  "faq": [
    {
      "question": "Is street food safe in Delhi?",
      "answer": "Street food in Delhi is generally safe if you follow a few rules..."
    }
  ],
  "heroImage": "/images/cities/delhi-hero.jpg",
  "datePublished": "2026-02-17",
  "dateModified": "2026-02-17"
}
```

This replaces YAML parsing entirely. No more edge cases with colons in titles or multiline descriptions.

Key fields stored in frontmatter JSON (not as columns):
- `category` -- content category (food, culture, etc.)
- `schema` -- JSON-LD schema types for the page
- `relatedPages` -- internal links for navigation
- `parentPage` -- breadcrumb parent
- `faq` -- question/answer pairs (PAA pages)
- `heroImage` -- hero image path
- `datePublished` / `dateModified` -- SEO dates

Key fields stored as columns (not in frontmatter):
- `title`, `meta_description` -- needed for queries and listings
- `city`, `slug`, `type`, `status` -- needed for routing and filtering
- `pangram_score` -- needed for pipeline decisions
- `body_markdown` -- the actual content

---

## Running Migrations

```bash
# Apply migration to local dev
wrangler d1 execute content --local --file=./migrations/0001_initial.sql

# Apply migration to production
wrangler d1 execute content --file=./migrations/0001_initial.sql

# Run ad-hoc queries
wrangler d1 execute content --command "SELECT count(*) FROM pages"
```

For subsequent migrations, create `0002_*.sql`, `0003_*.sql`, etc. Each migration file should be idempotent where possible (use `IF NOT EXISTS`).
