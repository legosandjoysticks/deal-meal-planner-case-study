# Deal Meal Planner API Tour

Representative local-development examples for the Deal Meal Planner API. These examples are for engineering review and are intentionally kept out of the shopper-facing app.

Base URL used below:

```bash
BASE_URL=http://localhost:8000
```

## Deal Count

```bash
curl -sS "$BASE_URL/api/deals/count?price_mode=shelf&flyer_scope=latest"
```

Example response:

```json
{
  "count": 826
}
```

## Best Deals

```bash
curl -sS "$BASE_URL/api/deals/best?limit=10&price_mode=shelf&flyer_scope=latest&rank_mode=smart"
```

Example response:

```json
[
  {
    "store": "Meijer",
    "name": "Family Pack Chicken Drumsticks",
    "price_value": 1.29,
    "price_text": "$1.29/lb",
    "price_unit": "lb",
    "flyer_id": 200,
    "deal_type": "price",
    "conditions": null,
    "category": "meat",
    "sale_start_date": null,
    "sale_end_date": null,
    "sale_window_label": null
  }
]
```

## BOGO Watch

```bash
curl -sS "$BASE_URL/api/deals/bogo?limit=10&flyer_scope=latest"
```

Example response:

```json
[
  {
    "store": "Meijer",
    "name": "Kellogg's Frosted Mini Wheats or Rice Krispies",
    "price_value": null,
    "price_text": "BOGO FREE",
    "price_unit": null,
    "flyer_id": 186,
    "sale_start_date": null,
    "sale_end_date": null,
    "sale_window_label": null
  }
]
```

## Search

```bash
curl -sS "$BASE_URL/api/search?q=chicken&limit_stores=10&flyer_scope=latest"
```

Example response:

```json
{
  "q": "chicken",
  "store": null,
  "flyer_scope_used": "latest",
  "note": null,
  "results": [
    {
      "store": "Meijer",
      "price_value": 1.29,
      "price_text": "$1.29/lb",
      "price_unit": "lb",
      "match_name": "Family Pack Chicken Drumsticks",
      "flyer_id": 200,
      "deal_type": "price",
      "conditions": null,
      "category": "meat",
      "sale_start_date": null,
      "sale_end_date": null,
      "sale_window_label": null,
      "match_count": 4,
      "matches": []
    }
  ]
}
```

## Weekly Brief

```bash
curl -sS "$BASE_URL/api/weekly"
```

Example response:

```json
{
  "summary": [
    {
      "store": "Meijer",
      "headline": "Useful protein and pantry deals lead this week's flyer."
    }
  ],
  "overall": [
    {
      "store": "Meijer",
      "name": "Family Pack Chicken Drumsticks",
      "price_text": "$1.29/lb",
      "category": "meat"
    }
  ],
  "by_store": {
    "Meijer": [
      {
        "name": "Family Pack Chicken Drumsticks",
        "price_text": "$1.29/lb"
      }
    ]
  }
}
```
