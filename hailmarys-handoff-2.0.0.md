# Hail Mary's — Thread Handoff: Alpha 2.0.0

**Date:** 2026-02-21
**Current version:** Alpha 1.5.2
**Next version:** Alpha 2.0.0

---

## WHAT THIS GAME IS

Hail Mary's (formerly TacFoot / Tactical Football) is a turn-based football strategy game. Single HTML file, React 18 + useReducer, no build tools. Target audience is "girlfriends learning football" — people who don't know what a cornerback is but want to have fun calling plays. Design filter is the "bar test": would this be fun to play on a phone at a bar?

18 plays (7 run, 8 pass, 3 trick), 5 defenses, 4 difficulty levels (Practice/Preseason/Regular/Playoffs). RPO system, contact resolution, two-voice commentary (Dan play-by-play, Kiki color/analysis). Preseason = heroic/offense-weighted, Playoffs = realistic.

---

## TWO COACHES

**Jim "The Old Fox"** — ~70, walrus mustache, heavy build. Andy Reid model. "Been doing this 40 years" energy. Steady, nurturing, consistent. Built and active in-game. Q4 boost. Shakes off bad plays.

**Joey "The Gunslinger"** — ~43, thin, rakish, cocky. Visor (not full cap), sunglasses pushed up on visor or worn during game, headset with mic boom, slim build, sharp jawline. Fitted polo or quarter-zip. Arms crossed default. "23-year-old girlfriend energy" coach. Higher ceiling, lower floor — dominates when ahead, frustrated/undisciplined when behind. Designed but not yet mechanically differentiated (that's 2.3.0).

---

## FILES FOR THIS THREAD

Upload all four of these:

1. **This handoff doc** — project status, roadmap, context
2. **gemini-fixes-consolidated.md** — 15 surgical bug fixes with exact line numbers and old/new code blocks
3. **tacfoot-rewrite.html** — the current game (Alpha 1.5.2)
4. **Notes.md** — build history

---

## WHAT'S WORKING (Alpha 1.5.2)

- 17 game state phases with full transition logic
- All 18 plays with formations, routes, RPO flags
- 5 defenses with weighted CPU selection
- 4 difficulty tiers with distinct parameters
- Contact resolution system (spin, juke, stiff-arm, dive)
- Two-voice commentary (Dan + Kiki)
- Coach advice system (Jim and Joey, different voices)
- Two-player mode (basic)
- CPU drive resolution with defensive posture choice
- CPU Result interstitial screen (working as of 1.5.2)
- Touchdown celebration with fireworks
- Halftime, quarter change, game over screens
- Field goal attempts
- Fourth down decision screen
- Practice mode overlays/tutorials
- Ghost routes on play select/presnap
- Matchup badges (good/ok/bad vs defense)
- Audible system with per-quarter limits

## WHAT'S BROKEN (Known Issues)

### Bugs (addressed by Gemini fixes in 2.0.0):
- **Phantom tackles** — failed evasion RNG triggers contact with defenders 15+ yards away
- **xScale bugs (3)** — pursuit vectors, DB tracking, and run arrow pathfinding all miscalculate because X-axis percentages aren't converted to Y-axis yardage equivalents
- **HIT/INT display shows 0%** — reading wrong property path from throw result
- **Contact moves always show 50%** — reading wrong property name, falls back to default
- **QB Keeper too easy vs Blitz** — box count logic accidentally makes it better against blitz
- **INTENDED receiver static** — points to playbook primary instead of dynamic best option
- **CPU field position after TD** — CPU gets ball at player's 25 instead of their own 25
- **Turnover 80-yard cap** — CPU can never be pinned inside their own 20 on turnovers (5 locations)
- **Halftime doesn't reset** — field position, down, distance not reset for Q3
- **Follow cam during DECISION** — zooms in when player needs full-field view
- **No safety detection** — tackles/sacks in own endzone clamp to 1-yard line instead of scoring 2 points (6 insertion points: RUNNER_MOVE, CONTACT_MOVE, intentional grounding, QB tuck, auto-sack, pressure sack)

### Structural gap (addressed by animation system in 2.0.0):
- **No movement during play** — the rendering rewrite removed the rAF animation loop but didn't replace it. The game is a working football strategy engine rendering to static diagrams. Tokens teleport between positions. No visible: pocket collapse, ball flight, receiver separation, defender pursuit, DL press.

### Not in 2.0.0 (deferred):
- Play descriptions use jargon newcomers don't understand (2.2.0)
- Defensive posture screen doesn't explain WHEN to pick each option (2.2.0)
- Coach advice sometimes uses football jargon (2.2.0)
- Title screen missing game description and feedback form (2.1.0)

---

## RENDERING ARCHITECTURE

Pure React rendering via useReducer. No rAF animation loop. State changes trigger re-renders, CSS transitions on tokens (0.4s ease) interpolate position changes smoothly.

The field uses a coordinate system where:
- **X axis:** 0-100 (percentage across field width, left to right)
- **Y axis:** yards from player's own goal line (0 = own goal, 100 = opponent's goal)
- **xScale = 0.53** converts X percentages to Y-axis yardage equivalents for distance calculations

Field colors: `#2E8B3E` (dark band) / `#3CAE4C` (light band) — bright cartoon green, NOT realistic grass. This is settled. Do not change.

---

## ALPHA 2.0.0 BUILD — TWO PARTS

### Part 1: Apply 15 Gemini Surgical Fixes

All fixes are in `gemini-fixes-consolidated.md` with exact old/new code blocks and line numbers. Categories:

- **xScale Vector Bugs (3):** advanceDefenders pursuit, DB tracking, generateRunArrows double-scaling
- **Display/Logic Bugs (5):** phantom tackles, HIT/INT display, contact moves display, QB Keeper vs Blitz, INTENDED receiver targeting
- **Coordinate/Position Bugs (3):** post-TD CPU field position, halftime reset, turnover 80-yard cap (5 locations)
- **Safety Logic (6 insertion points):** RUNNER_MOVE, CONTACT_MOVE, NEXT_PLAY score resolution, intentional grounding, QB tuck, auto-sack + pressure sack
- **Follow Cam (1):** restrict to RUNNER and CONTACT only

### Part 2: Animation Tick System

Add a lightweight tick mechanism that fires every 150-250ms during active play phases and dispatches incremental position updates. This makes tokens visibly move on the field.

**What needs to move:**
- DL press toward QB during DECISION (pocket collapse visual)
- OL get pushed back as pocket integrity drops
- Receivers advance along route waypoints during DECISION
- Defenders track toward nearest receiver / ball carrier
- Ball flies from QB to receiver on throw (not teleport)
- Runner advances along run path during RUNNER phase
- Defenders converge on ball carrier during RUNNER/CONTACT

**Implementation approach:**
- useEffect in App component that sets up setInterval during active phases (DECISION, RUNNER, CONTACT, ball-in-flight)
- Each tick dispatches an ANIMATION_TICK action to the reducer
- Reducer updates position arrays (currentReceiverPos, currentDefenderPos, carrierXY, etc.) by small increments toward their targets
- CSS transitions on tokens (already 0.4s) smooth between ticks
- Cleanup interval on phase change

**What NOT to animate:**
- PRESNAP (static formation display)
- MENU, PLAY_SELECT (no active play)
- RESULT (final positions, static)

---

## VERSIONING

Major.Minor.Patch — e.g., 2.0.0 is this build, 2.0.1 would be a bugfix on it.

---

## FULL ROADMAP

**2.0.0 — Surgical Fixes + Animation** (this build)
- 15 Gemini fixes
- Animation tick system
- Field color hardcoded

**2.1.0 — Title Screen & Navigation**
- Game description, Google Form feedback link
- Coach select with character descriptions
- Menu button during gameplay
- Note to Players screen

**2.2.0 — Gameplay Tuning**
- Play descriptions rewritten in plain English (bar test every tooltip)
- Defensive posture coaching (coach explains WHEN to pick each, tied to score/quarter/field position)
- **Coach presnap advice must be matchup-aware.** Currently `pickCoachAdvice` gets called with just `'presnap'` — it doesn't receive the matchup rating, so it gives generic platitudes even when the screen says "BAD call." Fix: pass matchup rating into coach advice. When matchup is bad, coach names TWO specific alternative plays that have good/ok matchups against the current defense (pulled from `getMatchupRating` filtering the playbook). Format: "This blitz is going to blow up your play. Try an **Inside Run** or a **Quick Slant** — both get the ball out quickly before the blitzing players can reach the QB." Every suggestion includes the one-sentence mechanical reason WHY it works against this defense. Not football idioms, not "trust me" — the actual cause and effect. The girlfriend watching her first game should learn something about football from every coach line.
- Contact percentages vary by context
- Result screen references actual events
- More Dan/Kiki lines per category
- Coach advice quality pass (bar test every line — no jargon, no "check down if nothing's there")
- Wider audible suggestions per defense, tendency counter, Look decay
- Verify route shapes with fixed field proportions
- Verify camera scroll + YAC tiers from 14.7

**2.3.0 — Coach Mechanical Differentiation**
- Joey: dominant ahead, frustrated behind, forces big plays, higher ceiling/lower floor
- Jim: steady regardless of score, Q4 boost, consistent

**3.0.0 — More Trick Plays**
- Philly Special, Hook and Lateral, Double Reverse, Halfback Pass, Fumblerooski, Hail Mary
- Each mechanically distinct, not reskins

**4.0.0 — Two-Player V1**
- Both pick plays blind, one audible per half
- Regular = offense-weighted, Playoffs = realistic

**5.0.0 — Defense V1**
- One post-snap choice: commit to run or stay in coverage
- Audibles return (defense can respond)

**6.0.0 — Defense V2**
- Pressure decision: send rusher or hold coverage
- Strip/pursue choices during YAC/contact

**7.0.0 — Game Clock & CPU Offense**
- Timeouts, two-minute drill
- CPU scores too
- Hot routes

**8.0.0 — Defense V3 (Full)**
- Fake blitz, delayed blitz, spy, bracket coverage
- Zone vs man toggle, defensive trick plays

**9.0.0 — Season/Meta**
- Season mode, player stats, fatigue
- Drive summary, playbook study

**10.0.0 — The Meleficent**
- Melissa's 92-yard field goal Easter egg
- 15+ random failure messages, pity mechanic, secret cheat code for Kip to trigger success

**Unscheduled:** GitHub Pages/Netlify deploy, player-submitted trick plays, instant replay, Announcer's Booth broadcast graphics

---

## CRITICAL RULES FOR CLAUDE

- **ABSOLUTE RULE:** NEVER write a build prompt until Kip explicitly says "write the prompt" or "go" or "let's build." No exceptions. This is Kip's #1 frustration.
- Build prompts must be downloadable .md files. Include time/token estimate, backup, NOTES.md update, DONE summary.
- Never use the phrase "scratches the itch" or variations.
- Never end replies with follow-up questions.
- Don't give directives/orders. Say "you don't need to" not "don't." Suggest, don't command.
- Suggest new thread when conversation gets long.
- **THREE STRIKE RULE:** If a bug fix fails twice, stop. Recommend structural rewrite. Never argue against a rewrite Kip requests — his instinct has been proven right.
- Field colors are #2E8B3E / #3CAE4C. This is final. Do not fight Kip on aesthetic choices.
- Workflow: design in chat, build in Claude Code (cd D:\TacFootball, claude --dangerously-skip-permissions).
- Files: thumb drive D:/E:/F:\TacFootball\. Home PC = Lash2, F: drive. GitHub: kpurdddy/TacFootball (public, master). Copy to thumb drive only, no desktop copies.
