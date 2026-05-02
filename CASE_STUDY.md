# Deal Meal Planner: Engineering Case Study

## One-Sentence Summary

Deal Meal Planner is a grocery deal intelligence platform that ingests weekly flyer data, normalizes inconsistent retail deals, and turns current promotions into searchable deal insights, meal ideas, and shopping-list workflows.

## Problem

Grocery flyers are messy. Prices appear in different formats, BOGO deals do not always have numeric prices, limited-day sales can sit next to regular weekly deals, and store pages often change layout. The user problem is simple: people want to save money on groceries. The engineering problem is harder: make inconsistent, time-sensitive retail data useful enough to plan around.

## Constraints

- Support multiple stores with different flyer formats.
- Preserve current flyer context so stale deals do not appear as weekly picks.
- Avoid ranking non-meal items as the best meal-planning choices.
- Keep BOGO and promotional rows visible without misrepresenting their prices.
- Make the frontend feel like a real product, not a data dump.
- Keep credentials, generated databases, and source code private for now.

## Approach

I designed the project as a system with four layers:

1. Data acquisition and review for weekly flyers.
2. SQLite storage and normalization for deal rows.
3. FastAPI endpoints for product-facing deal intelligence.
4. React screens for dashboard, weekly picks, search, BOGO, meal planning, and grocery lists.

## Key Tradeoffs

### SQLite Over a Hosted Database

SQLite kept the product lightweight, portable, and inspectable. For a portfolio project with local ingestion and frequent schema/data debugging, that was more valuable than adding hosted database complexity too early.

### Smart Ranking Over Lowest Price Only

The lowest shelf price is not always the best deal for meal planning. A cheap beverage can look like the "best" deal numerically while being useless for a dinner plan. Smart ranking makes space for price, savings, category, and recipe usefulness.

### Separate Promo Logic

BOGO and multi-buy deals are valuable, but they cannot always be compared against shelf-price rows. Separating promo rows makes the UI more honest and easier to reason about.

### Private Source, Public Case Study

The case study makes the project legible to recruiters while keeping the source code, data pipeline details, and operational credentials private until the release boundary is clear.

## Current Portfolio Proof

- Product screenshots for dashboard, weekly picks, best deals, search, BOGO, meal planning, and grocery list.
- A short no-audio demo GIF.
- API examples that show endpoint shape without exposing private deployment details.
- Written explanation of architecture, tradeoffs, learning, and next steps.

## What I Would Improve Next

- Add CI for backend smoke tests and frontend build.
- Improve difficult flyer extraction paths for promotions without prices.
- Add observability around ingestion freshness and endpoint failures.
- Build a cleaner admin workflow for reviewing extracted deals.
- Decide whether to commercialize, keep private, or selectively open-source small non-core pieces.
