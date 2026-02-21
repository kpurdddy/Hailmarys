TACFOOT REWRITE HANDOFF — February 20, 2026

This document captures decisions and complete game spec from the
diagnosis conversation. No code. No old variable names.
This is the starting point for the design doc thread.

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
     planned receiver routes with arrowheads)
     Blue for WR/TE routes, orange for RB routes

4. PRESNAP
   - Defense has been chosen (random, weighted by situation)
   - Shows: defense name, matchup rating for selected play
   - Matchup feedback varies by difficulty:
     Practice: nothing shown
     Preseason: "Great call!" / "Good call" / "Decent look" /
       "Careful — they're in X" / "This is a BAD call against X"
       with clickable suggestions for better plays
     Regular: only warns on bad (-1) and terrible (-2) matchups
     Playoffs: no feedback
   - Coach advice line (one-liner based on matchup)
   - Practice mode: tutorial overlay explaining the defense
   - Two buttons: SNAP and AUDIBLE
   - Ghost route lines still visible on field
   - SNAP starts the play
   - AUDIBLE returns to play select

5. SNAPPING / ANIMATING
   - Brief transition state
   - "Ball snapped..." or "Play developing..." text
   - Ball flight animation: center snaps to QB (parabolic arc)
   - No player input during this state

6. DECISION (pocket phase — passing plays)
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
   - Scramble lane button (when QB has a running lane)
   - Action buttons below throws:
     Drop Back, Roll Left, Roll Right, Step Up
     Look Left, Look Right (shifts safety)
     Throw Away, Tuck & Run, Tuck the Ball
   - Practice/Preseason: highlights recommended actions,
     shows explanatory tips about why targets are open
   - When pocket is collapsing: on-field warning text
     "POCKET COLLAPSING" (yellow) or "THROW OR TUCK!" (red)
     Shows direction of pressure (LEFT / RIGHT / BOTH SIDES)

7. PRESSURE (pocket broken — urgent)
   - Red pulsing border: "PRESSURE!"
   - Reduced options: Throw Away, Tuck & Run,
     Roll Left, Roll Right
   - Each shows success % 
   - "TUCK THE BALL" highlighted if pocket is critical
   - Sack odds visible

8. RPO READ (run-pass option on run plays)
   - POST-SNAP READ header
   - Scenario indicator: "THERE'S A HOLE!" (green) or
     "DEFENDER CLOSED THE LANE" (red) or
     "NO ROOM OUTSIDE" (red)
   - Three buttons: Hand Off / QB Keep / Quick Pass
   - Each shows success %, description, color coding
   - Practice labels are friendlier than Regular/Playoffs

9. RUNNER MODE (ball carrier has the ball)
   - "[CARRIER] — MAKE A MOVE" header
   - YAC counter (catches: "YAC 0/3") or move counter
   - Nearest defender warning: name, distance, color-coded
     Red (<3 yds) / Yellow (3-6 yds) / Green (>6 yds)
   - Action buttons in 2x2 grid, each with:
     Label, success %, description
     Color-coded by type
   - Run arrows on the field showing available directions

10. CONTACT (defender reaches carrier)
    - Red pulsing border: "CONTACT!"
    - Shows which defender, distance, defenders nearby
    - Tackles broken this play counter (risk escalates)
    - Options grid:
      Spin Move: break %, fumble %, tackle %
      Juke: break %, fumble %, tackle %
      Stiff Arm: break %, fumble %, tackle %
      Dive Forward: guaranteed +1-3 yards, NO fumble risk
    - Contact flash overlay on field shows move + break %

11. RESULT
    - Compact bar showing: yardage, description
    - Color: red=turnover/loss, yellow=incomplete,
      green=big gain, white=normal
    - NEXT button
    - Play-by-play log in monospace at bottom
    - Practice/Preseason: coaching debrief explaining what happened

12. TOUCHDOWN
    - Full overlay: football emoji bouncing, "TOUCHDOWN!" glowing
    - Fireworks particles (5 bursts, 18 particles each, 6 colors)
    - "+7 POINTS" display
    - Play description and play-by-play chain
    - KICKOFF button
    - Crowd sound effect

13. FOURTH DOWN OPTIONS
    - Three buttons: Punt / Field Goal (shows distance) / Go For It

14. TWO-PLAYER DEFENSE SELECT
    - Rotated 180° (pass the device)
    - Defense picker for player 2

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
  - 50% opacity

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

Receiver openness labels (during decision/pressure):
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
  - Drop (receiver was open)
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

Layout: broadcast-style overlay at top of field.
"ANNOUNCER'S BOOTH" header with gold underline.
Dan's line first, then Kiki's. Dismiss button.

================================================================
COACH SYSTEM
================================================================

Two coaches:

"Baby Boy Joey" — The Gunslinger (Type A):
  - Blue themed (#4a9eff)
  - SVG avatar: young face, cap, blue jacket
  - Aggressive advice: "Let it fly!" / "Make him miss!"
  - Pushes for big plays

"Grizzled Jim" — The Old Fox (Type B):
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
  - FSP (spread): QB at (50,-5), RB at (43,-5),
    WR1 at (5,0), WR2 at (95,0), TE at (78,0)

Coordinates: x = 0-100 (left-right percentage of field width)
             y = yards from line of scrimmage (negative = behind)

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

1. Inside Run (staple, I-form)
   RB goes straight up the middle through the linemen.
   Strong vs Nickel (lighter front), Weak vs Goal Line (packed box).
   WR1/WR2: go routes (decoy), TE: block, RB: dive

2. Outside Run (staple, I-form)
   RB takes it wide around the left edge.
   Strong vs Blitz (vacated edge), Weak vs Nickel (DB on edge).
   WR1: block, WR2: go, TE: block, RB: sweep left

3. Counter (staple, I-form)
   Fake one direction, cut back the other way.
   Strong vs Blitz (overcommit), Weak vs Goal Line (packed).
   WR1: go, WR2: block, TE: block, RB: counter right

4. HB Toss (special, I-form, 8% fumble on pitch)
   Quick pitch wide right. Big play potential, fumble risk.
   Strong vs Goal Line (open edge), Weak vs Cover 2 (safety in flat).
   WR1: block, WR2: go, TE: flat right, RB: toss right

5. QB Sneak (special, I-form)
   QB pushes forward behind the center. Short yardage (1-3 yards).
   Strong vs Nickel (light front), Weak vs Goal Line (stacked).
   Everyone blocks, QB pushes forward.

6. Power Run (staple, I-form)
   RB follows a lead blocker into the gap. Almost guaranteed 2-3 yards.
   Strong vs Nickel (can't handle physicality), Weak vs Blitz
   (blitzers beat the lead blocker).
   WR1/WR2: go, TE: block, RB: dive

7. Stretch Run (staple, I-form)
   RB moves sideways reading blocks, looking for the first opening.
   High variance. Strong vs Blitz (cutback lanes), Weak vs Base 4-3
   (disciplined DEs seal the edge).
   WR1: go, WR2: block, TE: block, RB: sweep right

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
Positions in (x%, yards from LOS) format:

Base 4-3 — "Standard defense. Balanced against run and pass."
  DL: DE1(34,1.5) DT1(46,1) DT2(54,1) DE2(66,1.5)
  LB: OLB1(28,5) MLB(50,5) OLB2(72,5)
  DB: CB1(10,7) CB2(90,7) SS(55,14) FS(45,18)

Nickel — "Extra DB in coverage. Strong vs pass, weaker run."
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
  - Base weights: 4-3=30, Nickel=22, Blitz=15, Cover2=22
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

OL integrity degrades each phase as DL wins matchups.
Visual: OL dot color changes green→yellow→red.

================================================================
PASSING MECHANICS
================================================================

Phase-based system:
  - Phase advances with each QB action (drop back, scramble, etc.)
  - Receiver routes progress one waypoint per phase
  - Phases 1-4: receivers getting more open (seeds increase)
  - Phase 5+: coverage recovers (seeds decrease)
  - Optimal throw timing: phase 2-4 for most routes

Openness calculation:
  - Base openness from defense coverage type vs receiver
  - Modified by receiver seed (evolves per phase)
  - Modified by look mechanic (looking L shifts safety R)
  - 5 levels: Wide Open / Open / Contested / Covered / Locked Down
  - Color: green (#22c55e) → yellow (#fbbf24) → red (#ef4444)
  - Completion % and INT % derived from openness + coverage modifiers

Coverage modifiers per defense:
  Each defense has per-receiver completion % and INT % adjustments
  (e.g., Cover 2 gives WR1/WR2 -8% completion, +3% INT)

Look mechanic:
  - QB can look left or right (one action)
  - Shifts the safety in that direction
  - Opens up the opposite side
  - Uses one phase tick

Pocket integrity:
  - 5 OL vs 4-5 DL matchups
  - Each matchup has an integrity value (0-100%)
  - Degrades each phase
  - Left/Right pocket tracked separately
  - When overall < 50%: "PRESSURE BUILDING" alert
  - When overall < 30%: "POCKET COLLAPSING" alert
  - Triggers sack check each phase

Blitz mechanics:
  - Blitz defense has LBs at tighter positions
  - Random uncovered receiver (WR2 or TE) when defense blitzes
  - Quick sack chance at snap (2%–20% based on difficulty)
  - Pocket degrades faster

================================================================
THROW RESOLUTION
================================================================

When QB throws to a target:
  - Ball flight animation: parabolic arc, duration scales
    with distance (300-900ms)
  - Completion roll based on openness, coverage modifier,
    pocket integrity
  - Results: Complete / Incomplete / Drop / Interception
  - If complete: receiver becomes ball carrier at that spot

Drops (wide open receiver misses catch):
  - Only when receiver was open
  - Multiple random failure messages
  - KikiTeach acknowledges the read was correct

================================================================
YAC (Yards After Catch) SYSTEM
================================================================

Three tiers based on nearest defender distance at catch:

1. Immediate tackle (defender < threshold):
   Small gain (0-1 yards after catch), tackle on contact.

2. One-move (defender medium distance):
   Enter runner mode for one opportunity to break free.
   Coach advice and directional arrows shown.

3. Open field (defender far away):
   Bonus yards automatically added (3-8 yards),
   then may enter runner mode for additional gains.

Maximum 3 YAC actions (MAX_YAC = 3).

================================================================
CONTACT SYSTEM
================================================================

When defender reaches ball carrier:
  - Contact screen appears with red pulsing border
  - Shows: which defender, distance, nearby defenders count
  - Tackles broken counter (risk escalates with each break)

Options:
  - Spin Move: % break free / % fumble / % tackled
  - Juke: same structure
  - Stiff Arm: same structure
  - Dive Forward: guaranteed +1-3 yards, ZERO fumble risk

Fumble risk escalates after each broken tackle.
Speed factor varies by position: WR=1.0, RB=0.9, TE=0.7, QB=0.75

================================================================
DIFFICULTY LEVELS
================================================================

Practice:
  - Minimal HUD, big clear buttons
  - Coach explains everything
  - Tutorial overlays at each game phase
  - Auto-sack after 3+ tuck warnings
  - Very low fumble risk (5% multiplier)
  - Very low blitz sack chance (2%)
  - Best throws highlighted with star and explanation
  - Ghost routes always shown during play
  - No score tracking emphasis

Preseason:
  - Adds openness labels, matchup info
  - Coach gives detailed teaching commentary (KikiTeach)
  - Auto-sack after 2+ tuck warnings
  - Reduced fumble risk (30% multiplier)
  - Low blitz sack chance (5%)
  - Best throws highlighted
  - Matchup warnings with clickable suggestions
  - Ghost routes shown during play

Regular Season:
  - Full control panel
  - Pocket integrity display, look buttons
  - Terse coach
  - Standard fumble risk (60% multiplier)
  - Standard blitz sack chance (12%)
  - Only warns on bad/terrible matchups
  - Ghost routes only at presnap

Playoffs:
  - Same panel, coach mostly silent
  - Full fumble risk (100% multiplier)
  - High blitz sack chance (20%)
  - No matchup warnings
  - No teaching commentary
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
  - Ball position (yards from own goal, 0-100)
  - Down (1-4)
  - Distance to first down
  - Score (player 1, player 2)
  - Quarter
  - Play result history (last plays array)
  - Play-by-play log for current play

Scoring:
  - Touchdown = 7 points (automatic PAT assumed)
  - Field goal = 3 points (calculated from distance)
  - No turnovers scored by defense (just possession change)

After each play:
  - Update ball position, down, distance
  - First down if gained enough
  - Turnover on downs if 4th down fails
  - Punt: ball goes to own 25
  - Field goal: attempt from current position + 17 yards
  - Touchdown: reset to own 25

Game log: running list of play results, fades with age.

================================================================
COACHES — PERSONALITY DESCRIPTIONS
================================================================

"Baby Boy Joey" — The Gunslinger:
  "Young, cocksure, and always scheming. Calls plays that make
  defensive coordinators swear. When they work, they're
  spectacular — and when they don't, his grit and hustle
  already have the next one drawn up."

"Grizzled Jim" — The Old Fox:
  "Forty years on the sideline. Players run through walls for
  him. Knows the game cold, never panics, and a steadying
  presence when things get crazy."

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
  - Player 1 calls offense, Player 2 picks defense
  - Defense selection screen is rotated 180° (upside down)
  - Alternating possession after each drive
  - Football emoji shows who has the ball

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
    noTransition, offP, OL_IDS, pLR, FI, FSG, FSP, OL_BLK,
    isPre, fmtTarget, ndInfo, dlS, blS) — if any appear, the
    build cheated
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
