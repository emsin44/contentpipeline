# Admin Panel & Content Generation

The admin panel (`admin/`) is a Next.js app at `localhost:3000`. It manages content, staycations, and experiences via GitHub API commits.

## File Structure

```
admin/src/
├── lib/
│   ├── promptBuilder.ts          # Shared prompt construction (generate + expand)
│   ├── github.ts                 # GitHub API client (readJSON, writeJSON)
│   ├── auth.ts
│   └── ai-import/                # Staycation AI import (see staycations.md)
├── app/
│   ├── api/
│   │   ├── content/
│   │   │   ├── route.ts          # CRUD + versioning (GET, POST, PUT)
│   │   │   └── research/route.ts # PAA research, generate, expand
│   │   ├── ai-import/route.ts    # SSE streaming for AI import
│   │   ├── homepage/             # GET/PUT homepage.json
│   │   ├── about/                # GET/PUT about.json
│   │   └── experiences/          # CRUD experiences
│   ├── content/[city]/
│   │   ├── page.tsx              # City hub editor (tabs, state)
│   │   └── components/
│   │       ├── GenerationControls.tsx  # Tone/wordCount/keywords + prompt preview
│   │       ├── PAAResearchPanel.tsx    # PAA research + selection
│   │       ├── MarkdownEditor.tsx      # Content editor with preview
│   │       └── VersionHistory.tsx      # Version list, preview, revert
│   └── content/[city]/[page]/page.tsx  # Sub-page editor
```

## Content Flow

```
Admin Panel → Save → GitHub API commit → Cloudflare rebuild → Live
```

City hub content stored in `india-experiences/src/data/content/{city}/hub.json`.

City page loads it at build time:
```javascript
const hubModules = import.meta.glob('../data/content/*/hub.json', { eager: true });
const hubData = hubModules[`../data/content/${city}/hub.json`];
if (hubData?.default?.generatedContent) {
  hubContent = await marked(hubData.default.generatedContent);
}
```

Fallback: if no hub.json, page uses `cities.json` description.

## Content Versioning

Every change creates a `ContentVersion` (max 20 retained):

```typescript
interface ContentVersion {
  id: string;            // "v_1707123456_abc1234"
  content: string;       // Full markdown
  wordCount: number;
  createdAt: string;     // ISO timestamp
  source: "ai" | "manual";
  generationConfig?: { tone, wordCount, keywords, paaQuestionIds, expandDirection };
  note?: string;
}
```

`currentVersionId` tracks which version is active. Users can preview any version and revert (creates a new version from old content).

## City Facts

Facts override AI knowledge cutoffs. Stored in `hub.json`:

```json
{ "facts": [
    "Goa has TWO airports: Dabolim (GOI) and Manohar International (GOX) at Mopa",
    "Current taxi rates 2026: Dabolim→Calangute ₹1500, Mopa→Calangute ₹900"
] }
```

Facts injected into every generation prompt as "CRITICAL FACTS (verified current information - MUST include these)". Sub-pages inherit parent hub facts automatically.

## Generation Modes

### Generate Content (fresh)
1. Select PAA questions, configure tone/wordCount/keywords
2. Preview prompt (client-side, deterministic)
3. API calls Claude → content appears in editor
4. Auto-published as new AI version

### Expand Content (additive)
1. Enter expansion direction (e.g. "Add section about monsoon festivals")
2. Configure additional word count (200–2000)
3. Claude receives existing content + instructions to merge
4. Returns full merged document (not a diff)
5. Auto-published with note "Expanded: {direction}"

`max_tokens: 16384` for expand to accommodate existing + new content.

### Sub-Page Generation
Route: `admin/content/[city]/[page]`

Three tabs: AI Generate, Editor, Viator config.

API: `POST /api/content/generate` — takes cityName, question, tone, wordCount, facts.

Sub-page content saved to `data/content/{city}/pages/{slug}.json`.

## Prompt Builder

`admin/src/lib/promptBuilder.ts` — single source of truth.

| Export | Purpose |
|--------|---------|
| `buildGeneratePrompt()` | Hub content generation from PAA questions |
| `buildExpandPrompt()` | Expand with existing content between delimiters |
| `TONE_PROMPTS` | Map of tones to style instructions |

**Available tones:** conversational, professional (default), enthusiastic, practical, storytelling

## PAA Research

Tab sends city names to Claude → generates 15–20 "People Also Ask" questions across categories. Persisted to hub's `paaResearch` field.

API: `POST /api/content/research` with `action: "research"` | `"save"` | `"bulk-research"`

## Accuracy Safeguards

Generated content includes:
1. **Uncertainty flags:** `[VERIFY: check current timings]`
2. **Date-stamped prices:** `₹800–1,200 (2026 rates)`
3. **Range-based pricing:** ranges, not specific amounts
4. **Hedging language:** "typically", "usually", "check current rates"
5. **Facts to Verify section:** checklist at end of every article
6. **No invented names:** won't fabricate business names

Editorial workflow: review `[VERIFY: ...]` tags before publishing.

## Homepage Admin

Route: `/homepage` — edits `data/homepage.json`

Editable: hero (location, title), intro paragraphs, 3 features (icon + title + description), featured experience (title, description, link, image).

Available icons: globe, layers, pin, heart, star, compass.

## About Page Admin

Route: `/about` — edits `data/about.json`

Editable: hero section, contact info, dynamic content sections (add/remove/reorder, markdown editor with live preview).

## Experiences Admin

Route: `/experiences` — edits `data/experiences.json`

Two tabs:
1. **Experience Types** — Viator-linked categories (toggle homepage visibility, upload images, edit Viator tag ID)
2. **Curated** — Hand-picked experiences with city targeting, pricing, booking URLs

Image auto-resize: card (400x400), list (800x500), hero (1200x600).
