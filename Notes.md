# Hail Marys — Rewrite Notes

## File
- **Game:** `index.html` (single file)
- **Backup:** `tacfoot-rewrite-backup-1.0.0.html` (pre-1.5.0), `tacfoot-rewrite-backup-1.5.0.html` (pre-1.5.1), `tacfoot-rewrite-backup-1.5.1.html` (pre-1.5.2), `tacfoot-rewrite-backup-1.5.2.html` (pre-2.0.0), `tacfoot-rewrite-backup-2.0.0.html` (pre-2.1.0), `tacfoot-rewrite-backup-2.1.0.html` (pre-2.1.1), `index-2.2.1-backup.html` (pre-2.2.2), `index-2.2.2-backup.html` (pre-2.2.3), `index-2.2.3-backup.html` (pre-2.2.4), `index-2.2.4-backup.html` (pre-2.2.5), `index-2.2.5-backup.html` (pre-2.2.6), `index-2.2.6-backup.html` (pre-2.2.7)
- **Spec:** `TACFOOT-REWRITE-HANDOFF-v3.md`
- **Versioning:** Major.Minor.Patch (e.g. 1.5.0 = this build, 1.5.1 = bugfix, 2.0.0 = next feature build)

## Architecture
- React 18 + ReactDOM + Babel standalone via CDN
- `useReducer` with single state object (wrapped with cameraMode auto-setter)
- Camera mode (`strategic` / `follow`) stored as explicit state field
- CSS class-based transitions only
- Pure functions for all game logic

---

## Alpha 2.2.7 — Break Tackle Rebalance, Route Breaks & QoL (2026-02-23)
- BALANCE: Break tackle math rebalanced — Juke 54% (was 82%), Spin 48%, Stiff Arm uses STR/18. Multiplicative degradation: 1st hit full, 2nd hit 40%, 3rd hit 5%.
- BALANCE: DB route break penalty — DBs track at 50% speed for one phase when their assigned receiver makes a sharp lateral cut (simulates hip flip)
- BALANCE: Contact threshold increased from 1.5 to 2.0 yards — tackles now fire when tokens visually meet, not after overlap
- FIX: resolveContact now returns breakChance/fumbleChance in all return paths — Contact screen shows real percentages instead of "Break 0%"
- UI: Removed "Run Upfield" from WR/TE runner moves — now 4 buttons (Sprint, Cut Outside, Juke, Dive Forward) in clean 2x2 grid
- UI: Defensive Posture now has 1.5s "Opponent driving..." pause before CPU result appears (CPU_DRIVING phase)
- UI: Look Left/Right buttons only appear on deep route plays (Deep Post, Four Verticals, TE Seam, Play Action, Flea Flicker)
- CAMERA: Ball flight now tracks midpoint between QB and receiver instead of snapping to target
- CAMERA: Bottom clamp adds 15% viewport padding for non-overlay phases — backfield visible on mobile
- FIX: Preseason difficulty now shows teaching overlays (was gated to Practice only)
- UI: Touchdown fireworks upgraded to 5 bursts × 18 particles with larger sizes. Confetti removed.

---

## Alpha 2.2.6 — Pursuit Math & Phantom Tackle Fix (2026-02-23)
- BUGFIX: Defender pursuit speed now scales by carrier movement distance per turn (was flat 0.82-1.0 yards regardless of carrier sprint distance — created treadmill effect where defenders appeared frozen)
- BUGFIX: Removed maxYacMoves phantom tackle — play no longer force-ends after 3 YAC moves regardless of defender position. Tackles now happen organically when a defender gets within 1.5 yards.
- UI: YAC move counter no longer shows "/3" limit

---

## Alpha 2.2.5 — Gameplay Fixes + Note to Players (2026-02-23)
- BUGFIX: RPO_READ and FOURTH_DOWN now render as overlay phases (buttons float over field, camera offset applied)
- BUGFIX: Field proportions — max-height 600→700px, calc(100vh-180px)→calc(100vh-120px). Field looks like football field now.
- BUGFIX: Camera tracks carrier higher during RUNNER/CONTACT (offset 0.45 vs 0.35 for other overlays)
- BUGFIX: Zero fumbles in Practice and Preseason (multiplier set to 0)
- BUGFIX: Run plays vs Blitz matchup matrix corrected (was +1/+2, now -1/0 for most runs — blitz stacks the box)
- BUGFIX: Quick pass RPO auto-resolves for 3-6 yards instead of entering full YAC mode
- REMOVED: "Scramble lane right/left" label (meaningless to target audience)
- NEW: Note to Players screen accessible from pause menu with project info and Google Form feedback link

## Alpha 2.2.4 — Run Game & Camera Hotfix (2026-02-23)
- BUGFIX: RPO handoff teleported RB to LOS (y:0) instead of backfield position (y:formation.RB.y)
- BUGFIX: ANIMATION_TICK was advancing defenders every 200ms during turn-based RUNNER phase
  - Defenders now frozen during RUNNER; movement only on player button press (RUNNER_MOVE)
- BUGFIX: Stale viewport measurements after phase change
  - Added state.phase to measurement useEffect dependency array
  - Double requestAnimationFrame ensures CSS reflow before measuring
- BUGFIX: Inline styles on overlay buttons prevented CSS media queries from compressing on mobile
  - Removed hardcoded padding/fontSize from button inline styles
- Version strings updated to v2.2.4

---

## Alpha 2.2.3 — Layout Fixes + TD Celebration + Name Fix (2026-02-22)
- BUGFIX: Body scroll locked — changed html,body height:100% to min-height:100vh
- BUGFIX: Field viewport too tall on non-overlay phases, pushing panels off-screen
  - Base: calc(100vh - 180px), max-height 600px
  - Overlay phases (.overlay-active): calc(100vh - 60px), max-height 900px
- BUGFIX: Camera bottom clamp prevented LOS from reaching top of screen when backed up
  - Added overlay-phase clamp expansion (+ 60% viewport height of extra scroll room)
  - Camera offset bumped from 0.2 to 0.35
- DECISION panel compressed from 6 rows to 4:
  - Removed "QB DECISION" header row
  - Receiver cards forced to single row of 4 (was 2x2 grid)
  - Movement buttons (4) and Look buttons (2) merged into single row of 6
  - Escape buttons (3) tightened
  - Overlay max-height reduced from 60% to 50%
- RUNNER/CONTACT button padding reduced from 12px 8px to 8px 4px for overlay fit
- TD celebration upgraded:
  - Gold screen flash on entry
  - 3 staggered firework bursts (left/center/right) replacing single burst
  - 35-piece confetti rain with flutter/tumble animation
  - Score countup +0 to +7 with pop at end
  - Bigger text bounce (scale 1.4 overshoot)
- Renamed "Hail Mary's" to "Hail Marys" (no apostrophe) everywhere
- Version strings updated to v2.2.3

---

## Alpha 2.2.2 — 2026-02-22

### What Changed
1. **Overlay Panel Layout:** DECISION, RUNNER, and CONTACT phases now render their panel content as an overlay on the bottom of the field viewport instead of below the field in document flow. Uses absolute positioning with a gradient fade background (`overlay-panel` class). Non-overlay phases (MENU, PLAY_SELECT, PRESNAP, RESULT, etc.) unchanged.
2. **Compact Overlay Styling:** Receiver cards (`.throw-target`, `.tgt-name`, `.tgt-detail`) and buttons are smaller inside the overlay panel. Additional mobile compression at `max-width: 600px`.
3. **Field Viewport Expansion:** `.field-viewport` height changed from `calc(100vh - 140px)` to `calc(100vh - 60px)`, min-height 320→400px, max-height 700→900px. Field fills more of the screen since the panel is overlaid, not stacked.
4. **HUD Collision Fix:** Down & distance HUD (`downDistHud`) moves to `top: 0` during DECISION/RUNNER/CONTACT phases so it's not hidden behind the overlay. Stays at `bottom: 0` during all other phases.
5. **Camera Framing for Overlay:** Camera offset changed during overlay phases — LOS positioned at 20% from top of viewport instead of centered. Keeps the action visible above the overlay panel.
6. **Primary Receiver Highlight Fix:** `primaryReceiver` now compares against `play.primary` (the play's designed target) instead of `bestKey` (most-open receiver). Gold border and "INTENDED" badge now correctly display.
7. **Coach Advice References Primary:** For pass plays with a primary target, coach advice now mentions the receiver name (e.g., "This play targets W1."). Joey and Jim have distinct phrasing. Run plays and `primary: null` still get generic advice.
8. **Difficulty Label on Scoreboard:** Difficulty name (e.g., "PRESEASON") displayed next to version string in the scoreboard.
9. **Version Strings:** Updated to v2.2.2 in both scoreboard and title screen.

---

## Alpha 1.5.0 — 2026-02-21

### What Changed
1. **Renamed:** "Tactical Football" → "Hail Mary's" everywhere (title, scoreboard, title screen, midfield logo)
2. **Hybrid Camera System:** Two camera modes controlled by `state.cameraMode`
   - `strategic` mode: entire 120-yard field (incl. endzones) fits in 450px viewport. Dynamic `yardPx = 450/120 = 3.75`. Used for MENU, PLAY_SELECT, PRESNAP, RESULT, FOURTH_DOWN, DEFENSIVE_POSTURE, CPU_RESULT, and all overlay phases.
   - `follow` mode: zoomed in at 8px/yard. Camera tracks ball carrier or ball-in-flight (not LOS). Used for SNAPPING, DECISION, RPO_READ, RUNNER, CONTACT.
   - Reducer wrapper auto-sets `cameraMode` on every phase transition.
   - CSS transition (0.4s) animates zoom between modes. Instant cam only within strategic-to-strategic changes.
   - Tokens, ball scaled proportionally via `tokenScale = yardPx / YARD_PX`.
3. **Ghost Route Fix:** Double-offset bug fixed. Waypoints stored as absolute in SELECT_PLAY were being offset again in the renderer. Now renderer uses `wp.x`/`wp.y` directly.
4. **CPU Result Interstitial:** New `CPU_RESULT` phase between POSTURE_PICK and MENU. Full-screen overlay card showing drive result headline, updated score, Dan+Kiki commentary, coach debrief, and CONTINUE button. Preserves quarter-change/halftime logic via `cpuResultNextPhase`.

### Fallbacks Taken
- None — hybrid camera shipped as designed.

---

## Alpha 2.2.1 — 2026-02-22

### What Changed
1. **Sound Fixes (3):** Added `SoundEngine._ensureCtx()` to 2-Player Mode button onClick. Added silent oscillator unlock in `_ensureCtx()` for Safari/iOS compatibility. Moved `SoundEngine.play('snap')` directly into SNAP button onClick for user-gesture context.
2. **Route Waypoint Clamping:** Waypoints clamped to `x: [2, 98]` and `y: max 110` in both SELECT_PLAY and ghost route preview. Prevents routes/tokens from extending past sidelines.
3. **Field Stripe Colors:** Dark stripe changed from `#38A84A` to `#42B554`. Narrower contrast gap between even/odd bands.
4. **Commentary Auto-Dismiss Fix:** Fixed useEffect condition from `!state.showCommentary` to `state.showCommentary` so the 2750ms dismiss timer actually fires.
5. **TD vs OOB Priority:** Touchdown check moved before out-of-bounds check in RUNNER_MOVE. Pylon dives now score instead of being ruled OOB.
6. **DB Assignment-Based Coverage:** Greedy proximity algorithm assigns each DB to a specific receiver at snap time. Man-assigned DBs track their receiver; extra DBs (nickel) play zone (drift toward QB x, hold depth). Eliminates double-coverage stacking bug.
7. **Preseason Fumble Multiplier:** Reduced from 0.30 to 0.10. First-contact fumbles now ~0.5%.
8. **Contact Limit (Max 2) + Dive Ends Play:** Auto-tackle after 2 broken contacts (no 3rd CONTACT screen). `resolveContact` returns `'tackle'` for DiveForward (was `'break'`), ending the play with safe 1-3 yards.
9. **DL Speed Formula Fix:** Fixed pocket integrity scale from 0-1 to 0-100 (`avgInt / 100`). DL no longer fly backward at snap.
10. **Version Strings:** Updated to v2.2.1.

### Backup
- `index-2.2.0-backup.html` (pre-2.2.1)

---

## Alpha 2.2.0 — 2026-02-22

### What Changed
1. **Decision Phase Now Turn-Based (Critical Fix):** Removed all gameplay movement from ANIMATION_TICK during DECISION/RPO_READ phases. The field is now completely frozen between player actions. Each button press advances one step of movement for QB, receivers, DBs, DL, and LBs via the QB_ACTION handler. DL press (0.8 + 3.0 scaling by pocket decay) and LB drift (0.4y per action) added to QB_ACTION. RUNNER/CONTACT phases remain real-time on the 200ms timer. `tickPhases` array reduced to `['RUNNER', 'CONTACT']`.
2. **Play Again Preserves Settings:** RESTART_GAME now preserves coach, difficulty, and twoPlayerMode instead of wiping to full INITIAL_STATE. Returns to MENU phase (play selection) instead of TITLE.
3. **AudioContext Unlock:** Added `SoundEngine._ensureCtx()` calls in the START_GAME difficulty button and SNAP button onClick handlers — both are user gesture contexts where browsers allow AudioContext creation.
4. **Version Strings Updated:** Scoreboard → "HAIL MARY'S v2.2.0", title screen → "ALPHA v2.2.0".
5. **Brighter Field Stripes:** Yard band colors changed from `#2E8B3E`/`#3CAE4C` to `#38A84A`/`#4CC95C` — brighter, more saturated stadium greens.

### Backup
- `index-2.1.1-backup.html` (pre-2.2.0)

---

## Alpha 2.1.1 — Hotfix (2026-02-22)
- Follow camera zoom adjusted from /40 to /55 (less jarring, more field visible)
- WR/TE carriers get "Run Upfield" move (HitTheHole type) — Sprint fatigue no longer a dead end
- Defense tokens (Base 4-3) now visible on field during MENU/PLAY_SELECT
- Jim emoji changed to 🦊 (fox) — overridden by SVG portrait
- Game Over "Play Again" uses RESTART_GAME dispatch instead of page reload
- Sound engine activated: snap, tackle, whistle+crowd on TD, sackBuzz
- Coach SVG portraits on title screen (Jim: fat, walrus mustache, headset; Joey: sunglasses, smirk, earpiece)

---

## Alpha 2.1.0 — 2026-02-22

### What Changed
1. **Double-Offset Teleport Fixed:** QB_ACTION no longer adds formation offset to already-absolute waypoints. Receivers move smoothly along routes during drop back.
2. **Audible Preserves Defense:** Audibling keeps the defense you read instead of re-rolling. chosenDefense cleared on all play-ending transitions to prevent "Forever Defense" lock-in.
3. **Defense Plain English Names:** All 5 defenses display cause-and-effect explanations alongside the real football term on PresnapScreen. Not on the floating defBadge.
4. **Sprint Fatigue:** 2 sprints per play. ⚡⚡ indicator depletes with use. Exhausted sprint button dims and shows red "FATIGUED" overlay. Creates resource tension in RUNNER phase.
5. **Defender Speed Tuning — DECISION Swarm Fixed:** DL press now tied to pocket integrity (slow when pocket is healthy, fast when collapsing). DBs shadow at ~35% speed during DECISION. LBs hold position. RUNNER pursuit unchanged (fast and scary).
6. **Title Screen Description:** Game description added below logo. Communicates genre and newcomer-friendliness.
7. **Title Screen Feedback Link:** Google Form link for player feedback (placeholder URL).
8. **Menu Button:** Pause/menu as modal overlay (isPaused boolean, not phase change) with Resume, Restart, and Quit options. ANIMATION_TICK pauses when isPaused is true.
9. **Follow Camera Fixed:** Follow mode now uses dynamic viewport-based zoom (viewportHeight/40) instead of hardcoded 8px/yard. Field no longer cuts off at half screen.

### Backup
- `tacfoot-rewrite-backup-2.0.0.html` (pre-2.1.0)

---

## Alpha 2.0.0 — 2026-02-21 (15 Surgical Bug Fixes + Animation Tick System)

### What Changed
1. **xScale Vector Fixes (3):** `advanceDefenders` pursuit, DB tracking in QB_ACTION, and `generateRunArrows` dot product all now use xScale correctly. Defenders track at proper angles instead of bunching horizontally.
2. **Phantom Tackles Fixed:** Contact only triggers when nearest defender is actually within 1.5 yards. Failed run moves no longer auto-trigger contact.
3. **HIT/INT Display Fixed:** Target cards now correctly read `throwResult.details.completionPct` and `throwResult.details.intChance` instead of non-existent top-level properties.
4. **Contact Move Rates Fixed:** Runner screen success percentages now read from `runResult.rate` (the actual field) instead of `runResult.successChance`.
5. **QB Keeper vs Blitz Nerfed:** keepBase drops to 25% when defense is blitz. QB can't cheese blitz with a keep.
6. **INTENDED Receiver Fixed:** Gold "INTENDED" badge now highlights the most open receiver dynamically (using `bestKey`) instead of the play's static `primary` field.
7. **Post-TD CPU Field Position Fixed:** `cpuBallPos` changed from 25 to 75 after TDs and FGs. CPU now correctly starts at their own 25, not yours.
8. **Halftime Reset:** Entering Q3 resets ballPos to 25, down to 1, dist to 10. Two-player mode flips possession.
9. **Turnover 80-Yard Cap Removed:** Changed all `Math.min(..., 80)` to cap at 99 with endzone touchback routing at 80. Turnovers deep in enemy territory no longer teleport the CPU forward.
10. **Safety Logic (6 locations):** Safeties now trigger in RUNNER_MOVE, CONTACT_MOVE, intentional grounding, QB tuck, auto-sack, and pressure sack when ball carrier is in own endzone. Awards 2 points to defense and triggers possession change via NEXT_PLAY.
11. **Follow Cam Narrowed:** FOLLOW_PHASES reduced to `['RUNNER', 'CONTACT']`. DECISION/RPO_READ/SNAPPING now stay in strategic (zoomed-out) view so players can see the full field during reads.
12. **Animation Tick System:** 200ms `setInterval` during DECISION/RPO_READ/RUNNER/CONTACT dispatches ANIMATION_TICK. Receivers advance along waypoints, defenders track, DL press toward QB, OL shift with pocket decay, carriers move, ball interpolates in flight. CSS transitions smooth between ticks.
13. **Field Colors Finalized:** `#2E8B3E` / `#3CAE4C` — darker, richer greens.

### Backup
- `tacfoot-rewrite-backup-1.5.2.html` (pre-2.0.0)

---

## Alpha 1.5.2 — 2026-02-21 (Quick Visual Fix Pass)

### What Changed
1. **Field Color:** Turf bands brightened to vivid cartoon greens (`#4EC24E`/`#5CD65C`). Retro Bowl energy, not ESPN documentary.
2. **Never an Empty Field:** Added fallback formation rendering for all non-active phases. MENU/FOURTH_DOWN show offense in I-Formation. PLAY_SELECT shows the formation of the expanded play card. RESULT shows all players at their final positions from the completed play. The field is never empty when visible.
3. **Field Width Measurement Fix:** `fieldWidth` now initializes to `0` instead of `400`. Measurement uses `requestAnimationFrame` to ensure layout is complete before reading dimensions. Tokens don't render until measurement is done, preventing the half-field bunching bug where W2 would land at pixel 368 instead of 690.
4. **Ghost Routes During Play Browsing:** When a play card is expanded during PLAY_SELECT, ghost routes are generated on-the-fly and rendered on the field, even before a play is selected. Routes update as the user browses different plays.

### Backup
- `tacfoot-rewrite-backup-1.5.1.html` (pre-1.5.2)

---

## Alpha 1.5.1 — 2026-02-21 (Bugfix Pass)

### What Changed
1. **Field Height:** Viewport now uses `calc(100vh - 140px)` instead of fixed 450px. Field takes 60-70% of screen. `viewportHeight` is measured dynamically from the DOM element.
2. **Field Color:** Turf bands brightened from dingy olive (`#2d5a1e`/`#326320`) to broadcast-TV green (`#3a8c2e`/`#44a035`).
3. **Strategic Mode Token Fix (Strike 2):** Fixed QB token losing its rotation (CSS `transform: rotate(45deg)` was being overridden by inline style). Rotation now applied in inline transform alongside scale. All coordinate conversions confirmed using dynamic `yardPx`. Yard number font reduced from 24px to 14px. Endzone text reduced from 28px to 16px.
4. **Back Button Fix:** "Back" from play selection no longer produces a dead screen. `SELECT_CATEGORY` with `null` category now transitions to `MENU` phase instead of `PLAY_SELECT`.
5. **Ghost Route Visibility:** Opacity increased from 0.5/0.3 to 0.8/0.5. Stroke width from 2 to 3. Colors changed to vivid cyan (`rgba(0,220,255,0.9)`) for primary routes and brighter blue for decoys.
6. **Coach Language Rewrite:** All coach advice lines (Joey + Jim, all 5 categories each) rewritten to plain English. No jargon. Every line understandable by someone watching their first football game.
7. **RPO Read Text:** "NO ROOM OUTSIDE" replaced with actionable descriptions: "GAP IS OPEN!", "STACKED BOX", or "MIDDLE IS CLOGGED" with specific advice on what to do. RPO success percentages now vary by gap status.
8. **Run Success + Difficulty:** `calculateRunSuccess` now accepts difficulty parameter. Practice gets +20% run success bonus, Preseason +12%, Playoffs -5%. Runs should no longer be universally stuffed in easier modes.
9. **QB Keeper Nerf:** QB carriers now get -15% break chance and +3% fumble base on all contact moves. QBs are no longer better runners than RBs.
10. **RPO Box Count Fix:** Gap-open threshold changed from `<= 5` to `<= 6`. Previously EVERY defense registered as "no room" (all have 6-8 in box). Now Nickel and Cover 2 (6 in box) correctly read as neutral/open.

### Camera Hybrid Status
- **Strike 2 passed.** Hybrid camera survives to next build.

### Remaining Attention Items
- Route data may still need tuning (shapes/angles) — wait for visual review after field height fix
- Play grid `max-height` reduced to 200px to fit alongside taller field
- Token scaling in strategic mode works but tokens are small; may need UX review

---

## What's Built
- 17 game states with full transition logic
- 18 plays (7 run, 8 pass, 3 trick) with formations, routes, RPO flags
- 5 defenses with exact positions and weighted situational selection
- 4 difficulty levels with every spec parameter
- 90-value matchup matrix (18 plays × 5 defenses)
- Pocket integrity engine (hold times, decay, left/right bars, sack mechanics)
- Throw resolution (completion formula, INT rates, drops, difficulty modifiers)
- Contact system (Spin/Juke/Stiff Arm/Dive Forward, degradation, fumble escalation)
- Runner Mode (position-specific moves, pursuit escalation, OOB detection, run arrows)
- RPO Read with gap-open probability and quick pass single-roll
- YAC system (3 tiers at 3/6 yard thresholds)
- Field goal resolution (distance formula, step-function success rates)
- Field position rules for every transition type
- Defensive Posture (CPU scoring odds, posture modifiers, turnover spots)
- Intentional grounding (tackle box check, difficulty scaling)
- Commentary (Dan + Kiki, 14 categories)
- Coach system (Baby Boy Joey + Grizzled Jim)
- Practice overlays (teaching cards)
- Two-player mode (coin toss, 180° defense select, possession order)
- Sound effects (Web Audio API synthesized)
- 8 CSS keyframe animations
- Full field rendering (turf, yard lines, hash marks, end zones, goal posts, tokens)

## Fingerprint Test
Passed. Zero banned variable names found:
qbP, bcP, defSch, opn, recSeeds, blitzUncovered, camYRef,
noTransition, offP, OL_IDS, pLR, OL_BLK, isPre, fmtTarget,
ndInfo, dlS, blS — none appear anywhere in the file.

## Playtest Attention Items
- Route waypoint coordinates may need visual tuning on field
- DB tracking and pursuit escalation rates are spec defaults marked [TUNE]
- SNAPPING phase timer (setTimeout → SNAP_COMPLETE) — verify in browser
- Commentary variety is 4-5 lines per category; could be expanded
- Two-player possession ordering should be tested across a full game
- All [TUNE] values are implemented as written and ready for adjustment
