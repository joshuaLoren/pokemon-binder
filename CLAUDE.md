# Pokémon Card Binder App

## What this is
A one-file web app (`pokemon-binder.html`) that shows the owner's real Pokémon card collection like a 9-pocket binder, with live prices and trend charts. The owner is a beginner who is learning — when you make changes, explain what you did in simple, friendly terms.

## Rules
- Keep everything in the single `pokemon-binder.html` file (HTML + CSS + JS together). No build tools, no npm, no frameworks.
- Test by opening the file in a browser (double-click it). No server needed.
- Don't break saved data. localStorage keys `binder.history`, `binder.meta`, and `binder.setids` hold the owner's price history built up over time. If a change needs a new data shape, migrate the old data instead of wiping it.

## How the app works
- The `CARDS` array (top of the script) is the collection. One object per card:
  - `id`: unique string — never change it once created, price history is keyed on it
  - `name`, `num` (like "160/132"), `set` (set name)
  - `total`: printed set size (132 for a /132 card); omit for promos
  - `sets`: candidate set IDs for the Pokémon TCG API (e.g. ["sv3"])
  - `nums`: card-number variants to try in queries (e.g. ["023","23"])
  - `kw`: lowercase keyword that must appear in the card's name (disambiguates)
  - `price`: baseline market price per copy in USD
  - Optional: `qty` (number of copies), `graded` (like "PSA 8" — graded cards never auto-update their price), `prev` (estimated price ~30 days before baseline, seeds the chart), `reveal` (true = name was unknown, adopt the API's name), `bulk` (a number = this is a bulk-lot tile with that many cards, no API lookup)
- Prices: `history` = `{ "YYYY-MM-DD": { cardId: pricePerCopy } }`. Baseline date is 2026-07-01 with an estimated 2026-06-01 point for some cards. The Refresh button fetches TCGplayer market prices from `https://api.pokemontcg.io/v2/cards` and saves today's snapshot. Trend chips compare the newest point vs the oldest.
- Card images come from the same API and are cached in `meta`. Cards the API doesn't know yet (very new sets) show a styled placeholder.
- `latestPrice()` falls back to a card's `price` field if it has no history yet — so adding a new card only requires adding it to `CARDS`, nothing else.

## Adding a card (the most common task)
Append one object to `CARDS`. Example — a Gengar ex 193/162 from Temporal Forces worth $12:

```js
{id:"gengar193", name:"Gengar ex", num:"193/162", total:162, sets:["sv5"], nums:["193"], kw:"gengar", set:"Temporal Forces", price:12}
```

It will show "— new" for its trend until it has two price snapshots. That's expected.

## Design tokens (keep the look consistent)
- Colors live in `:root`: deep navy background `--ink`, gold values `--gold`, holo cyan/violet accents `--foil-a`/`--foil-b`, trend colors `--up`/`--down`.
- Fonts: Fredoka for display text, Nunito for body.
- Signature effect: `.cardface` has a 3D tilt + rainbow holo shine driven by mousemove setting `--rx`/`--ry`/`--mx`/`--my`. New card-like elements should reuse it via `attachTilt()`.

## Feature ideas the owner is interested in
A search bar, an "add a card" form (so no code editing is needed), page-flip sound effects, and a rarity filter.
