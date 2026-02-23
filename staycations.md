# Staycations System

Boutique hotel recommendations with rich content, managed via admin panel.

## URLs

```
/staycations/                    → Index
/staycations/haveli-dharampura/  → Detail page
```

## Data Model

Stored in `india-experiences/src/data/staycations.json`:

```json
{
  "slug": "haveli-dharampura",
  "href": "/staycations/haveli-dharampura",
  "name": "Haveli Dharampura",
  "location": "Delhi",
  "description": "",
  "homePageImage": "/images/staycations/{slug}-homePageImage-homepage.jpg",
  "cardImage": "/images/staycations/{slug}-cardImage-card.jpg",
  "heroImage": "/images/staycations/{slug}-heroImage-hero.jpg",
  "whoItSuits": "Description of ideal guests...",
  "gallery": [],
  "overview": {
    "description": "Multi-paragraph description...",
    "highlights": ["Highlight 1", "Highlight 2"],
    "checkIn": "14:00",
    "checkOut": "11:00"
  },
  "rooms": [{ "name": "Heritage Room", "description": "...", "amenities": ["AC", "WiFi"], "priceRange": "₹8,000-12,000" }],
  "dining": { "description": "...", "restaurants": [{ "name": "Lakhori", "cuisine": "North Indian" }] },
  "destination": { "name": "Old Delhi", "description": "...", "nearbyAttractions": ["Red Fort", "Jama Masjid"] },
  "booking": { "directBooking": true, "externalUrl": "https://..." },
  "tours": {
    "enabled": true,
    "source": "viator",
    "viatorDestinationId": 804,
    "viatorTagIds": [21911, 11866],
    "customTourIds": [],
    "gygEnabled": true,
    "gygTourIds": ["464596", "810824"]
  }
}
```

## Image Sizes

| Type | Dimensions | Aspect Ratio | Usage |
|------|------------|--------------|-------|
| homePageImage | 640x960 | 2:3 | Homepage cards |
| heroImage | 1400x600 | 7:3 | Detail page banner |
| cardImage | 800x500 | 8:5 | Listing thumbnails |

Stored at `/public/images/staycations/` as `{slug}-{imageType}-{size}.jpg`.

## Admin Panel

5 tabs: Overview, Images, Rooms, Dining, Tours.

Changes commit directly to GitHub → automatic rebuild.

---

## AI Import

Import hotel data from any URL using Jina Reader + Claude.

### Flow

```
URL submitted → Jina Reader fetches → Claude extracts → Images downloaded → Data returned → User selects fields → Merged into form
```

### API

`POST /api/ai-import` — SSE stream with progress updates:
```
data: {"step":"fetching","progress":10}
data: {"step":"extracting","progress":30}
data: {"step":"downloading_images","progress":60}
data: {"step":"uploading_images","progress":85}
data: {"step":"complete","progress":100,"data":{...}}
```

### Extraction

Claude extracts: name, location, overview (description, highlights, amenities), rooms, booking info, images.

Anti-hallucination rules: extract ONLY explicit information, "Not found" for missing data, no guessing.

### Merge Strategy

- User toggles individual fields
- Only selected fields merged
- Empty fields never overwrite existing data
- Images appended to gallery
- Rooms appended to list

### File Structure

```
admin/src/lib/ai-import/
├── types.ts              # Type definitions
├── jina-fetcher.ts       # Fetches via https://r.jina.ai/[URL]
├── content-extractor.ts  # Claude extraction with cleaning
└── image-downloader.ts   # Parallel download (3 concurrent, 10s timeout)
```

### Dependencies

- **Jina Reader** (free: 100 req/min, no API key)
- **Existing ANTHROPIC_API_KEY**
- No new npm packages

### Edge Cases

| Scenario | Handling |
|----------|----------|
| JS-rendered pages | Jina handles most; minimal data triggers warning |
| Missing data | Only show checkboxes for fields with values |
| Image download fails | Mark failed, continue others, show warning |
| Rate limits | Error in progress stream |
| CORS/hotlink protection | 10s timeout, graceful failure |
