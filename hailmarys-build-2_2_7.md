# Hail Marys — Build Prompt: Alpha 2.2.7

**Estimated time:** ~8 minutes
**Estimated tokens:** ~20-25k

## SETUP

```bash
cd D:\TacFootball
cp index.html index-2.2.6-backup.html
```

Work in `index.html`. Single file, React 18 + useReducer. All rendering uses `React.createElement()`, not JSX. Update the version string from `v2.2.6` to `v2.2.7`.

---

## FIXES (10 items)

### 1. Break Tackle Rebalance — `resolveContact()`

The current break chance math is massively overpowered. A WR1 with skill 8 gets 82% break chance on Juke (`skill / 12 + 0.15`). With -15% flat degradation per tackle, third hit is still 52%.

**Change the math to:**
- Spin: base = `attrs.skill / 20 + 0.08` (WR1 skill 8 = 48%)
- Juke: base = `attrs.skill / 18 + 0.10` (WR1 skill 8 = 54%) — best option but still coin-flip
- Stiff Arm: base = `attrs.str / 18 + 0.08` (WR1 str 4 = 30%, RB str 7 = 47%)
- Default: 0.20

**Change degradation from flat subtraction to multiplicative:**
Replace `const breakChance = Math.max(0.05, breakBase - (tacklesBroken * 0.15));` with:
```
const degradeMultipliers = [1.0, 0.4, 0.05];
const degradeIdx = Math.min(tacklesBroken, 2);
const breakChance = Math.max(0.02, breakBase * degradeMultipliers[degradeIdx]);
```

This produces: WR1 Juke = 54% first hit → 22% second → ~3% third. Breaking the first tackle is exciting. Breaking two is rare. Three is almost impossible.

The existing `tacklesBroken >= 2` force-tackle in CONTACT_MOVE stays as a hard cap.

**Also:** `resolveContact()` currently does NOT return `breakChance` or `fumbleChance` in its return object. The ContactScreen tries to read `contactResult.breakChance` and gets undefined, displaying "Break 0%". 

Fix: add `breakChance` and `fumbleChance` to ALL return paths in resolveContact:
- DiveForward return: add `breakChance: 0, fumbleChance: 0`
- Fumble return: add `breakChance: breakChance, fumbleChance: fumbleChance`
- Tackle return: add `breakChance: breakChance, fumbleChance: fumbleChance`
- Break return: add `breakChance: breakChance, fumbleChance: fumbleChance`

### 2. DB Route Break Penalty — `QB_ACTION` reducer, DB tracking section

DBs currently use direct-vector tracking toward their assigned receiver. When a receiver makes a sharp cut (Post break, Out cut), the DB takes a diagonal intercept shortcut instead of being fooled by the cut.

In the DB tracking loop inside the QB_ACTION case (where it iterates `newDefPos` and updates CB/S positions), add a break penalty:

For each DB with an assigned receiver, before applying movement:
1. Check if the receiver's position changed sharply this phase. Compare the receiver's current waypoint direction to the previous phase. If the receiver's lateral (x) movement this phase changed direction compared to last phase (e.g., was moving mostly vertical, now moving mostly lateral), apply a penalty.
2. Specifically: calculate the receiver's movement vector this phase. If `Math.abs(dx_receiver) > 0.6 * Math.abs(dy_receiver)` AND the receiver moved at least 2 yards total this phase, the receiver is making a sharp cut.
3. When a sharp cut is detected, multiply the DB's `dbTracking` speed by `0.5` for that one phase only. This simulates the DB "flipping their hips."

To detect direction change, store receiver positions from the previous phase. Add a `prevReceiverPos` object to state (initialized to `{}` in initialState). In QB_ACTION, before updating receiver positions, save the current positions as `prevReceiverPos`. Then compare old vs new to detect the cut.

### 3. Contact Threshold — `RUNNER_MOVE` reducer

Change `nearestDef.distance < 1.5` to `nearestDef.distance < 2.0` in the contact trigger check. Player tokens are 26-30px wide, which represents nearly 3 yards of visual space. At 1.5 yards, tokens visually overlap before contact fires. At 2.0, the mechanical tackle matches what the player sees.

There is also a second reference to the contact threshold in the THROW_RESOLVE section (where it checks `nearDist` after a catch). Search for all instances of `< 1.5` that trigger contact and change them to `< 2.0`.

### 4. Remove "Run Upfield" from WR/TE — `getRunnerMoveLabels()`

The WR/TE return array has 5 items (Sprint, Run Upfield, Cut Outside, Juke, Dive Forward). This breaks the 2x2 grid layout. Remove the "Run Upfield" entry so it returns exactly 4:
```
return [
  { label: 'Sprint', type: 'Sprint', description: 'Burst forward with speed' },
  { label: 'Cut Outside', type: 'CutOutside', description: 'Break to the sideline' },
  { label: 'Juke', type: 'Juke', description: 'Quick cut to evade' },
  { label: 'Dive Forward', type: 'DiveForward', description: 'Fall forward for safe yards' }
];
```

### 5. Defensive Posture Pacing — `POSTURE_PICK` reducer

Currently clicking a posture button instantly resolves the CPU drive and transitions to CPU_RESULT. Add a delay:

1. Add a new phase: `'CPU_DRIVING'`
2. In the POSTURE_PICK case, instead of resolving immediately, transition to `CPU_DRIVING` phase. Store the chosen posture in state but don't resolve the drive yet.
3. Add a useEffect (or setTimeout in the reducer via a new action) that after 1500ms dispatches a new `'RESOLVE_CPU_DRIVE'` action.
4. The `RESOLVE_CPU_DRIVE` handler does the actual drive resolution math (currently in POSTURE_PICK) and transitions to CPU_RESULT.
5. The CPU_DRIVING phase renders: "Opponent driving..." text centered, with a subtle pulsing animation. Show the field behind it.

### 6. Camera — Ball Flight Tracking

In the camera useEffect, the ball-in-flight case currently tracks `state.ballFlightTarget.y` which snaps the camera to the receiver's position instantly when the throw button is pressed.

Change the ballInFlight camera tracking to use the midpoint between QB and target:
```
} else if (state.ballInFlight && state.ballFlightTarget) {
  const qbY = state.qbXY ? state.qbXY.y : 0;
  const targetY = state.ballFlightTarget.y;
  trackYard = state.ballPos + (qbY + targetY) / 2;
}
```

This smooths the camera transition during ball flight. The camera will then lock onto the receiver when the catch resolves and RUNNER/YAC begins.

### 7. Camera — Bottom Clamp

The current bottom clamp for non-overlay phases is `totalFieldHeight - viewportHeight`, which hides the backfield on mobile. For overlay phases it adds `viewportHeight * 0.6`.

Change the non-overlay clamp to also add extra scroll room:
```
var bottomClamp = isOverlay 
  ? (totalFieldHeight - viewportHeight + viewportHeight * 0.6) 
  : (totalFieldHeight - viewportHeight + viewportHeight * 0.15);
```

The 0.15 gives enough room to see the backfield formation without the massive overscroll of overlay mode.

### 8. Preseason Teaching Overlays — `PracticeOverlay` component

Line 6606: `if (state.difficulty !== 'practice') return null;`

Change to: `if (state.difficulty !== 'practice' && state.difficulty !== 'preseason') return null;`

The Preseason difficulty config already has `showTeaching: true`. This gate is overriding it.

### 9. Touchdown — Fireworks Upgrade

The spec calls for 5 bursts × 18 particles in 6 colors. Current code has 3 bursts with 8-10 particles (26 total) PLUS 35 confetti pieces that dilute the effect.

Replace the fireworks array with 5 bursts:
```
var bursts = [
  { left: '20%', top: '25%', particles: 18, delay: 0 },
  { left: '40%', top: '20%', particles: 18, delay: 0.3 },
  { left: '60%', top: '30%', particles: 18, delay: 0.6 },
  { left: '80%', top: '25%', particles: 18, delay: 0.9 },
  { left: '50%', top: '15%', particles: 18, delay: 1.2 }
];
```

Increase particle size range: `var size = 8 + Math.floor(Math.random() * 10);`

Delete the entire confetti generation block (the loop that creates `.confetti-piece` elements) and remove the confetti array from the render. Remove the `confetti` CSS class and `@keyframes confettiFall` from the style block.

### 10. Look Left/Right Contextual Visibility

In the DecisionScreen, Look Left and Look Right buttons currently show on every pass play. They only matter on plays where shifting the safety creates meaningful separation — deep routes.

In the DecisionScreen where `allMoveButtons` is built, conditionally exclude Look Left/Right. Use the selected play to determine if looks are relevant:

```
// Only show Look buttons on plays where safety manipulation matters
const play = PLAYS.find(p => p.id === state.selectedPlayId);
const deepPlays = ['deepPost', 'fourVerticals', 'teSeam', 'playAction', 'flea'];
const showLooks = deepPlays.includes(play && play.id);
```

Filter Look Left and Look Right out of `allMoveButtons` when `showLooks` is false. This reduces button clutter on Quick Slants, Screen Pass, Out Routes, and Curl/Comeback where the safety position doesn't affect the play design.

---

## AFTER BUILD

Update NOTES.md with what changed.

## DONE NOTE

When complete, report in plain English:
- What changed (one line per fix)
- Where files are saved
- Any issues encountered

**Playtest checklist:**
1. Run a pass play (Quick Slants). Verify Look Left/Right do NOT appear.
2. Run a Deep Post. Verify Look Left/Right DO appear. At phase 3-4, check if W1 gets open faster than before (DB break penalty working).
3. Complete a pass to W1. Verify camera doesn't snap violently — should smoothly track during ball flight.
4. In YAC/Runner mode after a catch, verify only 4 buttons show for WR (Sprint, Cut Outside, Juke, Dive Forward). No "Run Upfield."
5. Get into CONTACT. Verify Break % shows a real number (not 0%). First contact should show ~50% for Juke. If broken, second contact should show ~20%.
6. Run a play and check that contact triggers when the defender visually arrives (not after tokens overlap).
7. Pick a Defensive Posture. Verify there's a ~1.5 second "Opponent driving..." pause before the result appears.
8. Score a touchdown. Verify fireworks (5 bursts, big particles, no confetti).
9. Start a new game on Preseason. Verify teaching overlays appear on first MENU and first DECISION screen.
10. On mobile (or narrow browser window): check that the backfield is visible during PLAY_SELECT — QB and RB shouldn't be crammed against the top edge.
