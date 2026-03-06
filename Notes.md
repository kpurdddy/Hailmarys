# Hail Marys Build Notes

## G2.4.0 — Defensive Overhaul + Kick Flight + Melificent (2026-03-05)

Applied to 7,549-line G2.3.1 version. 827 insertions, 314 deletions.

### Fixes Applied
1. **CONTACT Phase LOS Gating** — Non-DL defenders can't cross LOS until carrier does. Convergence speed reduced from 1.0 to 0.15. Auto-tackle timer (25 ticks / 5s) forces "Forward progress stopped" on idle.
2. **advanceDefenders() LOS Gating** — Same LOS gate in the RUNNER_MOVE path. DL can penetrate (TFLs work), LBs/secondary hold until carrier crosses y > 0.
3. **YAC Limit Text Fix** — "Brought down after the catch" only when defender within 4 yards. Otherwise "Ran out of steam."
4. **Safety Depth Overhaul** — Zone safeties maintain 5+ yards behind deepest receiver (min 12 from LOS). Split field laterally (x=30/70) instead of clustering toward QB.
5. **CB Tracking Boost** — CB tracking multiplier raised from 5 to 7. Extra 0.8 yard push when CB falls behind receiver. Creates real coverage differentiation.
6. **LB Zone Coverage** — LBs track nearest receiver within 15 yards instead of blindly drifting toward QB. Makes short/middle routes harder.
7. **Kick Flight Animation** — New KICK_FLIGHT phase with 1.2s delay for punts and field goals. Ball flight visual before result resolves.
8. **Melificent Easter Egg** — FG attempts over 60 yards trigger dramatic failure messages. First attempt per possession: pity (down given back). Second+: real turnover. Cheat code: toggle Go For It 3x then FG = guaranteed success. 0.01% natural miracle chance.
9. **Possession/Quarter Advancement** — KICK_RESOLVE properly handles possession counting, quarter advancement, and two-player mode for both punts and FGs.

### State Fields Added
- `melificentAttempts` — tracks Melificent attempts per possession (resets on turnover/score)
- `goForItToggleCount` — cheat code counter (resets on FG attempt or possession change)

---

## G2.3.0 — Fix Pass (2026-03-01)

Applied to the 7,131-line full version. 325 insertions, 130 deletions.

### Fixes Applied
1. **Dive ALWAYS Ends the Play** — Moved DiveForward check BEFORE defender proximity gate in RUNNER_MOVE (line ~3330). Dive now always returns RESULT regardless of defender distance.
2. **YAC Limit Enforced** — Added maxYacMoves check at top of RUNNER_MOVE. Auto-resolves play when YAC moves exceed limit. Added visual YAC pips in RunnerScreen.
3. **RUNNER Defenders Converge on Tick** — Replaced no-op ANIMATION_TICK for RUNNER with convergence logic. Defenders creep toward carrier every tick, speed scales by difficulty (practice=0.15, playoffs=0.50). Auto-tackle at range 1.2.
4. **Sprint Label Swaps** — Three states: 0 sprints = "⚡⚡ Sprint", 1 sprint = "⚡ Sprint", 2+ = "🏃 Run".
5. **Dan & Kiki Commentary ON the Field** — Removed 2750ms auto-dismiss timer. Moved dismiss button below text (was top-right ×). Rendered CommentaryOverlay INSIDE field container with absolute positioning. Updated commentary-panel CSS.
6. **Decision Screen Layout Flip** — Early phases (currentPhase <= 1): QB actions on top, receiver cards muted/compressed below. Later phases: receivers expand to top, QB actions below. Escape row always at bottom.
7. **Preseason Declutter** — Look L/R requires 2+ deep passes on Preseason, 3+ on Practice (deepPassCount tracking). Escape row simplified on easy modes: Tuck (safe) + Scramble (risky!) only.
8. **Tuck vs Scramble Fumble Risk** — scrambledOut flag set on scramble. 40-50% fumble chance on first contact after scrambling.
9. **CPU Drive Play-by-Play Ticker** — Added plays arrays to resolveCPUDrive (3-5 lines per drive). CPU_DRIVE_NEXT_PLAY action steps through at 800ms. Drive tells a story before resolving to CPU_RESULT.
10. **Receivers Get Open Faster (Preseason)** — dbTracking lowered from 0.82 to 0.72. calculateReceiverOpenness now accepts difficulty param. Openness thresholds lowered for practice/preseason (Wide Open at 4yd vs 5yd, Open at 2.5 vs 3, Contested at 1.0 vs 1.5).

### State Fields Added
- `deepPassCount` — tracks deep pass plays for progressive Look introduction
- `scrambledOut` — flags QB scramble for fumble risk on contact
- `cpuDrivePlays` / `cpuDrivePlayIndex` — play-by-play ticker arrays
