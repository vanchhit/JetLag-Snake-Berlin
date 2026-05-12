# Berlin Snake

A Jet Lag the Game–inspired, real-life chase game played across Berlin's U-Bahn and S-Bahn network. One **Runner** is the head of a Snake; every station they claim joins a tail they can never re-enter. **Hunters** chase them through the city. The longer the snake, the more boxed-in the Runner gets.

This repository contains everything you need to play.

## What's in here

| File | What it is |
| --- | --- |
| `RULEBOOK.md` | The full game rules, role descriptions, snake mechanic, scoring, variants, safety notes, and quick-reference card. **Read this first.** |
| `CHALLENGES.md` | The canonical text of every Challenge, Curse, and Power card. Useful for review or remixing. |
| `cards.html` | Printable A4 card sheets, 3×3 per page. Open in a browser → "Print". |
| `index.html` | The playable companion web app. Open in any modern browser. |

## How to play

1. **Read the rulebook.** It's about a 10-minute read and explains the Snake twist that makes this game different from a standard manhunt.
2. **Print the cards** (optional). Open `cards.html` and print all sheets onto card stock. Cut and shuffle into three decks: Challenges, Curses, Powers. If you'd rather go all-digital, the app contains the same decks.
3. **Open the web app.** Each player loads `index.html` in their phone's browser. The Runner picks a starting station, sets duration (default 4 hours) and target length (default 15 stations), and starts.
4. **Run the game.** The Runner taps the map to claim adjacent stations; the app draws Challenges, tracks the tail, fires location pings every 30 minutes, and enforces the no-revisit rule. Hunters throw Curses every 45 minutes and confirm a tag when they catch up.

## Running the web app

`index.html` is a single self-contained file. No build step. You can:

- Open it directly in a browser by double-clicking, or
- Serve it from any static host (Netlify, GitHub Pages, Vercel, `python -m http.server`, etc.) so everyone in the group can load it from a phone.

The app uses **Leaflet** (loaded from a CDN) for the map and stores game state in `localStorage` on each device. Each player runs their own copy; the game is currently *local-first* with no server-side sync. The Hunter team coordinates over their own group chat as they would in any Jet Lag game.

### Browser requirements
- Modern browser (Chrome, Safari, Firefox, Edge, all current versions).
- Internet connection (Leaflet tiles load from CARTO/OpenStreetMap).
- localStorage enabled.

### What the app does
- Renders the Berlin U-Bahn and S-Bahn Ringbahn network as a clickable map.
- Tracks the Runner's tail, current head, and starting station.
- Enforces the self-collision rule and warns on non-adjacent moves.
- Suggests LEAP when a 2-stop move is needed.
- Rolls a 1d6 per claim to determine challenge difficulty (Easy 1pt / Medium 2pt / Hard 3pt).
- Schedules location pings every 30 minutes (every 20 if Curse of the Echo is active).
- Tracks the 45-minute curse cooldown for Hunters.
- Detects Ouroboros opportunities (looping back to start) and box-in losses.
- Persists everything in localStorage so a refresh doesn't reset the game.

## Station data caveat

Coordinates for ~200 U-Bahn and S-Bahn stations are embedded in the app. They're approximate (within roughly 200 metres for most) and the line adjacency graph is hand-built. Players use the real BVG app for routing in the wild — the map is a tracker, not a GPS replacement.

If you find an error or want to extend the network (more S-Bahn spokes, tram, regional rail), the `LINE_DATA` block at the top of the `<script>` section in `index.html` is the only place you need to edit. Each line is an ordered list of `[name, lat, lng]`; adjacency is derived from sequence.

## Customizing the game

- **Different city** — replace the contents of `LINE_DATA` with your city's transit graph. The rest of the app is city-agnostic.
- **Different scoring** — tweak `rollChallengeTier()` and the points table in `CARDS` to rebalance.
- **Different deck** — drop in themed challenge cards (food tour, public art, Cold War sites). The app picks uniformly at random from each tier.
- **Multi-device sync** — out of scope for the local-first version; the obvious extension is a small Firebase/Supabase layer broadcasting `state` between players.

## Safety & etiquette

The Rulebook has a full safety section. The two non-negotiables:
- **Tag = a hand on the shoulder.** Not a grab, not a tackle.
- **Pause if anyone is unwell, lost, or not having fun.** This is supposed to be fun.

Don't film strangers without consent, don't block transit exits, don't run on platforms, and stop by 10pm unless you've agreed otherwise.

---

Built for the Berlin transit map of 2026. Have fun. Don't get caught.
