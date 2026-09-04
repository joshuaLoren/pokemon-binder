# Pokémon Binder — Vision & Backlog

*The idea pile. Nothing here is promised — it's here so we don't lose it. (Captured Aug 27, 2026.)*

## The thesis (why this could be a real thing)

Two audiences, one app:

- **Parents love it** because it quietly teaches kids about investing. The cards are the hook; the lessons are real: prices move, trends have stories, patience pays, value concentrates ("half your binder's value is one Charizard"), and selling is a decision you can be early or late on.
- **Kids love it** because it's *their* collection: show it off, share it, peek at friends' binders — and eventually **play** with it by building decks from what they actually own.

The magic-link sharing (shipped Aug 2026) is the first "kids love it" pillar. Decks are the second. The investing layer is the parent pitch.

---

## Pillar 1 — Collection & sharing (v1 shipped)

Done: magic share links, visiting mode, friends shelf.

Later ideas:
- **Live binders** — today a share link is a snapshot. "Emerson's binder, always current" needs the data to live somewhere (see *Big fork: does this ever get a backend?*).
- Reactions ("🔥 on your Charizard"), trade wishlists ("cards my friend has that I want") — social features that only make sense once binders are live rather than snapshots.

## Pillar 2 — Decks (the play layer)

> Kids build decks **from their collection**, keep track of the decks they've built.

- Deck builder that pulls from the binder: pick cards you own, see what's left.
- Save multiple decks (`binder.decks`), name them, and **share a deck with the same magic-link trick** the binder uses.
- Show deck stats kids intuitively get: energy curve, type balance, "you have 3 of the 4 copies you're allowed."

**Design decisions to make before building:**
1. **Ownership-aware or freeform?** If decks strictly draw from the binder, we track "2 of your 3 Terapagos are in Deck A." That's the honest version of "build from your collection" — but it needs quantity accounting. Freeform deck lists are far simpler but break the fantasy. *Leaning: ownership-aware; it's the differentiator.*
2. **The bulk-lot problem.** The binder tracks bulk commons as one opaque tile ("×73"), but decks are built mostly *from* commons/trainers/energy. Deck building probably forces a way to itemize bulk (scan-to-itemize? a lightweight "I own this" checklist per set?).
3. **Format legality.** The Pokémon TCG API exposes standard/expanded legality per card — cheap to show a "tournament legal ✓" badge, fun for kids who play at school or league.

## Pillar 3 — AI deck coach

> Have AI chat about a deck: audit it, suggest tweaks, provide guidance.

The dream: a kid opens their deck and a friendly professor-type coach says what's strong, what's missing, and *why* — energy curve, draw power, win conditions — in encouraging kid-language. Maybe: "you're two $1.50 cards away from making this way better" (which also feeds the investing lesson: budget vs. upgrade).

**Design decisions to make before building:**
1. **Where does the AI live?** This is the big one, because the app is a static file with no server:
   - **Bring-your-own-key** (dad's Anthropic API key in localStorage) — zero infra, fine for family use, a non-starter for a public product (keys can't ship in a public page).
   - **Tiny serverless proxy** (Cloudflare Worker / Lambda) holding the key — breaks the "no server" rule, costs real money per chat, but is the only path to "any kid can use it."
   - **Copy-out-to-Claude** — a "📋 Copy my deck for Claude" button that formats the deck as a prompt. Zero infra, works today, kid pastes into the family Claude account. *Probably the v1.*
2. **Kid safety.** If chat ships inside the app for other people's kids, that's guardrails, moderation, and parental-consent territory (COPPA if it ever gets accounts). Family-only use dodges all of this — a public version does not.
3. **Coach personality.** Pokémon professors are the obvious archetype. Encouraging, never scolding, explains the *why* — the tone is the product.

## Pillar 4 — The investing layer (the parent pitch)

Already in the app: live prices, per-card trend charts, collection value.

Backlog:
- **Portfolio chart** — total collection value over time (the data already accumulates in `binder.history`; we just never graph the sum).
- **"What you paid vs. what it's worth"** — optional buy-price per card unlocks real P/L, the single best teaching tool in the app.
- **"Near its high / should I sell?" signal** (discussed pre-sharing-feature, never built) + longer chart windows.
- **Teachable moments, implicitly** — concentration ("top card = 40% of your binder"), best/worst movers of the month. *Design decision: keep lessons ambient (badges, one-liners) vs. an explicit "learn" mode with quizzes. Leaning ambient — it's a toy that happens to teach, not homework.*
- Maybe someday: a parent-facing view/recap ("Emerson checked prices 4× this week; binder up $12").

## Pillar 5 — Video intake: the self-building binder ✅ BUILT Aug 27, 2026

> Record yourself flipping through your real binder, upload the video, and the app catalogs the cards. When a card can't be matched confidently, an AI workflow makes best guesses and asks the user to **confirm / deny / update**.

Decided Aug 27, 2026: this is the video feature. ("Binder tours" — sharing the video itself as content — was considered and **explicitly rejected**; the video is an input, not the product.)

**Status: v1 shipped.** The 🎬 Video button runs the full pipeline below. Frame-picking is verified (a 3-page test flip-through → exactly 3 sharp page frames, 14 mid-flip frames rejected) and the review queue is verified end-to-end. The AI + lookup stages need a real key and a real binder video to exercise — **that's the next thing to try for real.**

**The v1 pipeline (all in the one file):**
1. **Capture** — record or pick a video (same `<input capture>` pattern as the photo scanner).
2. **Frame picking** — client-side: sample frames, detect page-flips (scene change), keep the sharpest frame per page. A flip-through video is really just a stream of 9-card page photos.
3. **Recognition** — a vision model (Claude) reads each kept frame and lists every card it can see: name, collector number if legible, set guess, confidence. Runs browser→API directly with a family API key kept in localStorage (the serverless way to have AI in a static file; "ask a grown-up" setup).
4. **Matching** — each guess is resolved against the Pokémon TCG API (the same lookup machinery the scanner uses) into a real card with picture + price.
5. **The review queue (the heart of it)** — confident matches come pre-checked; ambiguous ones show the top few candidates to tap; misses show the AI's best guess in editable fields to fix and retry. Confirm / deny / update each row, then add everything to the binder in one go. Cards already in the binder get flagged so you don't double-add.

**Later rungs:** live camera mode (no upload step); a client-side card matcher (perceptual hashing against the card-image DB) to remove the API-key requirement; smarter per-pocket tracking across frames.

---

## Platform decisions (the boring stuff that shapes everything)

- **Hosting** — undecided/pending: GitHub Pages (public repo, free URL) so share links work anywhere and the binder lives on phones. Wrinkle when it happens: `binder.usercards` is per-browser, so fold user-added cards into `BUILTIN_CARDS` at publish time.
- **Domain** — checked Aug 27, 2026: `mypokemonbinder.com` TAKEN (July 23, 2026, Cloudflare); `mypokemonbinder.app` and `.net` AVAILABLE; `pokebinder.com` taken. ⚠️ "Pokémon" is Nintendo's trademark — fine for a family project, a real conversation before anything public or monetized.
- **Big fork: does this ever get a backend?** Live binders, real friend accounts, in-app AI chat, and cross-device sync all point at a backend; snapshots-in-links, BYO-key, and copy-to-Claude all preserve the beloved one-file zero-infra design. The one-file rule has been the superpower so far — break it deliberately or not at all.

## Small stuff already on the list

Search bar · rarity filter · sell signal · longer chart windows.
