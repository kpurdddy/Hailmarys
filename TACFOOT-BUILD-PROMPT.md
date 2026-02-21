TACFOOT REWRITE — BUILD PROMPT

Build a complete single-file React game called Tactical Football
from the design document TACFOOT-REWRITE-HANDOFF-v3.md. That
document is the sole source of truth. Every mechanic, formula,
number, and visual spec is defined there. Do not reference any
previous TacFoot code — this is a ground-up rewrite.

THE FILE
  - Output: a single HTML file containing all React, CSS, and
    game logic inline
  - Use React via CDN (React 18, ReactDOM, Babel standalone)
  - No external dependencies beyond React and Babel
  - No build tools, no npm, no bundler

ARCHITECTURE (from the doc)
  - useReducer with a single state object
  - One dispatch per game tick, no rapid-fire chains
  - Game logic as pure functions: state in, state out
  - Camera position stored as a ref, never in React state,
    never triggers re-renders
  - CSS class-based transitions only — no inline transition
    manipulation, no noTransition flags
  - Single render pass: game coordinates to pixel positions
  - No dual position tracking systems

WHAT TO BUILD (in this order)
  1. Game state machine: all 17 states from the doc, with
     correct transitions between them including the post-snap
     branching (pass → Decision, RPO runs → RPO Read, non-RPO
     runs → Runner Mode, trick play branches)
  2. Field rendering: green field with yard lines, hash marks,
     end zones, goal posts, line of scrimmage, first down
     marker, midfield logo — all specs in the doc
  3. Player tokens: shaped by position (diamond QB, circle WR,
     etc.), colored by team, labeled, with drop shadows
  4. Title screen and difficulty select with coach pick
  5. Play select menu with categories, play cards, ghost routes
  6. Presnap screen with defense display and matchup feedback
  7. Decision phase (passing): pocket integrity with hold time
     engine, phase tracking, QB position tracking, throw targets
     with openness labels, all action buttons with defined
     movement amounts, pocket escalation tiers, scramble lane
     indicator
  8. Throw resolution: completion formula with pocket_pressure
     defined as (100 - overall_pocket_percent), INT rates, drops
     as cosmetic subset of incompletions, ball flight animation
  9. RPO Read for applicable run plays, with quick pass
     resolution as a single roll (not full Decision screen)
  10. Runner Mode: carrier starts at current field position,
      position-specific move labels mapped to generic mechanics,
      defined movement distances per action, contact trigger at
      1.5 yards, out of bounds at x <= 0 or x >= 100, pursuit
      speed using same difficulty scale as DB tracking, no hard
      cap on run moves (pursuit escalation is the limiter)
  11. Contact system: break/fumble/tackle percentages from the
      doc, all three moves degrade at -15% per broken tackle,
      base fumble rates (Spin 8%, Juke 5%, Stiff Arm 6%),
      Dive Forward always safe
  12. Sack mechanics: yardage lost = QB distance behind LOS,
      Tuck the Ball = fixed 1-2 yard loss
  13. YAC system with three tiers at 3/6 yard thresholds
  14. Field goal resolution: distance = (100 - ball_position) + 17,
      success rates from the doc's step function
  15. Field position rules for every transition: TD, punt, INT
      (with touchback clamp), fumble, turnover on downs, missed
      FG (min of kick spot and 80 in player coordinates)
  16. Defensive Posture: three choices, CPU scoring odds table,
      all multiplicative posture modifiers, difficulty scaling,
      CPU turnover spot (random 5-25 yard advance capped at
      player's 10), coach debrief
  17. Intentional grounding: tackle box check against QB tracked
      x-position, difficulty scaling on grounding risk
  18. Scoreboard, down & distance HUD, runner info HUD
  19. Commentary system: Dan and Kiki with all categories
  20. Coach system: Baby Boy Joey and Grizzled Jim with
      difficulty-adaptive advice
  21. Practice overlays (teaching cards at first encounter of
      each game state)
  22. Two-player mode: coin toss, 180-degree defense select,
      possession order from the doc, field position rules
      including INT coordinate flip (100 minus throwing player's
      value), FG rules, no Defensive Posture
  23. Sound effects: Web Audio API synthesized (tick, snap,
      tackle, crowd, whistle, sack buzz)
  24. All 18 plays with formations, routes, matchup matrix
  25. All 5 defenses with positions and weighted selection
  26. All 4 difficulty levels with every parameter from the
      Difficulty Levels section (DB tracking, hold times,
      pursuit speed, completion bonuses, INT multipliers,
      fumble multipliers, grounding risk, CPU scoring, audible
      limits, ghost route visibility, auto-sack thresholds)

WHAT NOT TO DO
  - Do not look at, reference, or copy from any previous
    tacfoot HTML files
  - Do not use requestAnimationFrame for a game loop — this is
    turn-based, pure React rendering with CSS transitions
  - Do not create dual position tracking (one system only)
  - Do not use inline transition style manipulation
  - Values marked [TUNE] in the doc are starting defaults —
    implement them as written, they'll be adjusted in
    playtesting

FINGERPRINT TEST
  When the build is complete, search the output file for these
  old variable names. If ANY appear, the build contaminated
  itself with old code and must be redone:
    qbP, bcP, defSch, opn, recSeeds, blitzUncovered, camYRef,
    noTransition, offP, OL_IDS, pLR, FI, FSG, FSP, OL_BLK,
    isPre, fmtTarget, ndInfo, dlS, blS

DONE NOTE
  When finished, write a plain-English summary: what was built,
  where the file is saved, and anything that needs attention.
  No jargon, no prompts to approve.
