# API Tour

Deal Meal Planner is API-first. The frontend is a thin client over a
FastAPI service backed by SQLite with normalized views. This document
walks the public-shape endpoints with real example responses so a
reviewer can see the surface without running the project.

> **Note.** The hostnames below are illustrative. The deployed
> instance serves these routes under the same origin as the demo
> frontend, behind a Cloudflare Tunnel. The application source is
> private; this tour exists so an engineering reviewer can see the
> API design without it.

## At a glance

| Route                            | Method | Returns                            |
| -------------------------------- | ------ | ---------------------------------- |
| `/api/deals/meta`                | GET    | Store roster and total deal count  |
| `/api/deals/best`                | GET    | Smart-ranked deals across all stores |
| `/api/deals/bogo`                | GET    | BOGO and promo rows (separate from price ranks) |
| `/api/deals/count`               | GET    | Live deal count for the dashboard  |
| `/api/search?q=...`              | GET    | Cross-store item search with fuzzy matches |
| `/api/weekly`                    | GET    | Weekly brief grouped by store and bucket |
| `/api/intents`                   | GET    | Normalized category taxonomy       |
| `/api/generate`                  | POST   | Deal-aware recipe generation       |
| `/api/admin/*`                   | —      | Ingestion + admin (auth-gated, non-public) |

## Design notes

Three decisions worth calling out before the endpoint walk.

**BOGO is a first-class deal type, not a price.** Buy-one-get-one
offers and "buy 1, get 1 50% off" promos don't survive a price-rank
view — a free item isn't actually $0. So `/api/deals/best` returns
shelf-priced and multi-unit deals only, and `/api/deals/bogo`
returns the promo rows separately. The frontend mirrors this split:
the BOGO Watch panel is a different visual treatment from the
ranked deal grid.

**Normalization happens once, in views.** The deal table holds raw
ingested rows from each store flyer. Cleaning, category mapping, and
the "is this price-comparable" flag are SQLite views over that
table. New ingestion logic does not need to touch query code.

**Scoring is configurable, not just "lowest price."** `/api/deals/best`
accepts a `rank_mode` (default `smart`) that weights savings percentage,
category usefulness, and shelf-price normalization. A grocery deal
intelligence product wants "the best deals to actually buy," which
is not the same question as "the lowest numbers in the database."

---

## `/api/deals/meta`

Returns the store roster currently tracked and the total normalized
deal count in the database. Used by the dashboard header to show
"826 deals · 6 stores."

```bash
curl -s https://app.dealmealplanner.com/api/deals/meta
```

```json
{
  "stores": ["Acme", "Aldi", "Giant Eagle", "Heinen's", "Marc's", "Meijer"],
  "count": 826
}
```

---

## `/api/deals/best`

Smart-ranked best deals across all stores. The default `rank_mode=smart`
returns deals weighted by usefulness (price savings + category fit +
shelf-comparable normalization), not just the lowest numbers.

**Query parameters:**

| Param          | Default   | Description                                    |
| -------------- | --------- | ---------------------------------------------- |
| `limit`        | `6`       | Number of deals to return                      |
| `price_mode`   | `shelf`   | `shelf` for unit-comparable deals only         |
| `flyer_scope`  | `latest`  | `latest` for current week, `all` for history   |
| `rank_mode`    | `smart`   | `smart` (weighted) or `price` (cheapest first) |

```bash
curl -s "https://app.dealmealplanner.com/api/deals/best?limit=3"
```

```json
[
  {
    "store": "Giant Eagle",
    "name": "Arizona",
    "price_value": 0.69,
    "price_text": "69¢ ea.",
    "price_unit": "ea.",
    "flyer_id": 182,
    "deal_type": "price",
    "conditions": "1 day sale Wednesday 4/29/26 ONLY",
    "category": null,
    "sale_start_date": null,
    "sale_end_date": null,
    "sale_window_label": null
  },
  {
    "store": "Acme",
    "name": "Colgate Toothpaste or Plus Toothbrush",
    "price_value": 0.69,
    "price_text": "69¢",
    "price_unit": null,
    "flyer_id": 198,
    "deal_type": "multi",
    "conditions": "Limit 6",
    "category": null,
    "sale_start_date": null,
    "sale_end_date": null,
    "sale_window_label": null
  }
]
```

Notes on the shape: `price_value` is the parsed numeric price for
ranking; `price_text` is the human display string ("69¢ ea." vs
"$2.99"). The two are stored separately so display logic never
fights parser logic.

---

## `/api/deals/bogo`

Buy-one-get-one and other promo rows that aren't directly
price-comparable. Separate endpoint, separate UI panel — same
philosophical reason.

```bash
curl -s "https://app.dealmealplanner.com/api/deals/bogo?limit=3"
```

```json
[
  {
    "store": "Meijer",
    "name": "Frederik's by Meijer Pre-Sliced Meat or Cheese*",
    "price_value": null,
    "price_text": "BOGO",
    "price_unit": null,
    "flyer_id": 186,
    "sale_start_date": null,
    "sale_end_date": null,
    "sale_window_label": null
  },
  {
    "store": "Meijer",
    "name": "Kellogg's Frosted Mini Wheats or Rice Krispies",
    "price_value": null,
    "price_text": "BOGO FREE",
    "price_unit": null,
    "flyer_id": 186
  },
  {
    "store": "Meijer",
    "name": "Tyson Tenders & Ready Fresh Chicken",
    "price_value": null,
    "price_text": "BOGO 50% off",
    "price_unit": null,
    "flyer_id": 186
  }
]
```

`price_value` is intentionally `null` on these rows — the parser
refuses to invent a price for a promo, and the UI is built to
handle that gracefully rather than masking it with a `$0`.

---

## `/api/search?q={query}`

Cross-store item search. The interesting part is the response
shape: every result carries both a single best match (for the
"compare price across stores" card) and a `matches[]` array (for
the modal's full list of variants).

```bash
curl -s "https://app.dealmealplanner.com/api/search?q=chicken" | head -c 600
```

```json
{
  "q": "chicken",
  "store": null,
  "results": [
    {
      "store": "Meijer",
      "price_value": 0.99,
      "price_text": "$0.99",
      "price_unit": null,
      "match_name": "Fresh from Meijer Family Pack Chicken Drumsticks",
      "flyer_id": 186,
      "deal_type": "price",
      "conditions": "sale",
      "category": "meat & seafood",
      "match_count": 7,
      "matches": [
        { "match_name": "Fresh from Meijer Family Pack Chicken Drumsticks", "price_text": "$0.99" }
        /* ... 6 more variants ... */
      ]
    }
  ]
}
```

`match_count: 7` means the same store had seven different chicken
SKUs in this week's flyer; the response keeps the best one
surfaced and the rest available without a second request.

---

## `/api/weekly`

Weekly brief — the JSON behind the "Best of the week" sections.
Returns deals grouped by overall ranking and by store, plus the
weekly category buckets ("meat," "produce," "pantry").

```bash
curl -s https://app.dealmealplanner.com/api/weekly
```

```json
{
  "overall": [
    {
      "store": "Meijer",
      "name": "Tyson Tenders & Ready Fresh Chicken or All Natural Frozen Chicken Breasts, Wings or Tenders*",
      "price_value": null,
      "price_text": "BOGO 50% off",
      "price_unit": null,
      "reference_price": null,
      "savings_pct": null,
      "bucket": "meat",
      "meal_plannable": true,
      "weekly_kind": "promo",
      "weekly_treat": false,
      "sale_window_label": null,
      "smart_score": 20,
      "score": 42.0
    }
    /* ... ~140 more entries grouped by store + bucket ... */
  ]
}
```

Notable fields:

- `meal_plannable` — does this deal make sense as a meal-plan
  input? Excludes toothpaste, paper goods, etc., even when those
  are great deals.
- `weekly_kind` — `price` vs `promo` vs `treat`. Keeps the UI
  honest about what kind of deal it's surfacing.
- `bucket` — coarse category for visual grouping.
- `smart_score` and `score` — the two scoring channels, kept
  separate so display logic can sort by either without recomputing.

---

## `/api/intents`

Normalized category taxonomy. Used by the planner's chip filter
and by the ingestion pipeline as the target vocabulary for
flyer-row category mapping.

```bash
curl -s https://app.dealmealplanner.com/api/intents | head -c 400
```

```json
[
  "adult incontinence",
  "air fresheners & home scent",
  "allergy & sinus",
  "apparel - loungewear",
  "applesauce & fruit cups",
  "automotive & garage",
  "avocados",
  "baby diapers",
  "baby health & care",
  "bacon",
  "bags & accessories",
  "bath tissue",
  "beans",
  "..."
]
```

The taxonomy is hand-curated and stable — adding a new store does
not change the vocabulary, only its mapping inputs.

---

## `/api/generate` (POST)

Deal-aware recipe generation. Takes user preferences (mood,
servings, required ingredients, excluded ingredients, available
appliances, allowed stores) plus clipped deals and returns a meal
plan that prefers recipes built around the clipped items.

```bash
curl -s -X POST https://app.dealmealplanner.com/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "servings": 4,
    "mood": "weeknight",
    "stores": ["Meijer","Aldi"],
    "appliances": ["oven","stovetop"],
    "required_ingredients": ["chicken"],
    "excluded_ingredients": ["mushrooms"],
    "clipped_deals": [186, 192, 204]
  }'
```

Response shape (abridged):

```json
{
  "plan_id": "wk-2026-18",
  "days": [
    {
      "day": "Mon",
      "recipe": "Sheet-pan chicken thighs with potatoes",
      "ingredients_used_from_deals": ["Family pack chicken thighs (Meijer)"],
      "estimated_cost": 8.40,
      "deal_coverage_pct": 78
    }
    /* ... 6 more days ... */
  ],
  "summary": {
    "estimated_total": 74.20,
    "deal_coverage_pct": 62
  }
}
```

AI generation is gated by explicit guardrails: required and excluded
ingredients are hard constraints, deal coverage is reported (not
maximized at the expense of edibility), and the response includes a
provenance line per ingredient so the UI can show which item came
from which clipped deal.

---

## `/api/admin/*` (non-public)

Auth-gated. Handles flyer ingestion, manual deal entry, category
remapping, and database hygiene. Not reachable on the public
deployment; runs locally and through a separately authenticated
subdomain in production.

Endpoints (for completeness):

| Route                       | Purpose                                |
| --------------------------- | -------------------------------------- |
| `POST /api/admin/flyers`    | Trigger flyer ingestion                |
| `GET /api/admin/flyers`     | List ingested flyers                   |
| `POST /api/admin/deals`     | Add a deal the parser missed           |
| `PATCH /api/admin/deals/{id}` | Edit / categorize a single deal      |

---

## Where to next

- Architecture and design rationale: [README.md](./README.md)
- Source-boundary policy: [SECURITY_AND_SOURCE_BOUNDARY.md](./SECURITY_AND_SOURCE_BOUNDARY.md)
- Demo video: [`assets/demo/deal-meal-planner-demo.mp4`](./assets/demo/deal-meal-planner-demo.mp4)
