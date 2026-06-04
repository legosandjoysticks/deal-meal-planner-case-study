# Deal Meal Planner API Tour

Deal Meal Planner is API-first. The frontend is a thin client over a FastAPI service backed by SQLite with normalized views. This document walks production-safe endpoint examples so a reviewer can see the public surface without access to the private source repo.

Base URL used below:

```bash
BASE_URL=https://api.dealmealplanner.com
```

Current metadata check, verified June 4, 2026:

```bash
curl -sS "$BASE_URL/api/deals/meta"
```

```json
{
  "stores": ["Acme", "Aldi", "Giant Eagle", "Heinen's", "Marc's", "Meijer"],
  "count": 1830,
  "updated_at": "2026-06-04T13:48:05"
}
```

## At A Glance

| Route | Method | Returns |
| --- | --- | --- |
| `/api/deals/meta` | GET | Store roster and total deal count |
| `/api/deals/count` | GET | Live deal count for the dashboard |
| `/api/deals/best` | GET | Smart-ranked current deals |
| `/api/deals/bogo` | GET | BOGO and promo rows, separate from price ranks |
| `/api/weekly` | GET | Weekly brief grouped by store and bucket |
| `/api/search?q=...` | GET | Product-facing item search, protected in the live app |
| `/api/generate` | POST | Deal-aware recipe generation, protected in the live app |
| `/api/admin/*` | - | Ingestion and admin routes, auth-gated and non-public |

## Design Notes

**BOGO is a first-class deal type, not a price.** Buy-one-get-one offers and "buy 1, get 1 50% off" promos do not survive a simple price-rank view. A free item is not actually a zero-dollar item. Deal Meal Planner keeps promo rows visible without pretending every offer is directly price-comparable.

**Normalization happens before product display.** Store flyer data arrives with inconsistent names, units, dates, multi-buy text, and sale conditions. The app uses normalized SQLite views so API responses stay more predictable than the raw flyer rows.

**Scoring is not just "lowest price."** The best deal for meal planning is not always the cheapest numeric row. Smart ranking accounts for meal usefulness, category, price, savings, and current flyer scope.

**Public examples respect the product boundary.** Metadata, best-deal, BOGO, and weekly-pick examples can be shown safely. Search, planner, and admin workflows may require authentication in the live app and are documented as architecture rather than guaranteed anonymous endpoints.

## Deal Count

```bash
curl -sS "$BASE_URL/api/deals/count?price_mode=shelf&flyer_scope=latest"
```

Example response:

```json
{
  "count": 1643
}
```

The metadata count and shelf-price count can differ because promo rows, BOGO rows, and non-price-comparable rows are intentionally modeled separately.

## Best Deals

```bash
curl -sS "$BASE_URL/api/deals/best?limit=10&price_mode=shelf&flyer_scope=latest&rank_mode=smart"
```

Example response:

```json
[
  {
    "store": "Acme",
    "name": "Fresh! Boneless Chicken Thighs",
    "price_value": 2.99,
    "price_text": "$2.99",
    "price_unit": "LB",
    "deal_type": "price_drop",
    "conditions": "SAVE $2.00/LB. WITH CARD $2.99 lb.",
    "category": "meat",
    "sale_window_label": "Jun 4 - Jun 10, 2026",
    "smart_score": 35
  }
]
```

`price_value` is the parsed numeric value used for filtering and ranking. `price_text` is the human display string. Keeping both lets the UI stay readable without forcing every promotion into one numeric model.

## BOGO Watch

```bash
curl -sS "$BASE_URL/api/deals/bogo?limit=10&flyer_scope=latest"
```

Example response:

```json
[
  {
    "store": "Giant Eagle",
    "name": "Cheerios Honey Nut Cereal",
    "price_value": 3.49,
    "price_text": "$3.49",
    "price_unit": "32 cents/oz",
    "deal_type": "bogo",
    "conditions": "Buy 2 Get 2 Free",
    "sale_window_label": "Jun 4th - Jun 10th"
  }
]
```

The separate BOGO endpoint keeps promotions visible without letting ambiguous promo rows distort the best-deal ranking.

## Weekly Brief

```bash
curl -sS "$BASE_URL/api/weekly"
```

Example response:

```json
{
  "overall": [
    {
      "store": "Meijer",
      "name": "Meijer Ground Beef or Ground Beef and Pork Blend",
      "price_text": "BOGO 40% off",
      "weekly_kind": "promo"
    }
  ],
  "by_store": {
    "Meijer": [
      {
        "name": "Meijer Stuffed Chicken Breasts",
        "price_text": "BOGO"
      }
    ]
  }
}
```

## Search And Planner Boundary

Search and planner workflows are product-facing flows and may require authentication in the live app. They are still part of the architecture:

```bash
curl -sS "$BASE_URL/api/search?q=chicken&limit_stores=10&flyer_scope=latest"
```

Representative response shape:

```json
{
  "q": "chicken",
  "store": null,
  "flyer_scope_used": "latest",
  "results": [
    {
      "store": "Acme",
      "price_value": 2.99,
      "price_text": "$2.99",
      "price_unit": "LB",
      "match_name": "Fresh! Boneless Chicken Thighs",
      "deal_type": "price_drop",
      "conditions": "SAVE $2.00/LB. WITH CARD $2.99 lb.",
      "category": "meat",
      "sale_window_label": "Jun 4 - Jun 10, 2026",
      "match_count": 3,
      "matches": []
    }
  ]
}
```
