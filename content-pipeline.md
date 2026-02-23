# Content Pipeline

> Reference for implementing the 6-stage content pipeline.
> Read this when: building pipeline stages, adding CLI commands, or debugging pipeline runs.

**The pipeline is TypeScript (NOT Python).** The original spec referenced Python but we decided on TypeScript for consistency with the rest of the codebase.

---

## Overview

```
+------------------------------------------------------------------+
|  1 Ideate -> 2 Write -> 3 Humanize -> 4 Detect -> 5 Review -> 6 Publish  |
|                 ^           ^                        |           |
|                 |           +-- retry if >20% AI ----+           |
|                 +-- revision if review fails ------------------------+
+------------------------------------------------------------------+
```

The pipeline is a single TypeScript CLI (`pipeline/src/cli.ts`) that processes ideas from the `ideas` table. No orchestration framework. Sequential API calls with conditionals.

---

## Stage 1 -- Ideate

Three automated sources + manual input. Ideas always require approval before entering the pipeline. The pipeline queries:

```sql
SELECT * FROM ideas WHERE status = 'approved' ORDER BY priority ASC, created_at ASC;
```

Automated sources are covered in [ideas-style.md](ideas-style.md). This stage selects the next approved idea and creates a `pipeline_runs` record.

---

## Stage 2 -- Write

Claude API call. System prompt includes:

1. Page template structure (based on page type: paa, category, hub, blog)
2. City facts: `SELECT fact FROM city_facts WHERE city = ?`
3. Active style guide: `SELECT guide FROM style_guides WHERE is_active = 1 AND JSON_EXTRACT(page_types, ...) matches`
4. One reference article matching page type: `SELECT body FROM style_references WHERE page_type = ? LIMIT 1`
5. Banned phrases list (from `pipeline/config/banned_phrases.json`)
6. URL this page will live at (for correct internal links)
7. Related pages from the idea (for the 3+ internal links requirement)
8. Tone from style guide (replaces the old `TONE_PROMPTS` single-line descriptors)

User prompt includes:
- Title, target keyword, PAA questions to answer
- Word count target (see Configuration below)
- Specific instructions from the idea's `metadata` JSON

Output: markdown body + JSON frontmatter (title, meta_description, faq array, relatedPages, schema types).

### Write Validation

All of these must pass or the generation is retried once with specific feedback:

- Title <= 60 characters
- Meta description 120-155 characters
- At least 3 internal links in body
- Minimum word count met
- No banned phrases
- FAQ answers present for PAA pages

On success: insert into `pages` table with `status: 'draft'`, create a `page_versions` entry with `source: 'ai'`.

---

## Stage 3 -- Humanize

Second Claude API call. System prompt is the full humanizer SKILL.md content (from https://github.com/blader/humanizer). Store this at `pipeline/prompts/humanizer_system.md`.

The 24 patterns to fix:

1. Significance inflation
2. Notability name-dropping
3. Superficial -ing analyses
4. Promotional language
5. Vague attributions
6. Formulaic challenges
7. AI vocabulary (Additionally, testament, landscape, showcasing)
8. Copula avoidance (serves as, features, boasts)
9. Negative parallelisms (It's not just X, it's Y)
10. Rule of three
11. Synonym cycling
12. False ranges
13. Em dash overuse
14. Boldface overuse
15. Inline-header lists
16. Title Case Headings
17. Emojis
18. Curly quotes
19. Chatbot artifacts
20. Cutoff disclaimers
21. Sycophantic tone
22. Filler phrases
23. Excessive hedging
24. Generic conclusions

Input: the draft markdown from stage 2. Output: rewritten markdown.

On success: create a `page_versions` entry with `source: 'humanizer'`.

Do NOT summarize or abbreviate the SKILL.md content in the prompt. Use it in full -- it contains specific before/after examples for all 24 patterns.

---

## Stage 4 -- Detect (Pangram)

API call to Pangram to score the humanized content:

```
POST https://api.pangram.com/v1/detect
{
  "text": "<humanized markdown, stripped of frontmatter>"
}
```

Store the score:
```sql
UPDATE pages SET pangram_score = ? WHERE id = ?;
UPDATE page_versions SET pangram_score = ? WHERE id = ?;
```

### Decision Logic

| Pangram Score | Action |
|--------------|--------|
| < 20% AI | Pass. Continue to review. |
| 20-40% AI | Loop back to stage 3 (humanize again with Pangram's flagged phrases as additional context). Max 2 retries. |
| > 40% after retries | Set `status: 'needs_review'` on `pipeline_runs`. Skip to manual queue. |
| Pangram API failure | Log error, continue to review. Do not block on detection. |

---

## Stage 5 -- Review

Third Claude API call with editorial reviewer system prompt. The reviewer checks:

1. **Frontmatter**: title <= 60 chars, description 120-155 chars, all required fields present
2. **SEO**: keyword in first paragraph, keyword in H1, clear heading hierarchy (h1 -> h2 -> h3, no skipping)
3. **Internal links**: minimum 3, all resolve to valid pages (provide list of existing slugs)
4. **AIO**: direct answer in first 50 words (PAA pages only), clear heading hierarchy
5. **Schema**: correct types for page type (PAA=FAQPage+Article+BreadcrumbList, Category=Article+BreadcrumbList)
6. **City facts**: verify facts from `city_facts` table appear correctly in content
7. **Banned phrases**: final sweep for any remaining AI-giveaway language
8. **Readability**: no walls of text, paragraphs under 4 sentences, varied sentence length
9. **Tone**: matches the active style guide

Returns JSON:

```json
{
  "pass": true,
  "score": 87,
  "issues": [
    {
      "severity": "blocker | warning | suggestion",
      "category": "seo | content | links | tone | frontmatter",
      "description": "what's wrong",
      "suggestion": "how to fix it",
      "location": "optional -- which section or line"
    }
  ]
}
```

### Decision Logic

| Result | Action |
|--------|--------|
| Pass with no blockers | Continue to publish. |
| Blockers present | Feed issues back to stage 2 (writer) for one revision, then re-run stages 3-5. |
| More than 2 revision loops | Set `status: 'needs_review'` on `pipeline_runs`. Flag for manual editing. |

If revisions were made, create a `page_versions` entry with `source: 'review'`.

---

## Stage 6 -- Publish

```sql
UPDATE pages
SET status = 'published',
    published_at = CURRENT_TIMESTAMP,
    updated_at = CURRENT_TIMESTAMP
WHERE id = ?;

UPDATE ideas SET status = 'published', page_id = ? WHERE id = ?;

UPDATE pipeline_runs
SET status = 'completed',
    completed_at = CURRENT_TIMESTAMP
WHERE id = ?;
```

Content is live instantly -- Astro reads from D1 at request time. No rebuild needed.

Optionally ping Search Console API to request recrawl of the new URL.

---

## Pipeline Run Tracking

The `stages` JSON column on `pipeline_runs` tracks each stage:

```json
{
  "ideate": { "completed_at": "...", "idea_id": "..." },
  "write": { "completed_at": "...", "version_id": "...", "word_count": 1200, "model": "claude-sonnet-4-20250514" },
  "humanize": { "completed_at": "...", "version_id": "...", "retries": 0 },
  "detect": { "completed_at": "...", "pangram_score": 0.12, "retries": 1 },
  "review": { "completed_at": "...", "score": 87, "issues_count": 2, "blockers": 0 },
  "publish": { "completed_at": "...", "url": "/delhi/best-biryani/" }
}
```

Each stage writes its entry on completion. If a stage fails, the `error` column on `pipeline_runs` is populated and `status` is set to `'failed'`.

---

## CLI Interface

Entry point: `pipeline/src/cli.ts`

### Commands

```bash
# Run full pipeline on approved ideas
npx tsx pipeline/src/cli.ts batch

# Run pipeline for a single page
npx tsx pipeline/src/cli.ts single --city delhi --slug best-biryani --type paa --title "Best Biryani in Delhi"

# Run ideation only (populate ideas table)
npx tsx pipeline/src/cli.ts ideate --source all            # all sources
npx tsx pipeline/src/cli.ts ideate --source gaps            # content gaps only
npx tsx pipeline/src/cli.ts ideate --source claude          # Claude creative ideas only
npx tsx pipeline/src/cli.ts ideate --source search_console  # GSC mining only

# Refresh stale content (pages with updated_at > 6 months and status=published)
npx tsx pipeline/src/cli.ts refresh --max 10

# Rerun a failed pipeline stage
npx tsx pipeline/src/cli.ts retry --run-id <pipeline_run_id>

# Migrate existing markdown files to D1 (one-time)
npx tsx pipeline/src/cli.ts migrate --dry-run     # preview what would be migrated
npx tsx pipeline/src/cli.ts migrate --execute      # actually run migration

# Extract style guide from references
npx tsx pipeline/src/cli.ts style-extract --page-type blog

# Score existing content with Pangram (backfill)
npx tsx pipeline/src/cli.ts score --city delhi --limit 20
```

---

## Configuration

`pipeline/src/config.ts`:

```typescript
// Pipeline defaults
export const MAX_HUMANIZER_RETRIES = 2;
export const MAX_REVIEW_RETRIES = 1;
export const PANGRAM_THRESHOLD = 0.20;        // Max AI score to pass
export const PANGRAM_HARD_FAIL = 0.40;        // Score above this -> needs manual review

// Word count targets by page type
export const WORD_COUNTS: Record<string, number> = {
  paa: 600,
  category: 1500,
  hub: 2500,
  blog: 1200,
};

// Claude model
export const MODEL = 'claude-sonnet-4-20250514';

// API base URLs
export const D1_API_URL = process.env.D1_API_URL;
export const PANGRAM_API_URL = 'https://api.pangram.com/v1';
```

### Environment Variables (pipeline `.env`)

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Claude API for write/humanize/review/ideation agents |
| `PANGRAM_API_KEY` | Pangram AI detection API |
| `D1_API_URL` | D1 API base URL |
| `D1_API_KEY` | D1 API auth |
| `GOOGLE_SEARCH_CONSOLE_KEY` | Optional -- for Search Console idea mining |
