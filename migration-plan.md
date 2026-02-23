# D1 Migration Plan

> Reference for planning and executing the migration from file-based content to D1.
> Read this when: working on any migration phase, setting up dependencies, or understanding what moves where.

---

## 5 Phases

### Phase 1 -- Add D1 (Week 1)

No impact on existing site. D1 is available but unused.

1. Create D1 database: `wrangler d1 create content`
2. Add binding to `wrangler.toml`
3. Run schema migration (see [d1-schema.md](d1-schema.md))
4. Add type definitions to `env.d.ts`
5. Deploy -- existing file-based routes still work

### Phase 2 -- Build Pipeline (Week 2-3)

Pipeline writes to D1. Existing site still serves from markdown files.

1. Create `pipeline/` directory with all stage scripts (TypeScript)
2. Implement CLI interface (`pipeline/src/cli.ts`, see [content-pipeline.md](content-pipeline.md))
3. Test full pipeline end-to-end: idea -> published page in D1
4. Verify pages table has correct data

### Phase 3 -- Dual Routing (Week 3-4)

Both systems serve content. New D1 routes alongside existing file routes.

1. Add D1-backed API routes for admin panel (`/api/d1/*`)
2. Add D1-backed page route at a test path (e.g., `/v2/[city]/[slug]`)
3. Compare rendered output between file route and D1 route for same content
4. Fix any rendering differences

### Phase 4 -- Migrate Content (Week 4)

One-time script to move all existing markdown content into D1.

Create `pipeline/src/migrate.ts`:

```typescript
// For each .md file in content/{city}/:
//   1. Parse YAML frontmatter
//   2. Extract body markdown
//   3. INSERT INTO pages (city, slug, type, status, title, meta_description, body_markdown, frontmatter)
//   4. Create initial version in page_versions

// For each hub.json in data/content/{city}/:
//   1. Extract generatedContent -> pages table (type='hub')
//   2. Extract versions[] -> page_versions table
//   3. Extract paaResearch -> paa_research table
//   4. Extract facts[] -> city_facts table
```

Validate: every URL that currently works still works after migration.

### Phase 5 -- Cut Over (Week 5)

Switch primary routes to D1.

1. Update `[...slug].astro` to read from D1
2. Update `[city].astro` to read from D1
3. Replace Astro sitemap with D1-backed sitemap
4. Update admin panel to use D1 API routes instead of GitHub API for content
5. Remove `content/{city}/*.md` files from repo
6. Remove `data/content/{city}/hub.json` files from repo
7. Remove `@astrojs/sitemap` integration
8. Keep Astro content collections config for any non-migrated content types

---

## What Stays in GitHub

These files do NOT move to D1. They remain in the repo as version-controlled reference data and code.

| Item | Reason |
|------|--------|
| Astro templates + layouts (`src/layouts/`, `src/pages/*.astro`) | Code, not content. Changes require deploys. |
| Components (`src/components/`) | Shared UI -- changes rarely |
| CSS + fonts (`src/styles/`, `public/fonts/`) | Design system -- version controlled |
| `data/cities.json` | ~40 cities, changes quarterly at most |
| `data/experiences.json` + `data/categories.json` | Reference data, low change frequency |
| `data/staycations.json` | Keep existing admin -> GitHub flow (low volume, manual) |
| `data/homepage.json` + `data/about.json` | Low change frequency, existing admin flow works |
| Static images (`public/images/`) | Binary assets don't belong in DB |
| `robots.txt`, `llms.txt`, favicons | Config files |
| `astro.config.mjs`, `wrangler.toml`, `package.json` | Infrastructure config |

### What Gets Removed (after Phase 5 cutover)

```
india-experiences/src/content/{city}/*.md       -- All city content markdown files
india-experiences/src/data/content/*/hub.json    -- Hub content + versions + PAA research
india-experiences/src/content/config.ts          -- Astro content collections config (if fully migrated)
```

### What is NOT Removed

```
india-experiences/src/data/cities.json           -- Stays (reference data)
india-experiences/src/data/experiences.json       -- Stays
india-experiences/src/data/categories.json        -- Stays
india-experiences/src/data/staycations.json       -- Stays (separate admin flow)
india-experiences/src/data/homepage.json          -- Stays
india-experiences/src/data/about.json             -- Stays
```

---

## New Environment Variables

### Cloudflare Workers (dashboard + wrangler.toml)

| Variable | Purpose |
|----------|---------|
| D1 binding `DB` | Configured in `wrangler.toml`, not an env var |
| `D1_API_KEY` | Shared secret for admin -> D1 API auth |

### Admin `.env.local`

| Variable | Purpose |
|----------|---------|
| `D1_API_URL` | Base URL for D1 API routes (e.g., `https://indiaesque.in/api/d1`) |
| `D1_API_KEY` | Same shared secret |
| `PANGRAM_API_KEY` | Pangram AI detection API key |

### Pipeline `.env`

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Claude API for write/humanize/review/ideation agents |
| `PANGRAM_API_KEY` | Pangram AI detection API |
| `D1_API_URL` | D1 API base URL |
| `D1_API_KEY` | D1 API auth |
| `GOOGLE_SEARCH_CONSOLE_KEY` | Optional -- for Search Console idea mining |

---

## New Dependencies

### Pipeline (`pipeline/package.json` -- TypeScript)

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "latest",
    "dotenv": "latest",
    "gray-matter": "latest"
  },
  "devDependencies": {
    "tsx": "latest",
    "typescript": "latest",
    "@types/node": "latest"
  }
}
```

| Package | Purpose |
|---------|---------|
| `@anthropic-ai/sdk` | Claude API for write/humanize/review/ideation agents |
| `dotenv` | Env var loading |
| `gray-matter` | For migration script (parsing existing .md YAML frontmatter) |
| `tsx` | TypeScript execution without build step |

### Admin (npm)

No new packages expected. Existing `fetch` handles D1 API calls.

### Astro (npm)

No new packages expected. D1 binding is native to Cloudflare Workers runtime. `marked` is already installed.

---

## File Structure Changes

### New Files

```
india-experiences/
  migrations/
    0001_initial.sql                    -- D1 schema

  src/
    pages/
      api/
        d1/
          pages.ts                      -- CRUD for pages table
          ideas.ts                      -- CRUD for ideas table
          pipeline.ts                   -- Pipeline run queries
          style.ts                      -- Style references + guides
          facts.ts                      -- City facts CRUD
      sitemap-index.xml.ts              -- Dynamic sitemap (replaces plugin)
      sitemap-0.xml.ts                  -- Dynamic sitemap
    env.d.ts                            -- Updated with D1 types

pipeline/
  package.json                          -- TypeScript dependencies
  tsconfig.json                         -- TypeScript config
  src/
    cli.ts                              -- CLI entry point
    config.ts                           -- Pipeline configuration
    stages/
      ideate.ts                         -- Stage 1: idea generation
      write.ts                          -- Stage 2: content generation
      humanize.ts                       -- Stage 3: humanizer pass
      detect.ts                         -- Stage 4: Pangram scoring
      review.ts                         -- Stage 5: editorial review
      publish.ts                        -- Stage 6: D1 write + status update
    sources/
      searchConsole.ts                  -- GSC idea mining
      gaps.ts                           -- Content gap analysis
      claudeIdeas.ts                    -- Claude creative ideation
      seasonal.ts                       -- Calendar-based ideas
    utils/
      d1Client.ts                       -- D1 API wrapper
      pangramClient.ts                  -- Pangram API wrapper
      validation.ts                     -- Content validation checks
    migrate.ts                          -- One-time migration script
  prompts/
    writer_system.md                    -- Writer agent system prompt
    humanizer_system.md                 -- Full humanizer SKILL.md content
    reviewer_system.md                  -- Review agent system prompt
    ideation_system.md                  -- Ideation agent system prompt
    style_extract_system.md             -- Style guide extraction prompt
  config/
    seasonal.json                       -- Seasonal content calendar
    banned_phrases.json                 -- Banned phrase list
  .env                                  -- Pipeline environment variables

admin/src/
  app/
    ideas/
      page.tsx                          -- Ideas management UI
    pipeline/
      page.tsx                          -- Pipeline dashboard UI
    style/
      page.tsx                          -- Style overview
      references/page.tsx               -- Reference articles
      guides/page.tsx                   -- Extracted guides
    api/
      ideas/route.ts                    -- Proxies to D1 API
      pipeline/route.ts                 -- Proxies to D1 API
      style/
        references/route.ts
        guides/route.ts
  lib/
    d1.ts                               -- D1 API client for admin
```

### Modified Files

| File | Change |
|------|--------|
| `india-experiences/wrangler.toml` | Add D1 binding |
| `india-experiences/astro.config.mjs` | Remove `@astrojs/sitemap` integration, remove sitemap exclude patterns |
| `india-experiences/src/pages/[...slug].astro` | Read from D1 instead of content collections |
| `india-experiences/src/pages/[city].astro` | Read hub + sub-pages from D1 instead of hub.json |
| `india-experiences/src/pages/blog/index.astro` | Read blog posts from D1 |
| `india-experiences/src/pages/blog/[slug].astro` | Read blog post from D1 |
| `india-experiences/src/env.d.ts` | Add D1 type definitions |
| `admin/src/lib/promptBuilder.ts` | Replace `TONE_PROMPTS` with style guide loading |
| `admin/src/app/content/[city]/page.tsx` | Switch from GitHub to D1 API |
| `admin/src/app/content/[city]/[page]/page.tsx` | Switch from GitHub to D1 API |

---

## Pangram API Details

- **Endpoint**: `https://api.pangram.com/v1/detect` (verify current docs at pangram.com)
- **Auth**: API key in header
- **Free tier**: 5 checks/day (sufficient for testing)
- **Production**: Paid plan needed for batch generation -- check current pricing at pangram.com/pricing

```typescript
const response = await fetch('https://api.pangram.com/v1/detect', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${process.env.PANGRAM_API_KEY}`,
  },
  body: JSON.stringify({ text: markdownContent }),
});
```
