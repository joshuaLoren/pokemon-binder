# Pokémon Card Binder App

## What this is
A one-file web app (`index.html`) that shows the owner's real Pokémon card collection like a 9-pocket binder, with live prices and trend charts. The owner is a beginner who is learning — when you make changes, explain what you did in simple, friendly terms.

## Rules
- Keep everything in the single `index.html` file (HTML + CSS + JS together). No build tools, no npm, no frameworks. One runtime exception: the card **scanner** lazily loads Tesseract.js from a CDN, but only the first time someone taps Scan — the app itself ships as one file and works fully without it.
- Test by opening the file in a browser (double-click it). No server needed. (The scanner needs internet the first time to fetch the OCR engine.)
- Don't break saved data. localStorage keys `binder.history`, `binder.meta`, `binder.setids`, `binder.usercards` (cards the owner added via the form), `binder.setlist` (cached set names for the form's dropdown), `binder.profile` (the name the owner shares as), `binder.friends` (saved friend binders), and `binder.aikey` (the Anthropic API key for video intake — device-only, never leaves the device except to api.anthropic.com, and never enters a share link) hold data built up over time. If a change needs a new data shape, migrate the old data instead of wiping it.

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
- The full card list is `CARDS = BUILTIN_CARDS.concat(userCards)`. `BUILTIN_CARDS` is the hard-coded list in the script; `userCards` is loaded from `binder.usercards` and holds cards the owner added through the app. `rebuildCards()` re-merges them after any add/remove.

## Adding a card
There are two ways:

1. **The + Add card button (what the owner uses).** Opens a form: card name, set (dropdown built live from the API), card number (e.g. `193/162` or `SVP 165`), copies, price (blank = auto-look-up), and graded. On submit it derives `nums`/`total`/`kw`, saves the card to `binder.usercards`, fetches the picture + today's price, and shows it. Cards added this way have `user:true` and can be deleted from their detail modal (the 🗑 Remove button — only shows for `user:true` cards). Never delete `BUILTIN_CARDS` this way.

2. **Editing code (for the built-in list).** Append one object to `BUILTIN_CARDS`. Example — a Gengar ex 193/162 from Temporal Forces worth $12:

```js
{id:"gengar193", name:"Gengar ex", num:"193/162", total:162, sets:["sv5"], nums:["193"], kw:"gengar", set:"Temporal Forces", price:12}
```

Either way it shows "— new" for its trend until it has two price snapshots. That's expected.

3. **Scanning with the camera (📷 Scan button).** Captures a photo (`#scanInput`, `capture="environment"`), downscales it, runs on-device OCR (`loadTesseract()` → Tesseract.js), and pulls the printed **name + collector number** out of the text (`lookupByOcr`). It queries the same API by number (filtered by printed total) — falling back to promo codes, then a name search — and shows the closest matches for the owner to confirm (`showScanResults`). Tapping one calls `confirmScanned(apiCard)`, which builds the same card shape as the form and stores image + today's price. It's suggest-and-confirm on purpose: OCR is imperfect, so it never blind-adds; "type it in" drops back to the manual form.

## Video intake (🎬 Video) — the self-building binder
Film a flip-through, and the app catalogs the cards. Four stages in `runVideo()`, each with its own screen (`vidProgress()` draws the 1-2-3-4 tracker):
1. **`extractPageFrames()`** — pure client-side, no AI. Seeks the video every ~0.35s, and for each sampled frame computes a 16×16 grayscale fingerprint (`frameStats().sig`) plus a focus score (neighbour-difference edge energy). A fingerprint jump > 9 starts a new "shot" (a page); within a shot the sharpest frame wins. **Selection is the important part:** shots are sorted by sharpness and kept only while `focus >= bestFocus * 0.5`, plus a near-duplicate skip (`sigDiff < 7`). Measured on a synthetic flip-through, held pages scored 26–31 and every mid-flip frame ≤ 12 — that gap is why the relative gate works, and it cut a 3-page test video from 14 frames to exactly 3. Don't replace it with a fixed top-N: that pads the batch with motion-blur junk and pays to read it.
2. **`aiReadFrame()`** — posts each page frame to the Claude Messages API (`claude-opus-5`, vision, `effort: medium`) with `AI_PROMPT`, straight from the browser (`anthropic-dangerous-direct-browser-access: true`). Returns `[{name, number, set, confidence}]`, parsed defensively (JSON is sliced out of the text; bad JSON → `[]`). Handles 401/403 (clears the key), 429 (one retry), and `stop_reason: "refusal"`.
3. **`matchGuess()`** — resolves each guess against the Pokémon TCG API by number (filtered by printed total) then by name, reusing `apiGet`/`parseNum`/`deriveKw`. Returns up to 4 ranked candidates.
4. **`showReview()` / `fixRow()` / `commitReview()`** — the confirm/deny/update queue, and the reason this feature is trustworthy. Rows are pre-checked only when confident (≥0.45) and not already owned; tags mark `sure` / `not sure` / `already in your binder` / `no match`. `fixRow()` is the "update" path: pick a different candidate, or retype name/number and re-search. **Nothing is added until the owner presses the button** — keep it that way; OCR and vision are both fallible.

`addApiCard(x, qty)` is the shared data-only "API card → binder card" helper (used by both the camera scanner and bulk video adds); it never touches the screen, so the video flow adds a stack and renders once.

## Sharing + friends ("magic links")
- **How sharing works:** there is no server and no accounts. The 📤 Share button packs the whole binder — every card's display name, set, number, quantity, graded label, latest price, and cached picture URL — into a compact object (`packBinder()`, one-letter keys), gzips it with the browser's built-in `CompressionStream`, base64url-encodes it, and puts it after `#b=` in the URL (`buildShareLink()`). The link IS the binder. A `0.`-prefixed uncompressed fallback exists for browsers without CompressionStream; `1.` means gzip.
- **Visiting mode:** opening a `#b=` link (on load or via `hashchange`) decodes it (`decodeShare()` → `hydrateCards()`) and sets the global `VIEW` (null = your own binder). Visited cards carry `_v:true`, a fixed `price`, `_img`, `_rarity` — render/price/modal code branches on those. `document.body.classList` gets `visiting`, which hides Refresh/Scan/Add/Share; the visit banner (`#visitBar`) shows 💾 Save to my friends (link visits only) and 🏠 Back to my binder. Card pop-ups use `openVisitModal()` — no charts, since price history stays with the owner.
- **Friends shelf:** 👥 Friends lists `binder.friends` (raw packed cards + name + dates), opens them in visiting mode, deletes with confirm, and has a paste-a-link field for opening a share link by hand. Saving dedupes by name (offers replace).
- **Trust nothing from a link:** `decodeShare`/`hydrateCards` length-cap every field, clamp numbers, drop non-`https://` image URLs, and everything rendered goes through `escapeHtml`. Keep it that way — a share link is outside input.
- **Prices in a share are a snapshot** from the day the link was made (`packBinder` stamps `t`); the header says "a snapshot from …" while visiting.

## Design tokens (keep the look consistent)
- Colors live in `:root`: deep navy background `--ink`, gold values `--gold`, holo cyan/violet accents `--foil-a`/`--foil-b`, trend colors `--up`/`--down`.
- Fonts: Fredoka for display text, Nunito for body.
- Signature effect: `.cardface` has a 3D tilt + rainbow holo shine driven by mousemove setting `--rx`/`--ry`/`--mx`/`--my`. New card-like elements should reuse it via `attachTilt()`.
- Real-binder look: the page sits inside a `.binder` cover (leatherette gradient + dashed-border "stitching"), bound by a `.spine` gutter and metal `.ring` D-rings. Each card sits behind a glossy clear sleeve (`.cardface::before`). Turning a page runs a hinged 3D flip — `flip(dir)` swaps the page content while it's edge-on, using the `turnOut*/turnIn*` keyframes, plus a soft synthesized paper riffle from `playFlip()` (Web Audio, no sound file). Keep the grid columns as `minmax(0,1fr)` so long card names ellipsis-clip instead of overflowing on mobile.

## Feature ideas the owner is interested in
**The full vision and backlog live in `ROADMAP.md` — read it before proposing or building new features.** Short version: two audiences (parents → teaches kids investing; kids → share collections, build decks from what they own, AI deck coach). Near-term small stuff: a search bar and a rarity filter; a "near its high / should I sell?" signal + longer chart windows. Hosting on GitHub Pages + a domain is discussed there too (note: `binder.usercards` is per-browser — fold user-added cards into `BUILTIN_CARDS` when publishing to a hosted URL). Done so far: add-card form, camera scanner, real-binder styling, page-flip sound, share-links + friends shelf.
