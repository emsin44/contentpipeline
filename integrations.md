# Integrations — Viator & GYG

## Viator Tours (City + Staycation Pages)

Affiliate tours fetched at build/request time via Viator Partner API v2.

### Where Used

| Page | What shows |
|------|------------|
| City pages (`/{city}/`) | "More Experiences" — 6 products |
| Category pages | Filtered by Viator tag ID |
| PAA pages | Contextual tours (2–4) based on question category |
| Staycation pages | "Experiences & Tours" section |

### API Config

- **Env:** `VIATOR_API_KEY` (`.env` + Cloudflare dashboard)
- **Access:** `import.meta.env.VIATOR_API_KEY || process.env.VIATOR_API_KEY`
- **Base URL:** `https://api.viator.com/partner`
- **Endpoint:** `POST /products/search`
- **Currency:** `INR`

### viator.ts Module

Located at `india-experiences/src/lib/viator.ts`:

```typescript
export async function searchProducts(params: { destId: number; limit?: number }): Promise<ViatorProduct[]>
export function getDestinationId(city: string): number | undefined
```

**Key behaviour:** If API key is missing or call fails → returns empty array. Section hides. No mock data ever.

### Key Destination IDs

| City | ID | City | ID |
|------|-----|------|-----|
| Delhi | 804 | Jaipur | 4627 |
| Mumbai | 953 | Goa | 4594 |
| Agra | 4547 | Varanasi | 22015 |
| Kolkata | 4924 | Udaipur | 5106 |

Full map of 80 Indian cities in `viator.ts`. IDs refreshed 2026-02-11.

### Image Selection

`selectBestImage` picks the cover image (`isCover: true`) and selects the smallest variant >= 600px wide.

### Category → Viator Tag Mapping

Category pages map content slugs to Viator tag IDs via `data/experiences.json`.

PAA pages use contextual mapping:
```javascript
const PAA_CATEGORY_TOURS = {
  'food': [21911],           // Food & Drink
  'transport': [12044],      // Transfers
  'heritage': [12029, 13030], // Historical + Walking
  'shopping': [12039],
  'culture': [12028],
  'wellness': [21522],       // Yoga & Meditation
};
```

### Affiliate Revenue

- Commission: 8% of booking value
- Cookie: 30 days
- Payment: Monthly via PayPal

---

## GetYourGuide (Staycation Pages Only)

Availability widget — inline booking, user stays on site.

### Where Used

**Staycation pages only.** Not on city pages.

### Config

- **Partner ID:** `OBZX5NA` (env: `GYG_PARTNER_ID`, fallback hardcoded)
- **Widget type:** Availability (calendar with live availability)
- **Tour IDs:** Per-staycation in `staycations.json` → `tours.gygTourIds[]`

### GygWidget.astro

Located at `src/components/GygWidget.astro`:

```astro
<div
  data-gyg-href="https://widget.getyourguide.com/default/availability.frame"
  data-gyg-tour-id={tourId}
  data-gyg-widget="availability"
  data-gyg-partner-id={partnerId}
  data-gyg-currency="INR"
>
```

### GYG Script Loading

Loaded once in `BaseLayout.astro` at **end of `<body>`** (NOT `<head>` — causes blank widgets):

```html
<script async defer src="https://widget.getyourguide.com/dist/pa.umd.production.min.js"
  data-gyg-partner-id="OBZX5NA" is:inline></script>
```

`is:inline` prevents Astro from bundling the external script.

### Rendering in [slug].astro

```astro
{(toursConfig.gygEnabled !== false) && toursConfig.gygTourIds?.length > 0 && (
  toursConfig.gygTourIds.filter(id => id).map(tourId => (
    <GygWidget tourId={tourId} />
  ))
)}
```

### Affiliate Revenue

- Commission: ~8% of booking value
- Attribution via partner ID in widget

---

## Curated Experiences

Hand-picked experiences injected into city pages (separate from Viator/GYG).

### Data Structure

```json
{
  "curated": true,
  "cities": ["delhi", "agra"],
  "price": "₹3,500",
  "duration": "4 hours",
  "bookingUrl": "https://...",
  "priority": 1,
  "active": true
}
```

### Injection

Filtered by city in `[...slug].astro`, rendered in "Featured in {City}" section (dark gradient background, gold badge) above Viator section.

```javascript
const featuredExperiences = curatedExperiences
  .filter(exp => exp.cities?.includes(citySlug))
  .sort((a, b) => (a.priority || 10) - (b.priority || 10))
```

Managed via Experiences admin (`/experiences` → Curated tab).
