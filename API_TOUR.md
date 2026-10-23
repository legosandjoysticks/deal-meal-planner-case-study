# Deal Meal Planner API Overview

Deal Meal Planner is API-first. The web and iOS apps are thin clients over a FastAPI service backed by SQLite with normalized views. This page describes how the service is shaped and why. It is an architecture summary, not a usage guide: the live endpoints are for the Deal Meal Planner apps only and are not a public API.

## What The Service Does

| Area | Purpose |
| --- | --- |
| Deal metadata | Store roster, deal counts and freshness for the dashboard |
| Best deals | Smart-ranked current deals for meal planning |
| BOGO and promos | Buy-one-get-one and multi-buy offers, kept apart from price ranks |
| Weekly brief | The week's deals grouped by store and category |
| Search | Shopper item search across stores (signed-in) |
| Meal planning | Deal-aware recipe generation and shopping lists (paid accounts) |
| Admin | Ingestion and review tools (staff only) |

## Design Notes

**BOGO is a first-class deal type, not a price.** Buy-one-get-one offers and "buy 1, get 1 50% off" promos do not survive a simple price-rank view. A free item is not actually a zero-dollar item. Deal Meal Planner keeps promo rows visible without pretending every offer is directly price-comparable.

**Normalization happens before product display.** Store flyer data arrives with inconsistent names, units, dates, multi-buy text, and sale conditions. The app uses normalized SQLite views so responses stay more predictable than the raw flyer rows.

**Two prices per deal.** Each deal keeps a parsed numeric value for filtering and ranking and the human display string from the flyer. The UI stays readable without forcing every promotion into one number.

**Scoring is not just "lowest price."** The best deal for meal planning is not always the cheapest numeric row. Smart ranking accounts for meal usefulness, category, price, savings, and current flyer scope.

**Counts are modeled, not summed.** Total deal counts and shelf-price counts differ on purpose, because promo, BOGO, and non-comparable rows are modeled separately.

**Access follows the product.** Shopper search and planning are signed-in features, meal generation is part of the paid plan, and ingestion tools are staff-only. Authentication and entitlement checks live on the server, not in the client.
