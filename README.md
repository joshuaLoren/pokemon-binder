# pokemon-binder

**Live:** https://joshualoren.github.io/pokemon-binder/

A Pokémon card binder for a kid's real collection: a 9-pocket page view, live TCGplayer market prices, and trend charts. One `index.html`, about 2,100 lines, no build step.

## What it does

- **Binder view.** Cards laid out nine to a page, with images and prices from the [Pokémon TCG API](https://pokemontcg.io).
- **Price history.** Each Refresh saves a dated snapshot locally, and the trend chips and charts are built from those snapshots.
- **Add by hand, camera, or video.** A form with a live set dropdown, OCR card scanning with Tesseract.js (loaded only on first scan), and video intake through the Claude API using a key that stays on the device.
- **Share links.** The binder is gzip-compressed (`CompressionStream`) into the URL fragment, so friends can open it without a server.

## Design choices

- **One file, no framework.** The owner is a beginner who learns by reading and editing the code, and a single file they can double-click is the whole app.
- **Local-first.** All state is in `localStorage` under versioned keys. Schema changes migrate old data instead of wiping it, because the price history can't be rebuilt after the fact.
- **Stable IDs.** Price history is keyed on card `id`, so IDs never change once they're created.
- **No backend.** Hosted on GitHub Pages. The fragment never reaches a server, and the API key is never put in a link.

## Run

Open `index.html` in a browser. Pushing to `main` redeploys the Pages site.

See [ROADMAP.md](ROADMAP.md) for what's next.
