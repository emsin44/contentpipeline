# Ideas System & Style References

> Reference for implementing the ideas pipeline and style guide system.
> Read this when: building idea sources, approval UI, style reference management, or guide extraction.

---

## Ideas System

### Idea Sources

Ideas come from 5 sources. Each inserts into the `ideas` table with its `source` value.

#### 1. Search Console Mining (`source: 'search_console'`)

File: `pipeline/src/sources/searchConsole.ts`

- Authenticate via Google Search Console API (service account)
- Pull queries for last 28 days with impressions > 100
- For each query, check if a matching page exists: `SELECT id FROM pages WHERE slug LIKE ? OR title LIKE ?`
- Group related queries (e.g., "rajasthan monsoon" + "rajasthan in august" + "rajasthan rainy season")
- For each group, insert one idea with the combined impressions in `metadata`
- Runs as a weekly cron

#### 2. Content Gap Analysis (`source: 'paa_gap'`)

File: `pipeline/src/sources/gaps.ts`

- Load city list from `cities.json`
- Load category list from `categories.json` or `experiences.json`
- Query existing pages: `SELECT city, type, slug FROM pages`
- Compare against full city x category matrix, identify missing combinations
- Check `updated_at` for pages older than 6 months -> insert refresh ideas
- Check seasonal calendar (see below) for upcoming topics
- Runs as a weekly cron

#### 3. Seasonal Calendar (`source: 'seasonal'`)

Config file: `pipeline/config/seasonal.json`

```json
[
  { "month": 3, "weeks_before": 6, "topics": ["monsoon travel tips", "kerala monsoon guide", "goa monsoon"], "type": "blog" },
  { "month": 10, "weeks_before": 8, "topics": ["diwali travel india", "diwali in jaipur", "diwali in varanasi"], "type": "blog" },
  { "month": 12, "weeks_before": 6, "topics": ["winter destinations india", "rajasthan in winter", "goa december"], "type": "blog" },
  { "month": 1, "weeks_before": 4, "topics": ["republic day delhi", "january travel india"], "type": "paa" }
]
```

The gap analysis source checks this calendar and generates ideas `weeks_before` weeks ahead of the target month, so content is published in time for seasonal search demand.

#### 4. Claude Ideation (`source: 'claude'`)

File: `pipeline/src/sources/claudeIdeas.ts`

- Fetch existing slugs: `SELECT city, slug, type FROM pages`
- Call Claude API with ideation prompt containing:
  - List of existing page slugs
  - Top 20 pages by performance (from Search Console data if available)
  - Current month + upcoming festivals/weather context
  - Site voice description
- Returns JSON array of ideas
- Insert each idea into `ideas` table
- Runs on-demand or scheduled

#### 5. Manual (`source: 'manual'`)

Via admin panel. User fills in: title, type, city (optional), target keyword, notes. Inserted with `source: 'manual'`.

### Approval Flow

Ideas always require manual approval before the pipeline processes them. The pipeline only picks up `status = 'approved'` ideas.

```
idea -> [approved] -> in_pipeline -> published
                  \-> [rejected]
```

Admin UI shows ideas grouped by source with approve/reject/edit actions. Bulk approve supported.

The pipeline queries for work:

```sql
SELECT * FROM ideas
WHERE status = 'approved'
ORDER BY priority ASC, created_at ASC;
```

When a pipeline run starts for an idea, its status changes to `'in_pipeline'`. On successful publish, it changes to `'published'` with the `page_id` linked.

---

## Style Reference System

### Adding References

Two methods:

**Paste text** -- User pastes article text directly into admin UI. Stored in `style_references` table with title, optional source URL, and tags.

**Fetch URL** -- Use Jina Reader (already integrated for staycation import) to fetch and extract article text from a URL. Same approach as `ai-import/jina-fetcher.ts`:

```
GET https://r.jina.ai/{article_url}
```

Clean the response, store in `style_references`.

### Tagging

Each reference article gets:

- `page_type`: which template type it is a reference for (`blog`, `paa`, `category`, `hub`)
- `tags`: additional labels like `["opening", "food-content", "practical-guide"]`
- `notes`: free text -- "love the conversational opening", "good example of specific pricing", etc.

### Style Guide Extraction

When user clicks "Extract Style Guide" in admin:

1. Fetch all `style_references` for the selected page type(s)
2. Call Claude API with extraction prompt:

```
Analyze these reference articles and extract a comprehensive writing style guide.

Cover:
- VOICE: Level of formality, use of first person, how the author addresses the reader
- OPENINGS: How articles begin -- scene-setting, direct fact, question, anecdote?
- SENTENCE STRUCTURE: Average length, rhythm, variety. Short punchy sentences vs flowing prose.
- PARAGRAPH STRUCTURE: Length, how transitions work between paragraphs
- SPECIFICITY: How prices, distances, times, names are used. Level of concrete detail.
- VOCABULARY: British vs American spelling. Technical vs accessible. Recurring word choices.
- WHAT'S AVOIDED: Cliches, filler phrases, hedging patterns. What this voice never does.
- SECTION FLOW: How the article moves between topics. Use of subheadings.
- TONE MARKERS: 3-5 short example phrases that capture the voice, pulled directly from the articles.

Output a style guide of approximately 400-600 words that another writer could follow to match this voice.
Do not mention the reference articles by name. Write the guide as universal instructions.
```

3. Store result in `style_guides` table with `is_active = 1`
4. Link the source article IDs in `source_article_ids` for re-extraction later

### Multiple Style Guides

Different page types can have different guides:

| Page Type | Guide Character |
|-----------|----------------|
| `blog` | More personal, narrative, first-person allowed |
| `paa` | Direct, factual, answer-first |
| `category` | Authoritative, comparison-focused |

The writer agent selects the guide matching the page type being generated:

```sql
SELECT guide FROM style_guides
WHERE is_active = 1
AND JSON_EXTRACT(page_types, '$') LIKE ?
ORDER BY updated_at DESC
LIMIT 1;
```

### Replaces TONE_PROMPTS

The existing `TONE_PROMPTS` map in `admin/src/lib/promptBuilder.ts` (conversational, professional, enthusiastic, practical, storytelling) is replaced by the style guides system. Each style guide is a full extracted voice description instead of a single-sentence tone descriptor.

The admin generation UI dropdown changes from tone names to style guide names. Fallback: if no style guide exists for a page type, use a hardcoded default guide.
