TACFOOT REWRITE HANDOFF — February 20, 2026 (v3 — FINAL)

Complete game spec. No code. No old variable names.
Starting point for the design doc thread.

All numeric values marked [TUNE] are reasonable defaults
subject to adjustment during playtesting.

================================================================
WHAT TACFOOT IS
================================================================

A single-file React app that teaches football strategy through
turn-based gameplay. Target audience: newcomers learning football,
while remaining engaging for experienced fans.

Current version: ALPHA 15.5 (working game, broken rendering)

================================================================
WHY A FULL REWRITE
================================================================

The rendering/state management layer generates bugs faster than
they can be patched. Three rounds of "fix the teleport bug"
failed — including one that was called a "complete rewrite" but
was actually a refactor that carried over the broken architecture.

The game logic works. The problem is exclusively how game state
becomes moving visuals on screen.

Decision: start from an empty file. Write a complete design doc
in plain English first. Build from the doc, not from old code.

================================================================
ARCHITECTURE DECISIONS
================================================================

State management:
  - useReducer with single state object
  - One dispatch per game tick — no rapid-fire chains
  - Game logic as pure functions: state in, state out

Rendering:
  - Single render pass: game coordinates → pixel positions
  - No dual position tracking systems
  - No visibility culling (25 dots don't need it)

Camera:
  - CSS transform on field container
  - Stored as a ref, never in React state
  - Never triggers re-renders

Transitions:
  - CSS class-based transitions
  - No inline transition manipulation
  - No noTransition flag toggling

================================================================
GAME STRUCTURE
================================================================

Possession-based, not play-count.

  - 4 quarters
  - 2 offensive possessions per quarter per player
    (single-player: 2 per quarter, 8 total;
     two-player: 2 each per quarter, 16 total drives)
  - Each drive lasts as many plays as it takes (score, punt,
    turnover, or turnover on downs)
  - After every possession change in single-player, the CPU
    gets the ball and the player picks a Defensive Posture
  - The CPU drive resolves, then the player's next possession
    starts

Quarter transitions:
  - Quarter advances when the possession count for that quarter
    is used up
  - Brief "END OF Q1" / "END OF Q2" / "END OF Q3" flash
    when quarters change
  - Audible counter resets each quarter

Halftime:
  - "HALFTIME" screen with score summary between Q2 and Q3
  - In two-player, marks the possession flip (see Two-Player)

Mid-drive at end of Q4:
  - If the player is mid-possession when Q4's possessions are
    used up, the current drive finishes before the game ends

Game end:
  - Final score screen: YOU vs CPU
  - Win / Loss / Tie result
  - Tie declared for v1 (overtime in a future update)

Clock:
  - Visual clock ticks down in all modes — it looks like
    football, but it does not constrain play
  - The clock is decorative in v1; quarters change based on
    possessions, not clock
  - FUTURE: in Regular/Playoffs, clock becomes mechanical —
    run plays cost more time, CPU drives eat clock, creating
    real time-management strategy

Safety rule:
  - Ball position clamped at the 1-yard line (minimum)
  - Safeties not implemented in v1
  - FUTURE: if ball carrier tackled behind own goal line,
    safety = 2 points to CPU + safety kick Defensive Posture

================================================================
GAME STATES (complete flow)
================================================================

1. TITLE / DIFFICULTY SELECT
   - Game title "TACTICAL FOOTBALL" with version
   - Intro text explaining what the game is
   - Feedback link (Google Form)
   - Step 1: Pick your coach (two choices, see Coaches below)
   - Step 2: Pick difficulty (4 levels, see below)
   - Also: 2-Player button (Regular Season difficulty)

2. MENU (play category select)
   - Three big buttons: RUN / PASS / TRICK
   - Color-coded: Run=orange, Pass=blue, Trick=purple
   - On 4th down: additional KICK OPTIONS button
   - Scoreboard visible at top (see Scoreboard below)

3. PLAY SELECT
   - Shows plays for chosen category
   - Grouped: Your Playbook / Situational / Specialty
   - Each play shows: icon, name, brief description
   - Tapping a play expands it: full description bullets,
     strong/weak matchup tags with explanations
   - Double-tap quick-snaps (selects + goes to presnap)
   - Back button to return to category select
   - Ghost route lines appear on the field when a play
     is selected (translucent colored lines showing
     planned routes with arrowheads)
     Blue for WR/TE routes, orange for RB routes
     Visibility by difficulty:
       Practice/Preseason: shown at play select, presnap,
         and during play
       Regular Season: shown at play select and presnap only
       Playoffs: shown at play select only
     In 2-Player mode, the defensive player has already
     locked in their defense choice before the offensive
     player sees presnap, so ghost routes do not leak info.

   Ghost routes on pass plays:
     Receiver ghost routes show the actual automated paths
     receivers will follow. What you see IS what they'll run.

   Ghost routes on run plays:
     Run ghost routes show the SUGGESTED path — the play's
     intended design. The player executes the actual run
     through Runner Mode choices. The ghost route is a plan,
     not a script.
     - RB route shown as solid orange line (dive path,
       sweep path, counter cutback, toss arc, etc.)
     - Counter play: dashed orange line for the fake
       direction, solid orange for the actual cutback
     - Decoy receiver go routes shown as faded/dim lines
       (30% opacity vs 50% for real routes)
     - Blocking assignments show no ghost line
     - Practice mode tutorial explains what the lines mean
       on first appearance

4. PRESNAP
   - Defense has been chosen (random, weighted by situation)
   - Shows: defense name, matchup rating for selected play
   - Matchup feedback varies by difficulty:
     Practice: no matchup rating tags shown (coach still
       gives a one-liner advice below the field)
     Preseason: "Great call!" / "Good call" / "Decent look" /
       "Careful — they're in X" / "This is a BAD call against X"
       with clickable suggestions for better plays
     Regular: only warns on bad (-1) and terrible (-2) matchups
     Playoffs: no feedback
   - Coach advice line (one-liner based on matchup, all modes)
   - Practice mode: tutorial overlay explaining the defense
   - Two buttons: SNAP and AUDIBLE
   - Ghost route lines still visible on field (see visibility
     rules above)
   - SNAP starts the play
   - AUDIBLE returns to play select
     Audible limits:
       Single-player Practice: unlimited
       Single-player Preseason: unlimited
       Single-player Regular: 2 per quarter
       Single-player Playoffs: 1 per quarter
       Two-player: 1 per quarter (offense only)
     Button grayed out if limit reached this quarter.

5. SNAPPING / ANIMATING
   - Brief transition state
   - "Ball snapped..." or "Play developing..." text
   - Ball flight animation: center snaps to QB (parabolic arc)
   - No player input during this state

   After snap, flow branches:
     Pass plays → State 6 (Decision)
     Run plays with RPO → State 7 (RPO Read)
     Run plays without RPO (HB Toss, QB Sneak) → State 8
       (Runner Mode) directly, skipping RPO Read
     Trick plays → varies by play (Flea Flicker has pitch
       fumble check first, then State 6; Statue of Liberty
       and QB Draw go to State 8)

6. DECISION (pocket phase — passing plays)
   Single continuous state with escalating urgency as
   pocket degrades. NOT multiple separate states.

   - QB DECISION header
   - Pocket integrity display: Left/Right bars, color-coded
     Green (>60%), Yellow (30-60%), Red (<30%)
   - Phase counter
   - Throw targets in a 2x2 grid, each showing:
     Receiver label (W1, W2, TE, RB)
     Openness tag (Wide Open / Open / Contested / Covered)
       color-coded green → red
     HIT % and INT % in monospace
     Primary/intended receiver marked with gold ring + "INTENDED"
     Best targets glow green in Practice/Preseason
   - Action buttons below throws:
     Drop Back: y moves -2 (further behind LOS)
     Roll Left: x moves -8 (toward left sideline)
     Roll Right: x moves +8 (toward right sideline)
     Step Up: y moves +2 (toward LOS — reduces sack loss
       if hit, but moves closer to interior DL)
     Look Left, Look Right (shifts safety, no position change)
     Scramble (abandon pass, QB enters runner mode, speed 0.75)
     Throw It Away (carries grounding risk — see Intentional
       Grounding section)
     Tuck the Ball (accept sack, protect ball, zero fumble,
       fixed 1-2 yard loss [TUNE] regardless of QB position —
       the "it's over, don't make it worse" option)
   All movement amounts [TUNE].
   - Practice/Preseason: highlights recommended actions,
     shows explanatory tips about why targets are open

   Scramble is always available as a button. A separate
   SCRAMBLE LANE INDICATOR (directional arrow on the field
   showing which direction is open) appears when pocket
   integrity on one side is below 40% while the other side
   is above 60% — asymmetric collapse with a clear escape
   direction. The indicator is a visual hint, not a gate.
   You can scramble without the indicator; you just won't
   know which direction is safer. In Practice, the coach
   says "The left side is collapsing — scramble right!"

   All QB actions advance the game by 1 phase. This means:
     - All receiver routes progress one waypoint
     - All OL hold times tick down by 1
     - All defender positions update (DB tracking, DL closing)
   There is no "free" action. Every choice costs time.

   QB position is tracked continuously from formation spot:
     I-form: starts at (50, -3)
     Shotgun: starts at (50, -5)
   Each action updates the QB's x/y. Grounding checks,
   sack yardage, and scramble direction all read from the
   QB's current tracked position.

   POCKET ESCALATION (continuous, not separate states):
   Overall pocket % = average of all matchup integrities.
   See Passing Mechanics → Pocket Integrity for the engine
   that drives these values. Each play feels different
   depending on the hold time rolls at snap.

   60%+ pocket (green):
     Full options. All throw targets visible. All action
     buttons available. No warnings. Normal play.

   30-60% pocket (yellow):
     "PRESSURE BUILDING" warning on field.
     Same options available but visual urgency increases.
     Yellow pocket bars. Coach chimes in.
     Completion percentages degrading from pressure.

   15-30% pocket (red):
     "POCKET COLLAPSING" warning. Direction shown
     (LEFT / RIGHT / BOTH SIDES).
     Throw targets still visible but completion percentages
     are bad and getting worse. Scramble and Tuck highlighted
     as recommended. Red bars.

   Below 15% with defender within 2 yards (survival):
     "THROW OR TUCK!" — pocket is gone.
     Throw target buttons disappear entirely.
     Only three options remain:
       - Tuck the Ball (accept sack, protect ball)
       - Scramble (try to escape, enter runner mode)
       - Throw It Away (grounding risk if inside tackle box)
     Visual: receiver buttons fade/collapse, leaving only
     escape options. Practice overlay explains why.

7. RPO READ (run-pass option — some run plays only)
   - Only appears for run plays that have RPO:
     Inside Run, Outside Run, Counter, Power Run, Stretch Run
   - Does NOT appear for: HB Toss (pitch already in air),
     QB Sneak (pure push play). These skip to Runner Mode.
   - POST-SNAP READ header
   - Scenario indicator: "THERE'S A HOLE!" (green) or
     "DEFENDER CLOSED THE LANE" (red) or
     "NO ROOM OUTSIDE" (red)
   - Three buttons: Hand Off / QB Keep / Quick Pass
   - Each shows success %, description, color coding
   - Practice labels are friendlier than Regular/Playoffs

   RPO Quick Pass target:
     TE breaks off blocking assignment into a quick flat route
     (5 yards toward sideline). Presnap ghost routes show the
     run version (TE blocking). The quick pass is the post-snap
     read — TE leaks out if the player picks Quick Pass.

   RPO Quick Pass resolution:
     Single roll, NOT the full Decision screen. This is a
     designed quick throw into space, not a full passing play.
     Base completion: 75% [TUNE]
     INT chance: 2% [TUNE]
     Modified by difficulty (same throw modifiers as other
     passes). Resolves immediately to complete/incomplete/INT.
     If complete, TE becomes ball carrier at the flat route
     spot (~5 yards from LOS toward sideline) and enters
     Runner Mode for potential YAC.

   RPO gap-open probability:
     Favorable matchup (+1 or +2): 70% gap open
     Neutral matchup (0): 50% gap open
     Unfavorable matchup (-1 or -2): 15% gap open

8. RUNNER MODE (ball carrier has the ball)
   Ball carrier enters Runner Mode at their current field
   position at the moment of transition:
     RB handoff: RB at the LOS (y=0) — handoff at the line
     QB scramble: QB at whatever position tracked in Decision
     QB Draw / QB Sneak: QB at formation spot
     RPO quick pass: TE at flat route spot (~5 yds from LOS)
     Catch + YAC: receiver at the catch point

   - "[CARRIER] — MAKE A MOVE" header
   - YAC counter (catches: "YAC 0/3") or move counter
   - Nearest defender warning: name, distance, color-coded
     Red (<3 yds) / Yellow (3-6 yds) / Green (>6 yds)
   - Action buttons in 2x2 grid, each with:
     Label, success %, description
     Color-coded by type
   - Run arrows on the field showing available directions

   Runner move labels vary by position (same underlying
   mechanics, different labels that feel right):

     RB:
       Sprint = Sprint (advance 3-5 yards [TUNE])
       Hit the Hole = Hit the Hole (advance 2-4 yards [TUNE])
       Juke = Juke (lateral 1-2 yards + forward 1-2 [TUNE])
       Dive Forward = Dive Forward (1-3 yards, zero fumble)

     WR/TE after catch:
       Sprint = Sprint (3-5 yards)
       Cut Outside = Cut Outside (2-3 lateral + 1-2 forward)
       Juke = Juke (1-2 lateral + 1-2 forward)
       Dive Forward = Dive Forward (1-3 yards, zero fumble)

     QB scramble:
       Sprint Upfield = Sprint (3-5 yards, QB speed 0.75)
       Cut Outside = Cut Outside (2-3 lateral + 1-2 forward)
       Slide = Dive Forward (1-3 yards, zero fumble)
       Truck It = Stiff Arm (uses strength formula)

   Movement distances listed above are yards gained on
   success. On failure (defender in the way), the move
   still advances the carrier 0-1 yards but triggers
   contact. All distances are [TUNE].

   Runner action success rates are contextual:
     Sprint / Hit the Hole (straight ahead):
       No defender within 5 yards ahead: 85%+
       Defender 3-5 yards ahead: 60-70%
       Defender within 3 yards ahead: 30-40%

     Cut Outside / Bounce Outside (lateral):
       Sideline route clear (no defender within 4 yards): 75%+
       Defender between carrier and sideline: 25-35%

     Juke / Spin (evasion):
       1 defender nearby: higher success (1-on-1)
       2+ defenders nearby: much lower success
       See Contact System for exact percentages

     Dive Forward / Slide (safe):
       Always 80-85% for 1-3 yards
       Always zero fumble risk
       The "I'll take what I've got" option

   Each runner action = 1 phase for timing purposes. This
   means OL hold times tick down during runs too (see
   Run Blocking below for how this affects gap closure).

   Defender pursuit speed (base, before escalation):
     Practice:   70% of ball carrier speed
     Preseason:  82%
     Regular:    90%
     Playoffs:   100%
   Same scale as DB tracking speed — one difficulty knob
   controls both pass coverage closing and run pursuit.

   Pursuit escalation:
     +15% defender speed per runner move after move 2.
     Tackle probability ramp: 80%+ by move 5.
     Hit the Hole success rate decay: -12-15% per use.
     Breakaway runs still possible, just rare.

   Run direction uses forward cone gap-finding to generate
   the visual run arrows (the hint system showing which
   directions have open lanes). Not automated pathfinding —
   the player still picks their move:
     ±45 degrees from straight ahead, 4-unit scan resolution,
     lateral penalty (straight ahead scores 60% higher),
     target clamped within 15 units of ball carrier.
     Massive score penalty for paths toward defenders within
     3 yards.

   Out of bounds: x ≤ 0 or x ≥ 100 = out of bounds. Play
   ends at the current y-position (yard line where carrier
   crossed the sideline).

   No hard cap on runner moves for runs (unlike MAX_YAC = 3
   for catches). Pursuit escalation is the natural limiter.
   Breakaway runs are rare but possible — a runner who breaks
   4 tackles is a highlight play, not a bug.

9. CONTACT (defender reaches carrier)
    - Red pulsing border: "CONTACT!"
    - Shows which defender, distance, defenders nearby
    - Tackles broken this play counter (risk escalates)
    - Options grid:
      Spin Move: break %, fumble %, tackle %
      Juke: break %, fumble %, tackle %
      Stiff Arm: break %, fumble %, tackle %
      Dive Forward: guaranteed +1-3 yards, NO fumble risk
    - Contact flash overlay on field shows move + break %

10. RESULT
    - Compact bar showing: yardage, description
    - Color: red=turnover/loss, yellow=incomplete,
      green=big gain, white=normal
    - NEXT button
    - Play-by-play log in monospace at bottom
    - Practice/Preseason: coaching debrief explaining what
      happened
    - Turnover on downs: result text acknowledges the gamble.
      "Stopped on 4th down! Opponent takes over at their 2."
      feels different from "Turnover on downs. Opponent takes
      over at your 30."

11. TOUCHDOWN
    - Full overlay: football emoji bouncing, "TOUCHDOWN!" glowing
    - Fireworks particles (5 bursts, 18 particles each, 6 colors)
    - "+7 POINTS" display
    - Play description and play-by-play chain
    - KICKOFF button (triggers Defensive Posture — CPU gets
      ball at their 25)
    - Crowd sound effect

12. FOURTH DOWN OPTIONS
    - Reached via KICK OPTIONS button in Menu (state 2)
    - Three buttons: Punt / Field Goal (shows distance) / Go For It
    - Punt and Field Goal resolve immediately (see Field Position)
    - Go For It returns to Menu (state 2) for normal play selection

13. DEFENSIVE POSTURE (single-player only)
    - Triggered after every possession change: punt, turnover on
      downs, interception, fumble, missed field goal, kickoff
      after a score
    - Shows: "OPPONENT'S BALL" with CPU starting field position
      (e.g. "Opponent ball, their 35-yard line")
    - Coach advises on posture choice:
      "They're starting deep — Bend Don't Break is fine here."
      "They're on your 35 — might want to go Aggressive."
    - Three big buttons (fast, snappy — no scrolling):

      BEND DON'T BREAK:
        Prevent big plays. CPU likely limited to field goal
        if in range. Lowest TD chance. Lowest turnover chance.
        Safe, predictable. Ceiling is capped.

      AGGRESSIVE:
        Blitz-heavy. Highest chance of forcing CPU turnover
        OR giving up a TD. Lowest FG chance (binary outcome).
        High variance — gamble.

      BALL HAWK:
        Balanced middle option. Moderate TD chance, moderate
        FG chance, moderate turnover chance. For v1, this is
        the straightforward middle ground.
        FUTURE: specifically counters pass-heavy CPU drives
        when CPU drive types are added.

    - Single roll determines CPU drive outcome, weighted by
      CPU starting field position + posture choice
    - Possible results:
        3-and-out (no score, player gets ball at own 25)
        Field goal (CPU +3, player gets ball at own 25)
        Touchdown (CPU +7, player gets ball at own 25)
        CPU turnover (no score, player gets ball at the
          turnover spot — see below for how spot is calculated)

    CPU turnover spot:
      CPU drives are a single roll, so the "drive distance"
      is generated for the turnover spot and flavor text.
      CPU advances random(5, 25) yards from their starting
      position before the turnover. Capped so the CPU can't
      "drive" past the player's 10-yard line equivalent.
      Example: CPU started at their 30, drove to their 48.
      Player gets ball at CPU's 48 (player's 52 in player
      coordinates).
      This feeds the result text: "Opponent drove to midfield
      before your defense forced a fumble."

    - Result screen: brief drive summary (2-3 lines)
      "Opponent drove to your 14 and kicked a field goal.
       CPU 3 — YOU 14."
      Dan and Kiki comment on the CPU drive.

    - Coach debrief evaluates the posture choice:
      Good call, good result:
        "Bend Don't Break kept them to a field goal.
         When they start deep, you don't need to gamble."
      Good call, bad luck:
        "Aggressive was the right read from that field
         position, but they broke a big play. That happens."
      Bad call, got away with it:
        "Ball Hawk against a team on your 30? You got lucky.
         Aggressive would have been smarter there."
      Bad call, paid for it:
        "Bend Don't Break in the red zone gives them a free
         field goal. That was the time to go Aggressive."

    - Screen should be snappy: pick posture, see result,
      auto-advance or tap through. Minimal delay.

    CPU scoring odds by starting field position (base rates
    before posture modifiers, all approximate [TUNE]):

      CPU starts at their own 0-20:   ~5% FG,  ~1% TD
      CPU starts at their 21-40:     ~15% FG,  ~3% TD
      CPU starts at their 41-50:     ~25% FG,  ~8% TD
      CPU starts at your 40-21:      ~40% FG, ~15% TD
      CPU starts at your 20 or less: ~50% FG, ~30% TD

    Posture modifiers (all multiplicative):
      Bend Don't Break: TD x0.5, FG x1.2, turnover 5%
      Ball Hawk: TD x1.0, FG x1.0, turnover 10%
      Aggressive: TD x1.5, FG x0.5, turnover 20%

    Difficulty scaling on CPU scoring:
      Practice:   base odds x0.6
      Preseason:  base odds x0.8
      Regular:    base odds x1.0
      Playoffs:   base odds x1.2

    Practice mode tutorial overlay on first appearance:
      "Your opponent has the ball now. Pick how your defense
       plays. Bend Don't Break is safe — they probably won't
       score big. Aggressive is risky but might stop them
       cold. Ball Hawk is in between."

14. TWO-PLAYER DEFENSE SELECT
    - Rotated 180° (pass the device)
    - Defense picker for player 2

15. HALFTIME
    - Brief screen: "HALFTIME" with score summary
    - In two-player: marks the possession flip

16. QUARTER TRANSITION
    - Brief "END OF Q1" / "END OF Q2" etc. flash
    - Updates scoreboard

17. GAME OVER
    - Final score, win/loss/tie
    - Tie for v1 (overtime in future update)

================================================================
FIELD POSITION RULES
================================================================

After player scores a touchdown:
  - Kickoff: CPU gets ball at their 25
  - Triggers Defensive Posture (single-player)
  - After CPU drive resolves, player starts at own 25

After punt:
  - Ball travels 35 + rand(0,15) yards from current position
  - CPU starts at the landing spot
  - If punt reaches or exceeds 100: touchback, CPU starts
    at their 20
  - Triggers Defensive Posture
  - After CPU drive resolves, player starts at own 25
    (unless CPU turnover gives better position)

After turnover on downs:
  - CPU gets ball at the spot of the failed 4th down
  - Triggers Defensive Posture

After interception:
  - CPU starts at approximately the targeted receiver's
    field position at the time of the throw
  - If that position would be inside the player's end zone
    (past yard 100), treat as touchback — CPU starts at
    their 20
  - Triggers Defensive Posture

After fumble:
  - CPU starts at the spot of the fumble
  - Trick play fumbles (Flea Flicker pitch-back, etc.)
    resolve at the fumble spot immediately — no passing
    phase follows a fumbled trick play setup
  - Triggers Defensive Posture

After missed field goal:
  - CPU starts at the kick spot or their own 20, whichever
    gives the CPU better field position.
    In player coordinates: min(ball_position, 80).
  - Triggers Defensive Posture

After CPU drive resolves:
  - Player gets ball at own 25, UNLESS the CPU drive
    result was "CPU turnover," in which case the player
    starts at the turnover spot (see Defensive Posture
    section for how spot is calculated)

CPU field position coordinate note:
  Ball position is tracked as "yards from player's own goal,
  0-100." CPU field position must be inverted for the scoring
  odds calculation. CPU at "their 20" = player's 80 in
  player-perspective coordinates. The scoring odds table
  uses CPU-perspective coordinates (distance from player's
  end zone). The build must explicitly handle this conversion.

================================================================
FIELD VISUAL ELEMENTS
================================================================

The field:
  - Dark green base (#1e6830)
  - Alternating turf stripes in 5-yard bands (#2d5a1e / #326320)
  - 100 yards visible, scrollable via camera

Yard lines:
  - 10-yard lines: white, 2px, 35% opacity
  - 5-yard lines: hash marks at left/right edges and center
  - Minor yard lines: subtle, 8% opacity
  - Yard numbers at 10-yard marks (20, 30, 40, 50, etc.)
    on left and right sides, large monospace, 18% opacity

Hash marks:
  - Small horizontal ticks at ~29.5% and ~70.5% of field width
  - Every non-5-yard position

End zones:
  - Opponent: dark red (#6a1a1a) with diagonal stripe pattern
    and "END ZONE" text
  - Own: dark blue (#1a2a5a) with same stripe pattern
  - Both have semi-transparent text overlay

Goal lines:
  - Opponent goal: bright white, 4px, with glow
  - Own goal: slightly dimmer white, 3px

Goal posts:
  - Gold/yellow (#ddb830), two uprights + crossbar
  - At each end of the field

Midfield logo:
  - "TF" in very faint white text at the 50-yard line

Line of scrimmage:
  - Cyan/blue line across the field at current position

First down marker:
  - Yellow line across the field (with glow)
  - Only shown when distance to gain ≤ 30 yards

Ghost route lines (presnap/playcall):
  - Translucent colored lines: blue for receivers, orange for RB
  - Arrow segments connecting route waypoints
  - Arrowhead at the end of each route
  - 50% opacity for real routes, 30% for decoy routes
  - Pass routes = automated paths; run routes = suggested plan
    (see Game States section for full distinction)

Run arrows (runner mode):
  - Colored directional arrows from ball carrier
  - Each arrow points to a potential running lane
  - Color indicates quality/safety of the lane

Players:
  - Offense: Blue fills (#1e56a0), blue borders
  - Defense: Red fills (#b82020), red borders
  - OL: darker blue (#2a5a90), color changes with integrity
    Green (#2a7a40) = holding, Yellow (#8a6a20) = struggling,
    Red (#8a2020) = beaten
  - DL: darker red (#7a1a1a)
  - Ball carrier: gold border (#fbbf24), 3px
  - Player shapes by position:
    QB: diamond (rotated square, 30x30)
    RB: rounded rectangle (28x22, large radius)
    WR: circle (26x26)
    TE: rounded square (28x28)
    OL: small square (22x22)
    DL: triangle (clip-path, pointing up)
    LB: hexagon (clip-path)
    CB/S: circle (26x26)
  - Labels inside: 2-letter abbreviation (QB, RB, W1, W2, etc.)
  - All have drop shadows

Receiver openness labels (during decision phase):
  - Small floating label above each receiver
  - Shows: "Wide Open" / "Open" / "Contested" / "Covered"
  - Color: green → red gradient by openness level
  - Dark semi-transparent background
  - Primary/intended receiver gets gold ring + "INTENDED" label

OL matchup dots:
  - Tiny dots in top-left of field during pass plays
  - One per OL-DL matchup, green/yellow/red
  - Also shown below each OL player on the field

Ball:
  - Brown circle (#8B4513) with white laces
  - When held: positioned at carrier
  - When in flight: CSS @keyframes parabolic arc animation
    Start point → mid-air peak → end point
    Scale up at peak to simulate depth
  - "IN THE AIR!" overlay label during flight

Field alerts (floating over field center):
  - "POCKET COLLAPSING" (yellow)
  - "W1 WIDE OPEN" (green)
  - Fade-in, hold, fade-out animation (2 seconds)
  - Large bold text with colored glow/text-shadow

Pocket collapse warning (above QB position):
  - "POCKET COLLAPSING" or "THROW OR TUCK!"
  - Shows pressure direction (LEFT / RIGHT / BOTH SIDES)
  - Practice: adds explanatory sub-text

Contact flash (at contact point):
  - Dark overlay showing move name, break %, fumble odds
  - Appears briefly when contact is resolved

================================================================
SCOREBOARD (top bar, always visible during play)
================================================================

Left side:
  - YOU (or P1) score in blue, large font
  - "vs"
  - CPU (or P2) score in red, large font
  - In 2-player: football emoji next to active offense

Center:
  - Game version text

Right side:
  - Down & distance (e.g. "1st & 10")
  - Field position (e.g. "OWN 25 · Q1")
  - Possession counter (e.g. "Poss 3/8")
  - Mute/unmute button
  - Menu button (hamburger)

Down & distance HUD (on field, bottom):
  - Dark semi-transparent overlay
  - Large monospace: "1st & 10"
  - Field position text

Runner info HUD (on field, bottom):
  - Shows during run/YAC
  - "YAC 0/3" or "Run #1"
  - "Nearest: CB 3.2yds"

Defense label (on field, top center):
  - Red semi-transparent badge
  - Shows defense name
  - Shows look direction arrow if QB has looked L/R

================================================================
COMMENTARY SYSTEM (Announcer's Booth)
================================================================

Appears after play resolves (2.75 second delay).
Two announcers:

DAN (play-by-play):
  - Blue accent color (#4a9eff)
  - Factual, describes what happened
  - Uses {target}, {yards}, {play}, {def} placeholders

KIKI (color commentary):
  - Orange/gold accent (#f5a623)
  - Italic text
  - Analysis and teaching
  - Two modes: regular Kiki (experienced fans) and
    KikiTeach (Practice/Preseason — explains WHY)

Categories of commentary (8+ lines each):
  - Big pass play (10+ yards)
  - Short pass (under 10 yards)
  - Drop (receiver was open but dropped it)
  - Contested incomplete
  - Covered incomplete
  - Interception
  - Sack
  - Big run (10+ yards)
  - Short run (1-9 yards)
  - Run stuffed (0 or negative)
  - Touchdown
  - Throw away
  - Fumble
  - CPU drive result (Dan/Kiki also comment on Defensive
    Posture outcomes)

Layout: broadcast-style overlay at top of field.
"ANNOUNCER'S BOOTH" header with gold underline.
Dan's line first, then Kiki's. Dismiss button.

================================================================
COACH SYSTEM
================================================================

Two coaches:

"Baby Boy Joey" — The Gunslinger (Type A):
  "Young, cocksure, and always scheming. Calls plays that make
  defensive coordinators swear. When they work, they're
  spectacular — and when they don't, his grit and hustle
  already have the next one drawn up."
  - Blue themed (#4a9eff)
  - SVG avatar: young face, cap, blue jacket
  - Aggressive advice: "Let it fly!" / "Make him miss!"
  - Pushes for big plays

"Grizzled Jim" — The Old Fox (Type B):
  "Forty years on the sideline. Players run through walls for
  him. Knows the game cold, never panics, and a steadying
  presence when things get crazy."
  - Green themed (#22c55e)
  - SVG avatar: older face, glasses, mustache, brown coat,
    headset
  - Conservative advice: "Secure the ball" / "Take what
    they give you"
  - Favors safe plays

Coach advice appears as a one-liner below the field:
  - Left border color matches coach color
  - During presnap: matchup assessment
  - During pocket: read suggestions
  - During run: directional advice
  - During Defensive Posture: advises which posture to pick
  - After Defensive Posture: debriefs whether the choice was
    right for the situation (see Defensive Posture section)
  - Adapts to difficulty (more explicit at lower levels)

================================================================
PRACTICE OVERLAYS (teaching system)
================================================================

Semi-transparent info cards that appear at key moments
in Practice mode. Each shows once per game state.

Triggers:
  - First time in menu: explains playbook categories
  - Second time in menu: "Pick your next play" reminder
  - Presnap: explains current defense formation
    (separate explanation for each of the 5 defenses)
  - First pocket phase: explains decision screen
  - First Defensive Posture: explains the three posture
    choices and what they do
  - First ghost route appearance on a run play: explains
    that run ghost routes are suggestions, not automated paths
  - First extreme pressure (options disappearing): explains
    why throw targets vanished
  - And more per game phase

Format: dark card, blue border, coach icon, text,
"GOT IT" dismiss button.

================================================================
PLAYS — COMPLETE SPECIFICATIONS
================================================================

Each play has: name, icon, type (run/pass/trick),
category (staple/situational/special), tooltip,
brief description, detailed description bullets,
strong matchup + why, weak matchup + why,
formation preset, and route definitions for each receiver.

FORMATIONS:
  - FI (standard I-formation): QB at (50,-3), RB at (50,-7),
    WR1 at (8,0), WR2 at (92,0), TE at (66,0)
    OL at (38,0), (43,0), (50,0), (57,0), (62,0)
  - FSG (shotgun general): QB at (50,-5), RB at (42,-5),
    WR1 at (6,0), WR2 at (94,0), TE at (68,0)
    OL at (38,0), (43,0), (50,0), (57,0), (62,0)
  - FSP (spread): QB at (50,-5), RB at (43,-5),
    WR1 at (5,0), WR2 at (95,0), TE at (78,0)
    OL at (38,0), (43,0), (50,0), (57,0), (62,0)

  OL positions are identical across all three formations.
  The offensive line doesn't shift based on backfield alignment.

Coordinates: x = 0-100 (left-right percentage of field width)
             y = yards from line of scrimmage

COORDINATE SIGN CONVENTION:
  Both offense and defense use positive-y = upfield (toward
  opponent's end zone) and negative-y = behind the line of
  scrimmage.
  Offense example: QB at (50, -3) means 3 yards behind the LOS.
  Defense example: FS at (45, 18) means 18 yards downfield
  from LOS.
  All positions are relative to the LOS, not absolute field
  position.

ROUTE TYPES (each produces 6 waypoints, ~yards per phase):
  - Go: straight upfield (5, 12, 21, 30, 40, 50 yards deep)
  - Slant: diagonal cut inward (~3 yds/phase, ~4x/phase lateral)
  - Post: straight up 12yds then angle toward middle
  - Out: straight up 10yds then cut toward sideline
  - Curl: up 11yds then turn back to 8yds (comes back)
  - Flat: wide lateral toward sideline, slight upfield
  - Seam: straight up the middle (slight lateral offset)
  - Wheel: lateral then curves upfield deep
  - Check: short lateral check-down route
  - Block: stays at the line (blocking assignment)
  - Dive: RB straight up the middle
  - Sweep: RB goes wide to the edge
  - Counter: RB fakes one way then cuts back
  - Toss: RB receives pitch going wide
  - Screen: RB delays then goes to flat
  - Flee: RB runs forward then comes back (flea flicker)
  - Statue: RB sneaks behind the line (statue of liberty)

--- RUN PLAYS ---

1. Inside Run (staple, I-form, has RPO)
   RB goes straight up the middle through the linemen.
   Strong vs Nickel (lighter front), Weak vs Goal Line (packed box).
   WR1/WR2: go routes (decoy), TE: block, RB: dive
   RPO Quick Pass: TE breaks off block into flat route

2. Outside Run (staple, I-form, has RPO)
   RB takes it wide around the left edge.
   Strong vs Blitz (vacated edge), Weak vs Nickel (DB on edge).
   WR1: block, WR2: go, TE: block, RB: sweep left
   RPO Quick Pass: TE breaks off block into flat route

3. Counter (staple, I-form, has RPO)
   Fake one direction, cut back the other way.
   Strong vs Blitz (overcommit), Weak vs Goal Line (packed).
   WR1: go, WR2: block, TE: block, RB: counter right
   RPO Quick Pass: TE breaks off block into flat route

4. HB Toss (special, I-form, NO RPO — pitch already in air)
   Quick pitch wide right. Big play potential, fumble risk.
   8% fumble on pitch (modified by difficulty multiplier).
   Strong vs Goal Line (open edge), Weak vs Cover 2 (safety in flat).
   WR1: block, WR2: go, TE: flat right, RB: toss right

5. QB Sneak (special, I-form, NO RPO — pure push play)
   QB pushes forward behind the center. Short yardage (1-3 yards).
   QB starts 3 yards behind LOS with Goal Line DL within 1-2
   yards — contact triggers within 1-2 moves naturally, making
   this a short push play without special-case logic.
   Strong vs Nickel (light front), Weak vs Goal Line (stacked).
   Everyone blocks, QB pushes forward.

6. Power Run (staple, I-form, has RPO)
   RB follows a lead blocker into the gap. Almost guaranteed 2-3 yards.
   Strong vs Nickel (can't handle physicality), Weak vs Blitz
   (blitzers beat the lead blocker).
   WR1/WR2: go, TE: block, RB: dive
   RPO Quick Pass: TE breaks off block into flat route

7. Stretch Run (staple, I-form, has RPO)
   RB moves sideways reading blocks, looking for the first opening.
   High variance. Strong vs Blitz (cutback lanes), Weak vs Base 4-3
   (disciplined DEs seal the edge).
   WR1: go, WR2: block, TE: block, RB: sweep right
   RPO Quick Pass: TE breaks off block into flat route

--- PASS PLAYS ---

8. Quick Slants (staple, shotgun, primary: WR1)
   Receivers cut inside for a quick throw. Ball out fast.
   Strong vs Blitz (quick release beats rush), Weak vs Nickel
   (extra DB clogs middle).
   WR1: slant in, WR2: slant in, TE: seam, RB: check-down

9. Deep Post (staple, shotgun, primary: WR1)
   WR sprints deep then angles to the middle.
   Requires 2-3 phases of pocket time.
   Strong vs Goal Line (no deep coverage), Weak vs Blitz
   (rush arrives before route develops).
   WR1: post, WR2: out, TE: curl, RB: check-down

10. Play Action (situational, I-form, primary: WR1)
    Fake the handoff, then throw. Needs established run game.
    Strong vs Base 4-3 (LBs bite on fake), Weak vs Blitz
    (blitzers don't care about the fake).
    WR1: post, WR2: out, TE: seam, RB: dive (fake handoff)

11. Screen Pass (situational, shotgun, primary: RB)
    Let the rush come, dump to the RB with blockers.
    Strong vs Blitz (rushers fly past), Weak vs Base 4-3
    (patient LBs read the screen).
    WR1/WR2: go, TE: flat, RB: screen

12. Four Verticals (situational, spread, primary: WR1)
    Every receiver goes deep. 4 targets vs 2 safeties.
    Strong vs Base 4-3 (math advantage), Weak vs Blitz
    (need time for deep routes).
    WR1/WR2: go, TE: seam, RB: wheel

13. Out Routes (staple, shotgun, primary: WR1)
    Receivers cut toward the sideline at 8-12 yards.
    Reliable 8-12 yard gains. Low INT risk.
    Strong vs Blitz (vacated zones), Weak vs Cover 2
    (CBs sit in the out route area).
    WR1: out right, WR2: out left, TE: curl, RB: check-down

14. Curl/Comeback (staple, shotgun, primary: WR1)
    Receivers sprint upfield then turn back to the QB.
    Safe throw — receiver coming back toward the ball.
    Strong vs Cover 2 (big cushion), Weak vs Goal Line
    (press coverage jams at line).
    WR1/WR2: curl, TE: flat, RB: check-down

15. TE Seam (staple, shotgun, primary: TE)
    TE runs straight up the middle between LBs and safeties.
    10-20 yard gains. Needs pocket time.
    Strong vs Base 4-3 (LBs drop shallow), Weak vs Nickel
    (extra DB covers the seam).
    WR1: go, WR2: slant, TE: seam, RB: check-down

--- TRICK PLAYS ---

16. Flea Flicker (primary: WR1, 8% fumble on pitch-back)
    Fake handoff, RB pitches back to QB, throw deep.
    Fumble on pitch-back stops the play immediately — no
    passing phase follows. Fumble at the backfield spot.
    Strong vs Base 4-3 (safeties bite on fake), Weak vs Cover 2
    (two deep safeties don't bite).
    WR1: post, WR2: go, TE: seam, RB: flee (run forward, pitch back)

17. Statue of Liberty (primary: none — RB run)
    Fake a throw, secretly hand to the RB going wide.
    Strong vs Nickel (ignores backfield), Weak vs Blitz
    (blitzers crash backfield).
    WR1/WR2: go, TE: flat, RB: statue (sneak behind line)

18. QB Draw (QB run)
    Drop back like a pass, then QB runs through the gaps.
    Strong vs Blitz (rushers fly upfield past QB),
    Weak vs Goal Line (packed gaps).
    WR1/WR2: go, TE: block, RB: block

================================================================
MATCHUP MATRIX (play vs defense ratings)
================================================================

Scale: -2 (BAD) to +2 (GREAT)
Labels: -2=BAD, -1=RISKY, 0=OK, +1=GOOD, +2=GREAT

                    Base43  Nickel  Blitz  Cover2  GoalLine
Inside Run           0       1       1       0      -2
Outside Run          0      -1       1      -1       1
Counter              1       0       2       0      -2
HB Toss              0      -1       1      -1       1
QB Sneak             1       1       0       1      -1
Quick Slants         0      -1       2       1       2
Deep Post            1       0      -2       0       2
Play Action          1       0      -1       1       1
Screen Pass          0       0       2       0      -1
Four Verticals       1      -1      -2      -1       2
Flea Flicker         1       0      -1      -1       1
Statue of Liberty    1       1       0       0       0
QB Draw              0       1       2       0      -2
Power Run            0       2      -2       1      -1
Stretch Run         -1       1       2       0      -1
Out Routes           0      -1       2      -2       1
Curl/Comeback        1       0      -1       2      -2
TE Seam              2      -2       0      -1       1

================================================================
DEFENSES (5 formations)
================================================================

Each defense has: name, description, and 11 defender positions.
Positions in (x%, yards from LOS) format.
All y-values are positive (upfield from LOS toward offense).

Base 4-3 — "Standard defense. Balanced against run and pass."
  DL: DE1(34,1.5) DT1(46,1) DT2(54,1) DE2(66,1.5)
  LB: OLB1(28,5) MLB(50,5) OLB2(72,5)
  DB: CB1(10,7) CB2(90,7) SS(55,14) FS(45,18)

Nickel — "Coverage-focused alignment. DBs play tighter, weaker against the run."
  DL: DE1(36,1.5) DT1(46,1) DT2(54,1) DE2(64,1.5)
  LB: OLB1(30,5) MLB(50,5) OLB2(78,5)
  DB: CB1(8,6) CB2(92,6) SS(60,12) FS(40,18)

Blitz — "LBs rush QB. Extreme pressure, fewer defenders deep."
  DL: DE1(34,1.5) DT1(46,1) DT2(54,1) DE2(66,1.5)
  LB: OLB1(38,2.5) MLB(50,3) OLB2(62,2.5)
  DB: CB1(10,7) CB2(90,7) SS(55,8) FS(45,15)

Cover 2 — "Two deep safeties. Shuts down deep, short middle open."
  DL: DE1(34,1.5) DT1(46,1) DT2(54,1) DE2(66,1.5)
  LB: OLB1(30,5) MLB(50,5.5) OLB2(70,5)
  DB: CB1(10,4) CB2(90,4) SS(30,18) FS(70,18)

Goal Line — "Packed line. Stops short runs, zero deep coverage."
  DL: DE1(32,1) DT1(44,0.8) DT2(56,0.8) DE2(68,1)
  LB: OLB1(36,2.5) MLB(50,2.5) OLB2(64,2.5)
  DB: CB1(14,4) CB2(86,4) SS(50,8) FS(50,13)

Defense selection is weighted by game situation:
  - Base weights: 4-3=30, Nickel=22, Blitz=15, Cover2=22,
    Goal Line=0 (only appears in situational overrides below)
  - Near goal line (≥95 yds, ≤3 to go): Goal Line weight=50
  - Red zone (≥90 yds): Goal Line weight=20
  - Short yardage (≤2 to go): more Base, more Blitz
  - Long yardage (≥8 to go): more Nickel, more Cover 2
  - 3rd and long (dn≥3, dst≥5): more Nickel, Cover 2, Blitz

================================================================
RUN BLOCKING ASSIGNMENTS
================================================================

Each run play specifies which OL blocks which DL:
  - Inside Run: OL pairs off L-R, hole at center
  - Outside Run: everyone pushes same direction, hole at edge
  - Counter: push one way, hole cuts back opposite
  - HB Toss: everyone seals inside, hole at right edge
  - QB Sneak: collapse center
  - Power Run: create right B-gap
  - Stretch Run: push right, hole at right edge

OL hold times on run plays:
  The same hold time system from Pocket Integrity applies.
  Each runner action = 1 phase, ticking down OL hold times.
  On run plays, hold time determines how long each gap
  stays open — when a blocker's hold time expires, his gap
  closes (the defender he was blocking gets free and pursues
  the ball carrier). Early runner moves go through open gaps;
  later moves face closing lanes and pursuing defenders.
  This is why Dive Forward (take 1-3 safe yards) is valuable
  when the line is collapsing — you grab what's there before
  the gaps close.
Visual: OL dot color changes green→yellow→red.

================================================================
PASSING MECHANICS
================================================================

Phase-based system:
  - Phase advances with each QB action (drop back, look,
    roll, step up — ALL actions cost 1 phase, no exceptions)
  - Each phase: receiver routes progress one waypoint,
    OL hold times tick down, defender positions update
  - Optimal throw timing: phase 2-4 for most routes
  - Phase 5+: coverage closes back in

Openness system — distance-based (WYSIWYG):
  Openness is determined by physical distance between each
  receiver and the nearest defender on the field. What the
  player sees on the field IS the game state.

  Thresholds (nearest defender distance):
    Wide Open:  5+ yards from nearest defender
    Open:       3-4 yards
    Contested:  1.5-3 yards
    Covered:    <1.5 yards

  The openness label must match the numbers. If a receiver
  shows 75% completion and 0% INT, the label should NOT say
  "Covered." When a defensive scheme creates physical space
  for a receiver (e.g. Cover 2 leaves the TE seam open), that
  space shows in the actual distance calculation and the label
  adjusts accordingly.

  Color coding:
    Wide Open: green (#22c55e)
    Open: yellow-green
    Contested: yellow (#fbbf24)
    Covered: red (#ef4444)

DB tracking speed (main balance knob):
  Practice:   DBs track at 70% of WR speed
  Preseason:  DBs track at 82% of WR speed
  Regular:    DBs track at 90% of WR speed
  Playoffs:   DBs track at 100% of WR speed

  These are design-intent values, not from the working alpha
  (which used 70%/85%/100% for 3 levels). Subject to
  playtesting [TUNE]. The principle: defenders close on
  receivers faster at higher difficulties, so routes that get
  wide open in Practice are merely open or contested in
  Playoffs.

Look mechanic:
  - QB can look left or right (one action, costs one phase)
  - Physically shifts the safety in that direction
    (defenders on look side tighten by -1 yard,
     opposite side loosen by +1.5 yards)
  - The benefit IS the distance shift — because openness is
    WYSIWYG, moving the safety changes the actual distances,
    which changes openness labels, which changes completion %.
    No separate bonus on top of the distance shift.

Pocket integrity:
  - 5 OL vs DL matchups (4 DL standard, 5+ when blitzing)
  - Each matchup is independent with its own state

  HOLD TIME (rolled at snap):
    Each OL-DL matchup gets a "hold time" — the number of
    FULL phases that lineman holds his block before he starts
    losing. Hold time N means: green during phases 1 through
    N, starts dropping at phase N+1.

    Rolled randomly at snap from a difficulty-dependent range:

    Practice:   hold time 3-6 phases (pocket almost always holds)
    Preseason:  hold time 2-5 phases
    Regular:    hold time 1-4 phases
    Playoffs:   hold time 1-3 phases

    Blitz defense: all hold times reduced by 1 (min 1).
    Additional LB rushers get hold time 1-2 (they're crashing).

    This means: in Practice, the pocket rarely collapses before
    phase 4. In Playoffs, a hold time of 1 means green during
    phase 1, starts dropping at phase 2 — the QB has one action
    before that edge starts failing. Most plays fall somewhere
    in between, with one side weaker than the other.

  MATCHUP PROGRESSION:
    Before hold time expires: matchup integrity = 100% (green dot)
    Hold time expires: integrity starts dropping
      Phase after hold expires: ~70% (yellow dot)
      Two phases after: ~40% (yellow-red)
      Three phases after: ~10% (red dot)
      Four+ phases after: 0% (beaten, defender closing on QB)

    The decline is NOT linear — it accelerates. Once a lineman
    starts losing, he loses fast.

  LEFT/RIGHT AGGREGATION:
    Left pocket bar = average integrity of left-side matchups
      (LT, LG — OL positions 1-2)
    Right pocket bar = average integrity of right-side matchups
      (RG, RT — OL positions 4-5)
    Center (OL position 3) contributes to both sides at 50%.

    This creates asymmetric pocket collapse:
      - Left bar red, right bar green → roll right to escape
      - Both yellow → time is short but you can still throw
      - Both red → tuck the ball or scramble NOW
      - One side red early (phase 1-2) → edge was beaten,
        get rid of the ball or move away from pressure

  SACK MECHANICS:
    When a matchup reaches 0% integrity, that defender begins
    closing on the QB. Closing speed is difficulty-dependent:

    Practice:   slow close — 2 phases to reach QB
    Preseason:  moderate — 1.5 phases
    Regular:    fast — 1 phase
    Playoffs:   very fast — 0.5-1 phase

    If the QB hasn't thrown, scrambled, or tucked by the time
    the defender arrives: SACK. Yardage lost = QB's current
    distance behind the LOS (absolute value of y). QB at
    y = -7 after two drop backs = 7-yard sack.

    Multiple defenders can be closing simultaneously if multiple
    matchups hit 0%. Scrambling away from one closer might run
    into another.

    Auto-sack safety net (prevents new players from sitting
    forever):
      Practice: auto-sack after 3+ tuck warnings without action
      Preseason: auto-sack after 2+ tuck warnings

  VISUAL FEEDBACK:
    - OL dots on field: green → yellow → red per lineman
    - OL dot cluster in top-left: overview of all 5 matchups
    - Left/Right pocket bars: color-coded aggregate
    - Field warnings: "PRESSURE BUILDING" / "POCKET COLLAPSING"
      / "THROW OR TUCK!" with direction (LEFT / RIGHT / BOTH)
    - Defender position on field visually closes in on QB when
      matchup hits 0% — the player can SEE the sack coming

  DEFENSE-SPECIFIC POCKET BEHAVIOR:
    Base 4-3: most stable pocket. Standard 4 rushers,
      standard hold time distribution. No modifiers.
    Nickel: same as Base. Standard 4 rush, no hold time
      modifier. Nickel's advantage is in coverage, not
      pressure.
    Blitz: fastest collapse. All hold times -1 (min 1),
      plus LB rushers with hold time 1-2. High sack chance,
      but also leaves receivers uncovered.
    Cover 2: same as Base. Standard 4 rush, no hold time
      modifier. Cover 2's advantage is two deep safeties,
      not pressure.
    Goal Line: interior DT hold times -1 (packed bodies
      near the ball). Edge DE hold times unchanged (wider
      alignment, slower to reach QB). Run-stopping
      formation, not a pass-rush formation.

  TWO-PLAYER NOTE:
    Pocket variability works identically in two-player.
    The defensive player's formation choice determines the
    hold time distribution. Picking Blitz creates fast
    pressure; picking Cover 2 gives the offense more time.
    This is a real defensive strategy decision even though
    the defense player doesn't control individual rushers.

Blitz mechanics:
  - Blitz defense has LBs at tighter positions
  - Random uncovered receiver (WR2 or TE) when defense blitzes
  - Quick sack chance at snap (2%–20% based on difficulty)
  - All hold times reduced by 1, LB rushers get hold time 1-2
    (see Pocket Integrity above for full details)

================================================================
THROW RESOLUTION
================================================================

When QB throws to a target:
  - Ball flight animation: parabolic arc, duration scales
    with distance (300-900ms)
  - Completion roll based on openness + pocket state
  - Results: Complete / Incomplete / Drop / Interception
  - If complete: receiver becomes ball carrier at that spot

Throw probability formulas:
  Base completion %:
    0.40 + (openness_level * 0.15) + (receiver.skill * 0.02)
    - (pocket_pressure / 250)
    Capped at 95% before difficulty bonuses are applied.

  Where openness_level:
    Covered = 0, Contested = 1, Open = 2, Wide Open = 3

  Where pocket_pressure:
    100 - overall_pocket_percent
    (so 70% pocket integrity = pressure of 30 = penalty of
    30/250 = 0.12 or 12% completion penalty.
    Full pocket = 0 penalty. 20% pocket = 0.32 penalty.)

  Interception %:
    Covered (openness 0):   10% + (phase * 2%)
    Contested (openness 1): 4%
    Open (openness 2+):     1%

  Difficulty modifiers on throws:
    Practice:
      Completion: +30% (cap 98%)
      INT chance: x0.05
      Pressure penalty: halved
    Preseason:
      Completion: +20% (cap 95%)
      INT chance: x0.3
      Pressure penalty: halved
    Regular:
      Completion: +5% (cap 95%)
      INT chance: x0.7
      Pressure penalty: reduced 25%
    Playoffs:
      Completion: unchanged
      INT chance: unchanged
      Pressure penalty: unchanged

Drops:
  Drops are cosmetic — a subset of incompletions, not a
  separate roll. When a throw is incomplete AND the receiver
  was Wide Open or Open, there is a 15% chance [TUNE] the
  incompletion is labeled a "drop" instead. Same mechanical
  outcome (ball hits the ground, next down). Different
  commentary — Dan says the receiver had it and let it go,
  KikiTeach acknowledges the read was correct. This teaches
  the player "you made the right decision, the game just
  didn't reward you this time."

================================================================
INTENTIONAL GROUNDING
================================================================

When the QB picks Throw It Away, grounding risk applies:

Tackle box: OL x-span, approximately x = 38 to x = 62.

Grounding is called when:
  - QB is inside the tackle box (x between 38-62) AND
  - No receiver is within ~5 yards of the QB's position

If QB has rolled outside the tackles (x < 38 or x > 62),
Throw It Away is always legal. This gives Roll Left / Roll Right
a secondary tactical purpose: escape the tackles before
throwing it away.

Grounding penalty: loss of down + ball spotted at the throw
point (behind LOS). On 4th down, this is a turnover on downs
at a worse spot than a normal failure.

Difficulty scaling on grounding risk:
  Practice:   low risk (~20% when conditions are met) [TUNE]
  Preseason:  moderate risk (~50%)
  Regular:    full risk (~80%)
  Playoffs:   full risk (~90%)

================================================================
FIELD GOAL RESOLUTION
================================================================

Field goal distance = (100 - ball_position) + 17 yards
  where ball_position = yards from own goal (0-100)
  and 17 = 10 yards endzone depth + 7 yards snap distance.

Example: ball at the 75-yard line (opponent's 25).
  Distance = (100 - 75) + 17 = 42 yards.

Success rates by distance (step function):
  ≤30 yards:  95%
  31-40 yards: 82%
  41-50 yards: 60%
  51-55 yards: 35%
  >55 yards:  15%

After a make: +3 points, kickoff (Defensive Posture with CPU
  starting at their 25 in single-player; other player gets
  ball at own 25 in two-player).
After a miss: Defensive Posture with CPU starting at kick spot
  or their own 20, whichever gives the CPU better field position.
  In player coordinates: min(ball_position, 80).
  In two-player: other player gets ball at kick spot or own 20,
  whichever gives them better field position.

================================================================
YAC (Yards After Catch) SYSTEM
================================================================

Three tiers based on nearest defender distance at catch:

1. Immediate contact (defender within 3 yards):
   Contact screen triggers immediately — receiver catches
   the ball and is hit on the spot. One chance to break free
   through normal contact resolution (Spin/Juke/Stiff Arm/
   Dive Forward). If tackled, small gain (0-1 yards after catch).

2. One-move (defender 3-6 yards away):
   Enter runner mode for one opportunity to break free.
   Coach advice and directional arrows shown.

3. Open field (defender 6+ yards away):
   Bonus yards automatically added (3-8 yards [TUNE]),
   then may enter runner mode for additional gains.

Maximum 3 YAC actions (MAX_YAC = 3).

================================================================
PLAYER ATTRIBUTES
================================================================

Each position has numeric attributes on a 1-10 scale.
These feed into contact, throw, and pursuit formulas.
All values [TUNE].

  Position  Skill  Strength  Speed
  WR1       8      4         9
  WR2       7      4         8
  TE        6      7         6
  RB        7      7         8
  QB        5      5         6

Where attributes are used:
  Skill → throw completion (receiver.skill * 0.02),
    spin move (runner.skill / 13 + 0.10),
    juke (runner.skill / 12 + 0.15)
  Strength → stiff arm (runner.strength / 12 + 0.10)
  Speed → pursuit calculations via speed factor
    (WR 1.0, RB 0.9, TE 0.7, QB 0.75 — derived from speed
    stat, already applied in Contact System)

These are starting values, subject to playtesting.
Defenders do not have individual attributes in v1 — their
effectiveness comes from formation position and tracking speed.

================================================================
CONTACT SYSTEM
================================================================

When defender reaches ball carrier (within 1.5 yards [TUNE]):
  - Contact screen appears with red pulsing border
  - Shows: which defender, distance, nearby defenders count
  - Tackles broken counter (risk escalates with each break)

Options and base percentages:
  Spin Move:
    Base break free: runner.skill / 13 + 0.10
    Degrades per successive contact: -15% per broken tackle

  Juke:
    Base break free: runner.skill / 12 + 0.15
    Degrades per successive contact: -15% per broken tackle

  Stiff Arm:
    Base break free: runner.strength / 12 + 0.10
    Degrades per successive contact: -15% per broken tackle

  All three contact moves degrade identically: base rate on
  1st contact, base-15% on 2nd, base-30% on 3rd, base-45%
  on 4th. This makes each successive broken tackle harder
  regardless of which move the player picks.

  Dive Forward:
    Guaranteed +1-3 yards
    ZERO fumble risk
    Always available, always safe
    Does NOT degrade — it's always the safe option.

Fumble risk escalation:
  Base fumble rate per contact move [TUNE]:
    Spin Move: 8%
    Juke: 5%
    Stiff Arm: 6%
    Dive Forward: 0% (always)

  Escalation multiplier per successive contact:
    1st contact: x1.0
    2nd contact: x1.5
    3rd contact: x2.5
    4th contact: x4.0

  Difficulty fumble multipliers:
    Practice:   base fumble x0.05 (essentially zero)
    Preseason:  base fumble x0.30
    Regular:    base fumble x0.70
    Playoffs:   base fumble x1.00

  Toss/trick play fumble (HB Toss 8%, Flea Flicker 8%):
    Practice:   x0.30 (so ~2.4%)
    Preseason:  x0.30 (so ~2.4%)
    Regular:    x0.70 (so ~5.6%)
    Playoffs:   x1.00 (full 8%)

Speed factor by position (affects pursuit calculations):
  WR: 1.0, RB: 0.9, TE: 0.7, QB: 0.75

================================================================
DIFFICULTY LEVELS
================================================================

Practice:
  - Minimal HUD, big clear buttons
  - Coach explains everything
  - Tutorial overlays at each game phase (including
    Defensive Posture, ghost route distinction, and
    extreme pressure explanations)
  - Auto-sack after 3+ tuck warnings
  - Very low fumble risk (5% multiplier)
  - Very low blitz sack chance (2%)
  - Best throws highlighted with star and explanation
  - Ghost routes shown at play select, presnap, and during play
  - DB tracking speed: 70%
  - OL hold times: 3-6 phases
  - Pursuit speed: 70%
  - Completion bonus: +30% (cap 98%)
  - INT multiplier: x0.05
  - Pressure penalty: halved
  - Grounding risk: low (~20%)
  - CPU scoring: base odds x0.6
  - Audibles: unlimited

Preseason:
  - Adds openness labels, matchup info
  - Coach gives detailed teaching commentary (KikiTeach)
  - Auto-sack after 2+ tuck warnings
  - Reduced fumble risk (30% multiplier)
  - Low blitz sack chance (5%)
  - Best throws highlighted
  - Matchup warnings with clickable suggestions
  - Ghost routes shown at play select, presnap, and during play
  - DB tracking speed: 82%
  - OL hold times: 2-5 phases
  - Pursuit speed: 82%
  - Completion bonus: +20% (cap 95%)
  - INT multiplier: x0.3
  - Pressure penalty: halved
  - Grounding risk: moderate (~50%)
  - CPU scoring: base odds x0.8
  - Audibles: unlimited

Regular Season:
  - Full control panel
  - Pocket integrity display, look buttons
  - Terse coach
  - Standard fumble risk (70% multiplier)
  - Standard blitz sack chance (12%)
  - Only warns on bad/terrible matchups
  - Ghost routes shown at play select and presnap only
  - DB tracking speed: 90%
  - OL hold times: 1-4 phases
  - Pursuit speed: 90%
  - Completion bonus: +5% (cap 95%)
  - INT multiplier: x0.7
  - Pressure penalty: reduced 25%
  - Grounding risk: full (~80%)
  - CPU scoring: base odds x1.0
  - Audibles: 2 per quarter

Playoffs:
  - Same panel, coach mostly silent
  - Full fumble risk (100% multiplier)
  - High blitz sack chance (20%)
  - No matchup warnings
  - No teaching commentary
  - Ghost routes shown at play select only
  - DB tracking speed: 100%
  - OL hold times: 1-3 phases
  - Pursuit speed: 100%
  - Completion bonus: none
  - INT multiplier: x1.0
  - Pressure penalty: unchanged
  - Grounding risk: full (~90%)
  - CPU scoring: base odds x1.2
  - Audibles: 1 per quarter
  - Pure skill

================================================================
SOUND EFFECTS (Web Audio API synthesized)
================================================================

All synthesized, no audio files:
  - Tick: short dry click for button taps
  - Snap: punchy pop (triangle wave + noise burst)
  - Tackle: deep sub-bass thud + impact transient
  - Crowd: shaped noise, swells up then fades (0.5s)
  - Whistle: referee whistle tone (2600Hz → 2000Hz sine)
  - Sack buzz: harsh growling low tone (sawtooth 80→35Hz)

Mute/unmute toggle in scoreboard.

================================================================
CSS ANIMATIONS
================================================================

  - ballFly: @keyframes for ball flight parabolic arc
  - pulse: breathing opacity/scale for warnings
  - fw: fireworks particle spread + fade
  - tdBounce: touchdown emoji entrance (scale up → bounce)
  - tdGlow: touchdown text golden glow pulsing
  - contactPulse: red shadow pulsing for contact/pressure borders
  - qbGlow: subtle blue glow on QB during pocket phase
  - alertFade: field alert text fade in → hold → fade out

================================================================
GAME STATE DATA
================================================================

Game tracking:
  - Ball position (yards from own goal, 0-100, min clamped at 1)
  - Down (1-4)
  - Distance to first down
  - Score (player, CPU)
  - Quarter (1-4)
  - Possession number within quarter (1-2)
  - Total possessions used (1-8)
  - Audibles used this quarter (counter, resets each quarter)
  - Play result history (last plays array)
  - Play-by-play log for current play

Scoring:
  - Touchdown = 7 points (automatic PAT assumed)
  - Field goal = 3 points (see Field Goal Resolution)
  - CPU scores via Defensive Posture system (FG = 3, TD = 7)
  - Safety = 2 points (FUTURE — not in v1)

After each play:
  - Update ball position, down, distance
  - First down if gained enough
  - Turnover on downs if 4th down fails → Defensive Posture
  - Punt → Defensive Posture (see Field Position Rules)
  - Field goal → attempt, then Defensive Posture
  - Touchdown → score, then Defensive Posture (kickoff)
  - Interception / Fumble → Defensive Posture

Game log: running list of play results, fades with age.

================================================================
EASTER EGG — THE MELEFICENT (future, not in v1)
================================================================

A 92-yard field goal attempt.
15+ random failure messages when it (almost always) fails.
Secret success trigger (details TBD in design phase).

================================================================
TWO-PLAYER MODE
================================================================

  - Same device, pass back and forth
  - Regular Season difficulty
  - Coin toss at game start: winner chooses to receive or defer
    (teaches real football strategy — deferring gives you last
    possession of the game + two consecutive possessions across
    halftime)

  Possession order (if Player A receives first):
    Q1: A1, B1, A2, B2
    Q2: A3, B3, A4, B4
    Q3: B5, A5, B6, A6  ← B receives to start second half
    Q4: B7, A7, B8, A8  ← A has final possession
    Each player gets exactly 8 possessions.

  - Player on offense calls plays normally
  - Player on defense picks defense via rotated 180° screen
    (pass the device — defense is a blind pick before offense
    sees presnap)
  - 1 audible per quarter for offensive player
  - No audible for defensive player (nothing to react to)

  - NO Defensive Posture in two-player — after a punt,
    turnover, or score, possession flips to the other player
    who becomes the offense
  - Other player gets ball at the appropriate field position:
    After TD: own 25 (kickoff equivalent)
    After punt: punt landing spot (see Field Position Rules)
    After turnover on downs: at the spot
    After INT: at approximately the receiver's position
      (if past the end zone, touchback — own 20, same as
      single-player rule). The receiver's position is in
      the throwing player's coordinate system — convert to
      the receiving player's coordinates (100 minus the
      value) before placing the ball.
    After fumble: at the spot
    After made FG: own 25 (kickoff equivalent)
    After missed FG: at kick spot or own 20, whichever
      gives them better field position

  - Football emoji shows who has the ball
  - Halftime screen marks the possession flip
  - End of game: final score, win/loss/tie
  - Tie declared for v1 (overtime in future update)
  - Player who goes first has slight advantage (last possession
    in game) — coin toss makes this fair

================================================================
VISUAL DIRECTION
================================================================

Not a terminal debug view. A game.

  - Player tokens: large, team-colored, shaped by position,
    readable as characters on a field
  - Field: bright greens, visible yard markings, colored end zones
  - Play selection: tappable cards, not text lists
  - Animations: smooth glides, ball flights that feel like throws
  - UI layout: broadcast-overlay feel for score/down/distance

UI scales with difficulty:
  - Practice: minimal HUD, big buttons, coach explains everything
  - Preseason: adds openness labels, basic info
  - Regular: full control panel, pocket display, look buttons
  - Playoffs: same panel, less hand-holding

Future direction: anthropomorphic player figures (not for v1,
but rendering should separate position from appearance so tokens
can evolve to sprites without rewriting the position system)

Kip liked Gemini's visual style (more game-like, cartoon feel)
but preferred the control panel from the current build, especially
for Regular Season difficulty. The control panel may be too busy
for Practice/Preseason — the UI simplification per difficulty
addresses this.

================================================================
SAFEGUARDS
================================================================

  - Design doc written entirely in plain English and tables
  - No code in the build prompt — only prose
  - tacfoot5.html and tacfoot4.html removed from build machine
  - Screenshots uploaded as visual reference only
  - Fingerprint test: search finished file for old variable names
    (qbP, bcP, defSch, opn, recSeeds, blitzUncovered, camYRef,
    noTransition, offP, OL_IDS, pLR, OL_BLK,
    isPre, fmtTarget, ndInfo, dlS, blS) — if any appear, the
    build cheated
    NOTE: FI, FSG, FSP are intentional formation names in this
    spec — they are NOT legacy contamination.
  - THREE STRIKE RULE remains: if a bug fix fails twice, stop

================================================================
WORKFLOW
================================================================

  - Design discussion in chat, builds through Claude Code
  - ABSOLUTE RULE: Never write a build prompt until Kip says
    "write the prompt" or "go" or "let's build"
  - Claude Code builds end with plain-English DONE note
  - Never use "scratches the itch"
  - Never end replies with follow-up questions

================================================================
FUTURE ADDITIONS (not in v1, noted for later)
================================================================

  - Mechanical clock: run plays cost more time, CPU drives eat
    clock, two-minute drill strategy
  - Overtime for tied games
  - Safety scoring (2 points + safety kick)
  - CPU drive types (run-heavy vs pass-heavy) making Ball Hawk
    posture mechanically distinct from Bend Don't Break
  - Defensive play involvement in Regular/Playoffs single-player
  - The Meleficent easter egg
  - Anthropomorphic player sprites
  - Season mode and stats tracking
  - More trick plays
  - Phone-responsive layout
