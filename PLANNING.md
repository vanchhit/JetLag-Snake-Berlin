# Berlin Snake — Webapp Design & Planning Document

## 1. Current State Audit

### What the app already does well
- Leaflet map with dark tile layer, all U-Bahn lines + S41 Ringbahn drawn as polylines
- Full station/adjacency graph built from ordered `LINE_DATA` arrays
- Game state persisted to `localStorage` — refresh-safe
- Runner and Hunter roles on the same device (toggle in header)
- Challenge card draw (random tier roll 1d6 → easy/medium/hard)
- Claim flow: pick adjacent → draw card → confirm claim
- Curse throw with 45-minute cooldown enforcement
- Power cards: LEAP (auto-triggered on 2-stop tap), VEIL, SHED
- Location ping scheduler (30 min, or 20 min when Curse of Echo is active)
- Win/loss detection: timer expiry, box-in, Ouroboros offer
- Pause/resume with accumulated offset
- Game log (last 50 events, monospace, color-coded by type)
- Printable card sheets (`cards.html`) — separate file, works independently

---

## 2. Known Bugs

| # | Bug | Location | Impact |
|---|-----|----------|--------|
| B1 | S41 adjacency loop broken | `LINE_DATA.S41` last stop = first stop ("Gesundbrunnen"), creates a self-edge, not a circular adjacency | Medium — Ring travel between Gesundbrunnen and Schönhauser Allee may mis-route |
| B2 | `isTwoStop` ignores LEAP-consumed state when auto-suggesting | `isTwoStop()` / `onStationClick()` | Minor — LEAP modal appears even when power is gone |
| B3 | `state.log` not initialized in `defaultState()` | `defaultState()` | Minor — first `logEvent` call adds to undefined |
| B4 | Box-in check incorrectly requires SHED to be unused | `checkWinConditions()` — `state.powers.shed && nowGame() > 30 * 60_000` | The SHED-available warning should not substitute the box-in loss |
| B5 | Re-roll button visible even when no pending challenge | `btn-redraw` always present | UX — button does nothing but doesn't explain why |

---

## 3. Feature Gaps

### 3a. Missing station data
- **S42** (counter-clockwise Ring) is absent — only S41 is in `LINE_DATA`. S42 has identical stops in reverse; it matters because adjacency on the Ring is directional only in real life, but the app currently makes Ring stations adjacent in both directions via S41 alone.
- **S-Bahn spokes** (S1, S3, S5, S7, S9, S25, S26, S45, S46, S47, S75, S85) are all missing. The game zone can extend to all of Zone AB, but the current map only covers the Ring interior.
- **Transfer stations** (e.g., Ostkreuz, Jungfernheide) would gain additional line memberships if spokes were added.

### 3b. No photo proof
The rulebook and all challenge cards require photo evidence. The app currently says "take a photo" but has no capture/display mechanism. Options:
- Native `<input type="file" accept="image/*" capture="environment">` — zero dependencies, works on all mobile browsers, stores Base64 in `localStorage` (risky for storage quota).
- Display-only: just trust the player; keep it physical like the card game. This is the simplest and most robust approach for a local-first game.

### 3c. No coin flip for Curse of the Coin
The curse says "flip a coin in the app" but the app just shows the text. Needs a simple binary random result shown in the curse modal.

### 3d. No ghost station reference
Curse of the Ghost requires players to identify former Geisterbahnhöfe. No in-app reference exists. A short hardcoded list (Nordbahnhof, Potsdamer Platz U2, Stadtmitte U6 / Unter den Linden direction, etc.) in the curse modal would help.

### 3e. Curse tracking (active curses)
When a curse is thrown, its effect persists for N turns or N minutes. The app logs the curse but never displays "you are currently under: Curse of the Slowpoke (18 min remaining)". The Runner has no at-a-glance view of active curse obligations.

### 3f. Hunter-side tail visibility
The Hunter device cannot see which stations are in the tail (by design — Hunters don't know until "Reveal Tail" is used). But the current implementation uses the same shared `localStorage` state — meaning a Hunter on the same device who switches the role toggle can see everything. This is a local-only limitation but worth documenting.

### 3g. Ouroboros check is passive
The app offers Ouroboros when the Runner taps their start station. But it doesn't proactively alert the Runner when an Ouroboros opportunity appears, and it allows the Runner to miss it. Should flash a notification in the sidebar.

### 3h. No snake path visualization
Tail stations turn green on the map, but there's no polyline drawn along the snake's actual path — just dots. A path overlay would make it much easier to see the shape of the snake.

---

## 4. Design Priorities

### Priority 1 — Bug fixes (must fix before play)
- B1: Fix S41 circular adjacency (drop the duplicate terminal stop from the data or handle it in `buildGraph`)
- B3: Initialize `state.log = []` in `defaultState()`
- B4: Correct box-in detection logic
- B2: Guard LEAP suggestion with `state.powers.leap` check

### Priority 2 — Core gameplay improvements
- Add S42 to `LINE_DATA` (reverse of S41, same stops)
- Add S-Bahn spokes (at minimum S1, S3, S5, S7, S9 which cover the main inner-city arteries)
- Active curse display panel (sidebar widget showing current curses and countdown)
- Coin flip in Curse of the Coin modal
- Ghost station reference list in Curse of the Ghost modal
- Proactive Ouroboros alert

### Priority 3 — UX polish
- Snake path polyline overlay (drawn in insertion order, color-faded from start to head)
- Re-roll button disabled unless `state.pendingChallenge` is set
- Mobile sidebar: make the bottom panel collapsible/tabbed so the map takes more vertical space
- Power button labels that show used state more clearly (already strikethrough, but a ✓/✗ glyph would help)

### Priority 4 — Nice-to-have
- Photo attachment (Base64 thumbnail stored per claim — accept storage limitations)
- Export/share game summary (JSON download or shareable text summary)
- Leaderboard view at game end (multi-game series scoring)
- Theming: swap `LINE_DATA` from a city picker screen

---

## 5. Architecture Decisions

### Single-file vs. multi-file
The current single-file approach is intentional and correct for this use case: players open it directly from the filesystem or a static URL. No build step. **Keep it single-file.** The data block (`LINE_DATA`, `CARDS`) can grow without pain.

### State management
`localStorage` with a single JSON blob is fine. The only issue is storage quota if photo thumbnails are added. Mitigation: cap photo size at 200px wide before storing, or skip photo storage entirely.

### Multi-device sync
Out of scope for v1. The natural extension is a small Firebase Realtime Database or Supabase realtime subscription — the `state` object maps cleanly to a single document. A sync layer would need:
- A game ID (short random code shown at setup)
- Role-keyed write permissions (Runner writes tail/powers; Hunters write curses)
- Read access for all players

For now, document clearly that players coordinate over a group chat.

### Adjacency graph
Currently derived from sequential stops in each line array. This is correct but the S41 terminal duplicate must be removed. For multi-line stations, the adjacency is the union of all per-line neighbors — already handled correctly by `buildGraph()`.

---

## 6. Station Data Expansion Plan

### S42 (immediately actionable)
S42 is S41 reversed. Add as a separate entry in `LINE_DATA` with the same stops array in reverse order. The color is the same (`#C73C2E`). This adds no new stations — only new adjacency edges for the Ring.

### S-Bahn spokes (estimated ~120 additional stations)
Suggested order of addition by gameplay relevance:
1. **S5** (Ostbahnhof → Strausberg Nord, inner section: Ostbahnhof – Ostkreuz – Frankfurter Allee – Lichtenberg) — crosses the east Ring
2. **S7** (Potsdam – Ostbahnhof, inner: Wannsee – Charlottenburg – Zoo – Hauptbahnhof – Ostbahnhof) — major east-west trunk
3. **S1** (Oranienburg – Wannsee, inner: Gesundbrunnen – Nordbahnhof – Friedrichstraße – Brandenburger Tor – Potsdamer Platz – Südkreuz – Schöneberg) — north-south spine
4. **S9** (Airport – Ostbahnhof inner section) — southeast connector
5. **S3** (Erkner – Ostbahnhof – Hauptbahnhof – Zoo – Charlottenburg – Westkreuz) — east-west

Each line should be added as an ordered stop array. Transfer stations (e.g., Hauptbahnhof, Ostbahnhof, Zoo) already exist on U-Bahn lines and will automatically gain S-Bahn line membership in `STATIONS[name].lines`.

---

## 7. Mobile UX Redesign

Current mobile layout: sidebar collapses to a ~45vh scrollable panel above the map. This leaves only ~55vh for the map — barely usable on a phone.

**Proposed: bottom-sheet tab bar**
```
┌─────────────────────────────┐
│         MAP (full height)   │
│                             │
│                             │
│                             │
├─────────────────────────────┤
│ [Map] [Actions] [Log] [Info]│ ← tab bar
│   ← active tab content →   │ ← collapsible bottom sheet
└─────────────────────────────┘
```
- Map tab: just the map, no overlay (default view during travel)
- Actions tab: claim/curse/power buttons
- Log tab: game log
- Info tab: cadence timers, stats

This is achievable with CSS Grid + a small JS tab switcher — no external libraries needed.

---

## 8. Implementation Sequence

```
Phase 1 — Stability (1–2 hours)
  [x] Fix S41 loop adjacency bug (B1)
  [x] Initialize state.log in defaultState() (B3)
  [x] Fix box-in detection (B4)
  [x] Guard LEAP suggestion (B2)
  [x] Add S42 to LINE_DATA

Phase 2 — Gameplay completeness (2–4 hours)
  [ ] Active curse tracking sidebar panel
  [ ] Coin flip in Curse of the Coin
  [ ] Ghost station list in Curse of the Ghost
  [ ] Proactive Ouroboros alert in sidebar
  [ ] Snake path polyline overlay

Phase 3 — Data expansion (2–3 hours)
  [ ] Add S-Bahn spokes (S1, S3, S5, S7, S9)
  [ ] Verify transfer station coordinates against Leaflet map

Phase 4 — Mobile UX (2–3 hours)
  [ ] Bottom-sheet tab bar for mobile
  [ ] Re-roll button disabled guard
  [ ] Power button state glyphs

Phase 5 — Nice-to-have (open-ended)
  [ ] Photo thumbnail attachment
  [ ] Game summary export
```

---

## 9. Open Questions

1. **Should Hunters see the full map (including tail) on their device?** Current design: no — but same-device role switching breaks this. Do we want a true hidden-info mode for Hunters, or is the honor system fine?

2. **Photo proof**: is it needed in the app at all, or do players just show each other phones? Storing Base64 images in localStorage is fragile on older Android devices.

3. **S-Bahn scope**: should the app expand to the full Zone AB map, or stay Ring-centric for a tighter 3–4 hour game? Affects how much station data to add.

4. **Card re-roll cost**: the current re-roll deducts 1 challenge point. Should this be configurable at setup, or removed entirely (re-roll is free, just different card)?

5. **Multi-device sync**: is there appetite to add a small backend (Firebase/Supabase) for a future version, or is this always meant to be played with a single shared device or honor-based multi-device?
