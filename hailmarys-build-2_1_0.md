# Hail Mary's — Build Prompt: Alpha 2.1.0

**Date:** 2026-02-21
**Current version:** Alpha 2.0.0
**Target version:** Alpha 2.1.0
**Estimated time:** ~3–3.5 hours
**Estimated tokens:** ~70–90k

---

## SETUP

```bash
cd D:\TacFootball
cp tacfoot-rewrite.html tacfoot-rewrite-backup-2.0.0.html
```

Work in `tacfoot-rewrite.html`. Single file, React 18 + useReducer, no build tools.

---

## ITEMS (9 total: 3 bugfixes, 6 features)

### 1. BUG: Double-Offset Receiver Teleport

**What's happening:** When the player clicks Drop Back (or any QB action), receivers fly off the right side of the screen, then animate back to their correct positions over the next few ticks.

**Root cause:** In the `QB_ACTION` reducer case (~line 2623–2646), receiver positions are calculated as:
```javascript
newRecPos[rk] = {
    x: formation[rk].x + wp.x,
    y: formation[rk].y + wp.y
};
```
But `wp` is already an absolute coordinate — the formation offset was already added in `SELECT_PLAY` (lines 2213–2216). This double-adds the formation offset, sending W2 (who starts at x:92) to x:184 — way off screen. The `ANIMATION_TICK` then interpolates them back to the correct absolute position, which is why they "run back on screen from the right."

**Fix:** In the `QB_ACTION` case, find the receiver waypoint advancement block (~lines 2623–2646). Replace ALL FOUR coordinate assignments (two in the main branch, two in the clamp-to-last-waypoint branch):

Find:
```javascript
      // Advance receiver positions to next waypoint
      const receiverKeys = ['W1', 'W2', 'TE', 'RB'];
      for (const rk of receiverKeys) {
        if (state.receiverWaypoints[rk] && state.receiverWaypoints[rk][newPhase - 1]) {
          const wp = state.receiverWaypoints[rk][newPhase - 1];
          const formation = play.formation;
          if (formation[rk]) {
            newRecPos[rk] = {
              x: formation[rk].x + wp.x,
              y: formation[rk].y + wp.y
            };
          }
        } else if (state.receiverWaypoints[rk]) {
          // Clamp to last waypoint
          const wps = state.receiverWaypoints[rk];
          const lastWP = wps[wps.length - 1];
          const formation = play.formation;
          if (formation[rk] && lastWP) {
            newRecPos[rk] = {
              x: formation[rk].x + lastWP.x,
              y: formation[rk].y + lastWP.y
            };
          }
        }
      }
```

Replace with:
```javascript
      // Advance receiver positions to next waypoint
      // Waypoints are ALREADY absolute (formation offset added in SELECT_PLAY)
      const receiverKeys = ['W1', 'W2', 'TE', 'RB'];
      for (const rk of receiverKeys) {
        if (state.receiverWaypoints[rk] && state.receiverWaypoints[rk][newPhase - 1]) {
          const wp = state.receiverWaypoints[rk][newPhase - 1];
          if (play.formation[rk]) {
            newRecPos[rk] = {
              x: wp.x,
              y: wp.y
            };
          }
        } else if (state.receiverWaypoints[rk]) {
          // Clamp to last waypoint
          const wps = state.receiverWaypoints[rk];
          const lastWP = wps[wps.length - 1];
          if (play.formation[rk] && lastWP) {
            newRecPos[rk] = {
              x: lastWP.x,
              y: lastWP.y
            };
          }
        }
      }
```

---

### 2. BUG: Audible Re-Rolls Defense

**What's happening:** Player sees a blitz, audibles to beat it, but the new play select re-rolls the defense from scratch. The defense they audibled against disappears. This completely breaks the tactical loop — reading the defense and reacting to it is the entire point of audibling.

**Root cause:** The `AUDIBLE` handler (line 2264) sends the player back to `PLAY_SELECT` and clears `selectedPlayId`. When `SELECT_PLAY` fires for the new play choice, it calls `selectDefense()` fresh, ignoring the defense the player already read.

**Fix:** In the `SELECT_PLAY` case (~line 2196), check whether a defense is already chosen (meaning the player arrived here via audible). If so, keep it instead of re-rolling.

Find the line:
```javascript
      const defense = selectDefense(state.ballPos, state.down, state.dist);
```

Replace with:
```javascript
      // Preserve defense when audibling — the whole point is reacting to what you saw
      const defense = state.chosenDefense || selectDefense(state.ballPos, state.down, state.dist);
```

Also, in the `AUDIBLE` handler (~line 2264), make sure `chosenDefense` is NOT cleared. Currently it isn't explicitly cleared, but verify that the `PLAY_SELECT` phase transition doesn't reset it elsewhere. The `AUDIBLE` handler should preserve `chosenDefense` in its return state. Confirm `chosenDefense` survives through the audible flow: PRESNAP → AUDIBLE → PLAY_SELECT → SELECT_PLAY.

**CRITICAL — Prevent "Forever Defense" bug:** If `chosenDefense` persists via the audible fix above, it MUST be cleared at the end of every play. Otherwise one audible locks the same defense in for the rest of the game. Add `chosenDefense: null` to ALL play-ending state transitions:
- `NEXT_PLAY`
- `SCORE_TOUCHDOWN`
- `FOURTH_DOWN_OPTION` (punts and field goals)
- `POSTURE_PICK` (turnovers)
- Any other reducer case that ends a play and resets for the next drive/possession

Search the reducer for every place that resets `down: 1` or `runMoves: 0` to catch all of them.

**WARNING: `NEXT_PLAY` has ~8 separate `return { ...state` statements** (touchdown, safety 2P, safety 1P, turnover 2P, turnover 1P, first down, next down, general advance). `chosenDefense: null` must be added to EVERY return statement inside that case, not just the first one. Same applies to `FOURTH_DOWN_OPTION` which has multiple branches for punt, FG attempt, and go-for-it.

**Additional:** The defense label displayed on the PRESNAP screen after the audible should still show the same defense. Verify visually.

---

### 3. Defense Formation Plain English Names

**What's happening:** Defense names like "Nickel" and "Cover 2" appear on screen with no explanation. The target audience doesn't know what these mean.

**Design goal:** Real term stays visible (the game teaches football vocabulary) but always appears alongside a plain English cause-and-effect explanation. No football jargon inside the explanation.

**Update the 5 DEFENSES entries** (starting ~line 503). Add a `plain` field to each:

```javascript
{
  id:'base43', name:'Base 4-3',
  plain:'Four big rushers, three mid-field defenders. Balanced — no major weakness, no major strength.',
  desc:'Balanced defense with 4 down linemen and 3 linebackers.',
  ...
},
{
  id:'nickel', name:'Nickel',
  plain:'Five pass defenders instead of four. Harder to throw against, easier to run against.',
  desc:'Five defensive backs for pass coverage. Sacrifices a LB for a slot CB.',
  ...
},
{
  id:'blitz', name:'Blitz',
  plain:'Extra rushers coming after your quarterback. You have less time, but quick throws and screens can exploit the gaps they leave behind.',
  desc:'Six rushers! High pressure but vulnerable to quick throws and screens.',
  ...
},
{
  id:'cover2', name:'Cover 2',
  plain:'Two deep defenders splitting the field in half. Hard to throw deep, but the short middle is open.',
  desc:'Two deep safeties protect against the big play. Vulnerable to the run and short middle.',
  ...
},
{
  id:'goalline', name:'Goal Line',
  plain:'Maximum bodies packed at the line. Built to stop runs. Passing over the top can beat it.',
  desc:'Maximum run-stopping power. Packed front with eight in the box.',
  ...
}
```

**Display the `plain` text in these UI locations ONLY:**
- `PresnapScreen` UI panel (under the defense label, below the field)
- `DefensivePostureScreen`
- `CPUResultScreen` (if defense is mentioned)

**Do NOT put the `plain` text in the floating `defBadge` inside `GameField`.** That badge is a tiny absolutely-positioned box over the turf — multi-line explanations would stretch across the screen and cover ghost routes and player tokens. The `defBadge` keeps the short defense name only.

Format: Defense name in bold/prominent text, `plain` text directly below in smaller, lighter text. The player should never see a defense name without its explanation *in any UI panel*. The floating `defBadge` on the field is the one exception — it stays compact with the short name only.

---

### 4. Sprint Fatigue System

**What's happening:** Sprint is currently available unlimited times per play. No tension, no resource management.

**Design goal:** Two sprints per play. Creates a moment where the player burns their sprints, the button shows FATIGUED, and the crowd around them groans as defenders close in.

**State changes:**
- Add `sprintCount: 0` to initial state (~line 2011)
- Reset `sprintCount: 0` in the `SNAP` case (~line 2283) — every play goes through SNAP, so one reset there covers all play types. This is simpler and safer than adding it to the ~12 separate `runMoves: 0` locations scattered across SNAP_COMPLETE's trick play branches.

**Sprint limit logic:**
In the `RUNNER_MOVE` case (~line 3040), when processing a Sprint move:
- Increment `sprintCount` ONLY when `genericType === 'Sprint'`. Do NOT increment on Juke, Stiff Arm, Dive, Cut Outside, Hit the Hole, or any other move type.
- Cap: if `state.sprintCount >= 2`, Sprint should not be selectable (handled in UI)

**UI changes in the RUNNER move list rendering:**

**IMPORTANT: The `RunnerScreen` component (~line 5640) generates ALL move buttons in a single generic loop** over `moveLabels` (~line 5664). The Sprint fatigue styling must be applied INSIDE this existing loop by checking `ml.type === 'Sprint'` — do NOT create a separate Sprint button outside the loop.
- Sprint button shows a stamina indicator: `⚡⚡` when fresh, `⚡` after one use
- When `sprintCount >= 2`:
  - Button opacity drops to 0.3
  - Button is disabled (onClick does nothing / is removed)
  - Red translucent text overlays the button: **"FATIGUED"**
  - Style: `color: rgba(239, 68, 68, 0.7)`, `position: absolute`, centered over the button text, `fontSize: '13px'`, `fontWeight: 'bold'`, `pointerEvents: 'none'`
- The Sprint button should still be VISIBLE (not hidden) when fatigued — the player needs to see what they can't do anymore

---

### 5. Defender Speed Tuning — DECISION Phase (Stop the Swarm)

**What's happening:** Every defender — DL, DBs, LBs — starts moving at full speed immediately on snap during DECISION phase via ANIMATION_TICK. DL press at 1.0 yard/tick, DBs track at `dbTracking * 2` per tick, LBs drift at 0.5/tick. At 200ms intervals, DL are closing 5 yards/second from frame one. It looks like every defender is swarming in on the QB the instant the ball is snapped.

**Design goal:** DECISION = breathing room to learn and read. RUNNER = scary pursuit. Two different emotional beats. The pocket integrity system is the player's clock during DECISION — defenders shouldn't visually outrun it.

**Fix the ANIMATION_TICK DECISION/RPO_READ block** (~lines 3843–3933):

**DL press — tie to pocket integrity, not constant speed:**
- Instead of a flat 1.0 yard/tick, DL press speed should scale with pocket decay. When pocket is healthy (>70%), DL barely move (0.1–0.2/tick). As integrity drops to 50%, speed ramps to ~0.5/tick. Below 30%, full speed (1.0+/tick). This way the player SEES the pocket collapsing as it happens in the mechanics, not before.
- The ANIMATION_TICK already has access to `state.holdTimeData`. Calculate average pocket integrity from the hold time data and use it to scale DL speed: `dlSpeed = 0.2 + (1.0 - avgIntegrity) * 1.2` or similar curve.

**DBs — shadow, don't sprint:**
- During DECISION, DBs should track toward receivers at ~30–40% of current speed. They're covering, not pursuing. Replace `trackSpeed = dbTrack * 2` with something like `trackSpeed = dbTrack * 0.7` during DECISION phase.
- This means windows open and close gradually as routes develop, rather than DBs instantly jumping on the nearest receiver.

**LBs — hold position during DECISION:**
- Remove or drastically reduce the LB drift (currently 0.5/tick toward QB). LBs should hold their zone position during the passing phase. They drift ~0.1/tick at most.

**RUNNER phase stays fast:** The existing `advanceDefenders` function handles RUNNER pursuit and is working well per playtesting. Increase pursuit speed slightly if needed so defenders close within 2–3 ticks after a catch.

**Playtest targets:**
- In Practice/Preseason: pocket should survive ~4 phases comfortably. DL press is barely visible early, ramps up. Pursuit in RUNNER is noticeable but not overwhelming.
- In Regular: pocket ~3 phases. DL ramp faster. Pursuit is aggressive.
- In Playoffs: pocket ~2 phases. DL are coming fast. Pursuit is scary. Sprint is survival.

---

### 6. Title Screen — Game Description

Add a brief description to the title screen below the "Hail Mary's" logo. Something like:

> **A turn-based football strategy game.**
> Call plays. Read the defense. Make the throw.
> No football knowledge required.

Keep it short. Three lines max. The title screen should communicate instantly what this game is and that newcomers are welcome.

---

### 7. Title Screen — Feedback Link

Add a "Send Feedback" button/link to the title screen that opens a Google Form (URL TBD — use a placeholder like `https://forms.gle/PLACEHOLDER` that can be swapped later). Style it unobtrusively — smaller text, below the main buttons, not competing with Play/Settings.

---

### 8. Menu Button During Gameplay

Add a small menu/pause button visible during active gameplay phases (PRESNAP, DECISION, RUNNER, CONTACT, RESULT, etc.). Tapping it pauses play and offers:
- **Resume** — back to where you were
- **Restart Game** — fresh game, back to title
- **Quit to Menu** — back to title screen

**CRITICAL — Implement as modal overlay, NOT a phase change.** If Claude dispatches a `PAUSE` action that changes `state.phase`, the game will lose track of the active phase and "Resume" will be impossible. Instead:
- Add `isPaused: false` to the initial state
- The menu button dispatches `{ type: 'TOGGLE_PAUSE' }` which flips `isPaused`
- Render the pause menu as a modal overlay (semi-transparent dark background + centered card) when `isPaused === true`
- The ANIMATION_TICK `useEffect` should check `state.isPaused` and skip setting up the interval when paused. **CRITICAL: Add `state.isPaused` to the dependency array** (currently `[state.phase]` at ~line 6363). Without it in the dependency array, the effect won't re-evaluate when pause toggles and the tick interval keeps firing.
- The SNAPPING timer `useEffect` (~line 6322) and THROW_RESOLVE timer `useEffect` (~line 6344) should also check `state.isPaused` before setting their timeouts, and include `state.isPaused` in their dependency arrays. Otherwise a ball-in-flight resolves behind the pause overlay.
- "Resume" just flips `isPaused` back to false — the phase, positions, and all game state are untouched

Position: top-right corner, small enough not to interfere with gameplay. Use a hamburger icon (☰) or "MENU" text.

---

### 9. BUG: Follow Camera Half-Field

**What's happening:** When the game switches to follow camera mode (RUNNER, CONTACT), the field renders at a hardcoded 8px per yard. On most screens (~800–900px viewport), a 120-yard field at 8px/yard = 960px total — barely larger than the viewport. The scroll logic bottoms out and the lower half of the field is empty black space.

**Root cause:** Line 4164 uses `YARD_PX = 8` (hardcoded constant) for follow mode:
```javascript
const yardPx = state.cameraMode === 'strategic' ? (viewportHeight / 120) : YARD_PX;
```

**Fix:** Make follow camera dynamic based on viewport height. Show ~40 yards of field in the viewport for a proper zoom:

```javascript
const followYardPx = viewportHeight / 40;
const yardPx = state.cameraMode === 'strategic' ? (viewportHeight / 120) : followYardPx;
```

**Do NOT delete** the global `const YARD_PX = 8;` — `tokenScale` on line 4166 uses it as a baseline reference for scaling player icons.

---

## WHAT NOT TO CHANGE

- Field colors: `#2E8B3E` / `#3CAE4C`. Final. Do not touch.
- Existing play definitions, matchup matrix, or commentary lines.
- Camera system behavior (hybrid strategic/follow) — except the follow yardPx fix in item 9.
- Sound effects.
- Two-player mode logic (beyond what's needed for shared state resets).

## CRITICAL CODE STYLE RULE

**Write all new UI components using `React.createElement()` — the same style as the entire existing codebase. Do NOT use JSX syntax (`<div className="...">`) anywhere.** The file includes Babel standalone which can compile JSX, but mixing paradigms creates inconsistent, hard-to-maintain code. Every new element must use `React.createElement('div', { className: '...' }, children)`.

## VARIABLE REFERENCE

The prompt references game parameters. Here are the ACTUAL variable names in the codebase — do not invent others:
- **Pocket hold times:** `holdTimeMin` and `holdTimeMax` in `DIFFICULTY_PARAMS` (e.g., `regular: { holdTimeMin: 1, holdTimeMax: 4 }`)
- **Pocket integrity data:** `state.holdTimeData` — array of matchup objects with `{ holdTime, integrity, phasesExpired }`
- **Average pocket integrity:** Calculate from `state.holdTimeData` by averaging `integrity` values across all matchups
- **DB tracking speed:** `dbTracking` in `DIFFICULTY_PARAMS`
- **Pursuit speed:** `pursuitSpeed` in `DIFFICULTY_PARAMS`
- **Run moves counter:** `state.runMoves`
- **Defense chosen:** `state.chosenDefense`

---

## NOTES.MD UPDATE

After build is complete, append a new section to Notes.md:

```markdown
## Alpha 2.1.0 — [DATE]

### What Changed
1. **Double-Offset Teleport Fixed:** QB_ACTION no longer adds formation offset to already-absolute waypoints. Receivers move smoothly along routes during drop back.
2. **Audible Preserves Defense:** Audibling keeps the defense you read instead of re-rolling. chosenDefense cleared on all play-ending transitions to prevent "Forever Defense" lock-in.
3. **Defense Plain English Names:** All 5 defenses display cause-and-effect explanations alongside the real football term on PresnapScreen, DefensivePostureScreen, and CPUResultScreen. Not on the floating defBadge.
4. **Sprint Fatigue:** 2 sprints per play. ⚡⚡ indicator depletes with use. Exhausted sprint button dims and shows red "FATIGUED" overlay. Creates resource tension in RUNNER phase.
5. **Defender Speed Tuning — DECISION Swarm Fixed:** DL press now tied to pocket integrity (slow when pocket is healthy, fast when collapsing). DBs shadow at ~35% speed during DECISION. LBs hold position. RUNNER pursuit unchanged (fast and scary).
6. **Title Screen Description:** Game description added below logo. Communicates genre and newcomer-friendliness.
7. **Title Screen Feedback Link:** Google Form link for player feedback (placeholder URL).
8. **Menu Button:** Pause/menu as modal overlay (isPaused boolean, not phase change) with Resume, Restart, and Quit options. ANIMATION_TICK pauses when isPaused is true.
9. **Follow Camera Fixed:** Follow mode now uses dynamic viewport-based zoom (viewportHeight/40) instead of hardcoded 8px/yard. Field no longer cuts off at half screen.

### Backup
- `tacfoot-rewrite-backup-2.0.0.html` (pre-2.1.0)
```

---

## DONE SUMMARY

When complete, report in plain English:
- What changed (one line per item)
- Where files are saved
- Anything that needs manual attention (e.g., replacing the Google Form placeholder URL)
- Version number confirmed in code
