# Hail Marys Build Notes

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
