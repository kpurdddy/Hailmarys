# Hail Mary's — Build Prompt: Alpha 2.1.1 Hotfix

**Source file:** `index.html` (6659 lines, Alpha 2.1.0)
**Estimated time:** 15–20 minutes
**Estimated tokens:** 20–30k

---

## BEFORE YOU START

1. `cd D:\TacFootball`
2. `cp index.html tacfoot-rewrite-backup-2.1.0.html`
3. Verify backup exists before touching anything

---

## ITEM 1: Follow Camera Zoom — `/40` → `/55`

**What:** Camera zooms too aggressively during RUNNER/CONTACT. Shows only 40 yards of field. Need ~55.

**Where:** Line ~4226

```javascript
// CHANGE THIS:
const followYardPx = viewportHeight / 40;

// TO THIS:
const followYardPx = viewportHeight / 55;
```

One number change. Nothing else.

---

## ITEM 2: WR/TE "Run Upfield" Move

**What:** When Sprint is FATIGUED, WR/TE carriers have zero forward movement options. Add a basic forward run.

**Where:** `getRunnerMoveLabels()` function, line ~1820. The WR/TE block is the fallback `return` at the bottom (~line 1838).

```javascript
// CURRENT WR/TE list:
return [
  { label: 'Sprint', type: 'Sprint', description: 'Burst forward with speed' },
  { label: 'Cut Outside', type: 'CutOutside', description: 'Break to the sideline' },
  { label: 'Juke', type: 'Juke', description: 'Quick cut to evade' },
  { label: 'Dive Forward', type: 'DiveForward', description: 'Fall forward for safe yards' }
];

// CHANGE TO:
return [
  { label: 'Sprint', type: 'Sprint', description: 'Burst forward with speed' },
  { label: 'Run Upfield', type: 'HitTheHole', description: 'Push forward through traffic' },
  { label: 'Cut Outside', type: 'CutOutside', description: 'Break to the sideline' },
  { label: 'Juke', type: 'Juke', description: 'Quick cut to evade' },
  { label: 'Dive Forward', type: 'DiveForward', description: 'Fall forward for safe yards' }
];
```

Uses existing `HitTheHole` type — same 2–4 yard forward movement as RB's Hit the Hole. No new switch case needed in `RUNNER_MOVE`. The RunnerScreen button loop already handles any number of moves.

---

## ITEM 3: Fallback Defense Tokens on MENU/PLAY_SELECT

**What:** Field looks half-empty during MENU and PLAY_SELECT because only offense renders. Add Base 4-3 defense as visual decoration.

**Where:** The fallback rendering block starting at line ~4668. After the receivers loop ends (~line 4735), BEFORE the closing `}` of the `if (fallbackPhases.includes(...))` block.

**Add this block after the receiver loop (after line ~4735):**

```javascript
// Defense tokens (visual only — Base 4-3 default)
var fallbackDefense = DEFENSES[0].players; // base43
for (var di = 0; di < fallbackDefense.length; di++) {
  var def = fallbackDefense[di];
  var defPx = posToPixel(def.x, def.y);
  playerTokens.push(
    React.createElement(PlayerToken, {
      key: 'fallback-def-' + def.id,
      id: def.id,
      role: def.role,
      x: defPx.left,
      y: defPx.top,
      isBallCarrier: false,
      isOffense: false,
      label: '',
      instant: true,
      scale: tokenScale,
    })
  );
}
```

**IMPORTANT:** This goes ONLY in the `else` block (the fallback for MENU/PLAY_SELECT/FOURTH_DOWN/CPU_RESULT), NOT in the RESULT sub-block that renders final positions. The RESULT sub-block already has its own defense rendering from `state.currentDefenderPos`.

---

## ITEM 4: Jim Emoji → 🦊

**What:** Jim shows 🧠 (brain). Should be 🦊 (fox) to match his "The Old Fox" subtitle.

**Where:** Line ~5169 in the `coachCard` function inside `TitleScreen`.

```javascript
// CHANGE THIS:
name === 'joey' ? '\u{1F3C8}' : '\u{1F9E0}'

// TO THIS:
name === 'joey' ? '\u{1F3C8}' : '\u{1F98A}'
```

`\u{1F98A}` = 🦊. One character change.

**NOTE:** This line gets REPLACED entirely by Item 7 (SVG portraits). Include the emoji fix anyway as fallback in case the SVG conversion has issues — Item 7 overwrites this line.

---

## ITEM 5: Game Over — `reload()` → `RESTART_GAME`

**What:** "Play Again" button does `window.location.reload()` causing a hard browser flash. Should use the existing `RESTART_GAME` reducer action.

**Where:** `GameOverScreen` function. Search for `window.location.reload`.

```javascript
// CHANGE THIS:
onClick: function() { window.location.reload(); }

// TO THIS:
onClick: function() { dispatch({ type: 'RESTART_GAME' }); }
```

The `RESTART_GAME` case already exists (~line 4038) and resets to `INITIAL_STATE`. No new code needed.

---

## ITEM 6: Activate Sound Engine

**What:** A complete Web Audio API `SoundEngine` exists (~line 512, ~100 lines) with sounds for snap, tackle, whistle, crowd, and sackBuzz. It is never called anywhere. Wire it up.

**CRITICAL:** Sound calls go in COMPONENTS (useEffect, onClick handlers), NOT in the reducer. The reducer must be a pure function. Calling `SoundEngine.play()` inside the reducer is a side effect and will cause bugs.

**CRITICAL:** Do NOT use arrow functions. The codebase uses `function()` everywhere. Keep it consistent.

**Where:** Add ONE new `React.useEffect` in the `App` component, near the other useEffect blocks (~line 4460 area). Place it AFTER the existing useEffects.

```javascript
// Sound effects — triggered by phase transitions
React.useEffect(function() {
  if (state.phase === 'SNAPPING') {
    SoundEngine.play('snap');
  }
  if (state.phase === 'CONTACT') {
    SoundEngine.play('tackle');
  }
  if (state.phase === 'TOUCHDOWN') {
    SoundEngine.play('whistle');
    setTimeout(function() { SoundEngine.play('crowd'); }, 300);
  }
  if (state.phase === 'RESULT' && state.resultText && state.resultText.includes('SACKED')) {
    SoundEngine.play('sackBuzz');
  }
}, [state.phase, state.resultText]);
```

**Key details:**
- `function()` syntax, NOT arrow functions
- The sound method name is `'sackBuzz'` (NOT `'sack'` — that name doesn't exist in the SoundEngine and would silently fail)
- The mute button already works: `TOGGLE_MUTE` reducer calls `SoundEngine.setMuted()`, and each `SoundEngine.play()` method checks `this.muted`. No mute wiring needed.
- Dependency array: `[state.phase, state.resultText]`

---

## ITEM 7: Coach SVG Portraits on Title Screen

**What:** Replace the emoji icons (🏈 / 🦊) in the coach selection cards with inline SVG character portraits.

**CRITICAL INSTRUCTION:** Convert the EXACT SVG code below into nested `React.createElement` calls. Do NOT invent new coordinates or guess shapes. Every path, circle, ellipse, rect, and line below must be translated 1:1 into React.createElement calls with the exact same attribute values.

**SVG attribute conversion rules:**
- `stroke-width` → `strokeWidth`
- `stroke-linecap` → `strokeLinecap`
- `fill-opacity` → `fillOpacity`
- `viewBox` stays as `viewBox` (already camelCase)
- `xmlns` is NOT needed in React SVG elements — omit it
- All other attributes (`cx`, `cy`, `rx`, `ry`, `d`, `fill`, `stroke`, `opacity`, `x`, `y`, `width`, `height`, `r`, `rx`) stay the same

### Step 1: Define two portrait functions

Place these ABOVE the `TitleScreen` component definition (or just inside it, before `coachCard`). Each returns a `React.createElement('svg', ...)`.

### Jim Portrait — convert this EXACT SVG:

```html
<svg width="100" height="100" viewBox="0 0 140 140">
  <!-- Background circle -->
  <circle cx="70" cy="70" r="68" fill="#1a2e1a" stroke="#22c55e" stroke-width="2"/>

  <!-- THICK neck -->
  <rect x="40" y="105" width="60" height="20" rx="8" fill="#b8906a"/>
  <path d="M44 110 Q70 108 96 110" stroke="#a07a54" stroke-width="0.8" fill="none" opacity="0.5"/>
  <path d="M46 114 Q70 112 94 114" stroke="#a07a54" stroke-width="0.8" fill="none" opacity="0.4"/>

  <!-- BIG ROUND FACE -->
  <ellipse cx="70" cy="72" rx="38" ry="36" fill="#c49a72"/>

  <!-- Double chin -->
  <ellipse cx="70" cy="102" rx="24" ry="10" fill="#b8906a"/>
  <path d="M50 98 Q70 108 90 98" stroke="#a07a54" stroke-width="0.8" fill="none" opacity="0.5"/>
  <path d="M46 94 Q70 100 94 94" stroke="#a8845c" stroke-width="1" fill="none" opacity="0.5"/>

  <!-- Puffy cheeks -->
  <ellipse cx="42" cy="74" rx="10" ry="8" fill="#c9a078" opacity="0.5"/>
  <ellipse cx="98" cy="74" rx="10" ry="8" fill="#c9a078" opacity="0.5"/>

  <!-- Age spots -->
  <circle cx="38" cy="62" r="1.5" fill="#a8845c" opacity="0.4"/>
  <circle cx="96" cy="56" r="1.2" fill="#a8845c" opacity="0.3"/>
  <circle cx="42" cy="82" r="1.3" fill="#a8845c" opacity="0.35"/>
  <circle cx="100" cy="68" r="1" fill="#a8845c" opacity="0.3"/>

  <!-- Small deep-set eyes -->
  <path d="M52 56 Q58 52 64 56" fill="#b8906a" opacity="0.6"/>
  <path d="M76 56 Q82 52 88 56" fill="#b8906a" opacity="0.6"/>
  <ellipse cx="58" cy="58" rx="4" ry="2.5" fill="#ddd8c8"/>
  <ellipse cx="82" cy="58" rx="4" ry="2.5" fill="#ddd8c8"/>
  <circle cx="59" cy="58" r="1.8" fill="#3a5a3a"/>
  <circle cx="83" cy="58" r="1.8" fill="#3a5a3a"/>
  <circle cx="59.5" cy="57.5" r="0.5" fill="#fff"/>
  <circle cx="83.5" cy="57.5" r="0.5" fill="#fff"/>
  <!-- Bags under eyes -->
  <path d="M52 61 Q58 64 64 61" stroke="#a8845c" stroke-width="1.2" fill="none" opacity="0.5"/>
  <path d="M76 61 Q82 64 88 61" stroke="#a8845c" stroke-width="1.2" fill="none" opacity="0.5"/>

  <!-- Bushy gray eyebrows -->
  <path d="M48 52 Q54 47 66 52" stroke="#aaa" stroke-width="3.5" fill="none" stroke-linecap="round"/>
  <path d="M74 52 Q86 47 92 52" stroke="#aaa" stroke-width="3.5" fill="none" stroke-linecap="round"/>
  <path d="M50 51 L48 48" stroke="#bbb" stroke-width="0.8" opacity="0.5"/>
  <path d="M90 51 L92 48" stroke="#bbb" stroke-width="0.8" opacity="0.5"/>
  <path d="M56 50 L55 47" stroke="#bbb" stroke-width="0.6" opacity="0.4"/>

  <!-- Big bulbous nose -->
  <path d="M66 56 Q62 66 58 72 Q64 76 70 76 Q76 76 82 72 Q78 66 74 56" fill="#b08a64"/>
  <ellipse cx="70" cy="74" rx="8" ry="4" fill="#b8906a" opacity="0.5"/>
  <ellipse cx="65" cy="74" rx="2.5" ry="1.5" fill="#906a4a" opacity="0.4"/>
  <ellipse cx="75" cy="74" rx="2.5" ry="1.5" fill="#906a4a" opacity="0.4"/>

  <!-- THE WALRUS MUSTACHE -->
  <path d="M40 78 Q46 68 56 73 Q64 76 70 74 Q76 76 84 73 Q94 68 100 78 Q98 88 86 86 Q76 84 70 85 Q64 84 54 86 Q42 88 40 78Z" fill="#b0b0b0"/>
  <path d="M42 80 Q52 72 62 76 Q66 77 70 76" stroke="#8a8a8a" stroke-width="1.2" fill="none"/>
  <path d="M98 80 Q88 72 78 76 Q74 77 70 76" stroke="#8a8a8a" stroke-width="1.2" fill="none"/>
  <!-- Walrus droop ends -->
  <path d="M40 78 Q36 82 34 86 Q33 90 36 90" stroke="#9a9a9a" stroke-width="2.5" fill="none" stroke-linecap="round"/>
  <path d="M100 78 Q104 82 106 86 Q107 90 104 90" stroke="#9a9a9a" stroke-width="2.5" fill="none" stroke-linecap="round"/>
  <path d="M34 88 L31 86" stroke="#ccc" stroke-width="0.5" opacity="0.4"/>
  <path d="M106 88 L109 86" stroke="#ccc" stroke-width="0.5" opacity="0.4"/>

  <!-- Mouth barely visible -->
  <path d="M58 88 Q70 86 82 88" stroke="#906a4a" stroke-width="1" fill="none" opacity="0.3"/>

  <!-- Forehead wrinkles -->
  <path d="M44 42 Q56 38 70 42 Q84 38 96 42" stroke="#a8845c" stroke-width="0.8" fill="none" opacity="0.5"/>
  <path d="M46 46 Q58 42 72 46 Q84 42 94 46" stroke="#a8845c" stroke-width="0.7" fill="none" opacity="0.4"/>

  <!-- Crow's feet -->
  <path d="M38 56 L44 54" stroke="#a8845c" stroke-width="0.8" opacity="0.5"/>
  <path d="M36 58 L43 57" stroke="#a8845c" stroke-width="0.7" opacity="0.4"/>
  <path d="M102 56 L96 54" stroke="#a8845c" stroke-width="0.8" opacity="0.5"/>
  <path d="M104 58 L97 57" stroke="#a8845c" stroke-width="0.7" opacity="0.4"/>

  <!-- Headset band -->
  <path d="M24 52 Q26 22 70 18 Q114 22 116 52" stroke="#555" stroke-width="5" fill="none" stroke-linecap="round"/>
  <!-- Earpieces -->
  <rect x="18" y="48" width="13" height="20" rx="4" fill="#3a3a3a" stroke="#555" stroke-width="1"/>
  <rect x="20" y="51" width="9" height="12" rx="2" fill="#2a2a2a"/>
  <rect x="109" y="48" width="13" height="20" rx="4" fill="#3a3a3a" stroke="#555" stroke-width="1"/>
  <rect x="111" y="51" width="9" height="12" rx="2" fill="#2a2a2a"/>
  <!-- Mic boom -->
  <path d="M22 62 Q14 66 12 74 Q10 82 14 88 Q20 94 26 90" stroke="#555" stroke-width="3" fill="none" stroke-linecap="round"/>
  <ellipse cx="26" cy="90" rx="5" ry="3.5" fill="#3a3a3a" stroke="#555" stroke-width="1"/>
  <circle cx="25" cy="90" r="2" fill="#2a2a2a"/>

  <!-- Gray hair at temples -->
  <path d="M34 46 Q28 40 30 34 Q32 30 36 32" fill="#bbb" opacity="0.6"/>
  <path d="M32 50 Q26 46 28 40" stroke="#aaa" stroke-width="1.5" fill="none" opacity="0.4"/>
  <path d="M106 46 Q112 40 110 34 Q108 30 104 32" fill="#bbb" opacity="0.6"/>
  <path d="M108 50 Q114 46 112 40" stroke="#aaa" stroke-width="1.5" fill="none" opacity="0.4"/>

  <!-- Cap -->
  <path d="M32 42 Q34 18 70 14 Q106 18 108 42 Q70 36 32 42Z" fill="#2a5028"/>
  <path d="M28 42 Q70 48 112 42 Q110 34 70 32 Q30 34 28 42Z" fill="#1a3818"/>
  <path d="M44 38 Q56 36 68 38 Q80 36 92 38" stroke="#3a6030" stroke-width="1.5" fill="none" opacity="0.3"/>
</svg>
```

### Joey Portrait — convert this EXACT SVG:

```html
<svg width="100" height="100" viewBox="0 0 140 140">
  <!-- Background circle -->
  <circle cx="70" cy="70" r="68" fill="#1a1a2e" stroke="#4a9eff" stroke-width="2"/>

  <!-- Neck -->
  <rect x="56" y="100" width="28" height="14" rx="4" fill="#d4a574"/>

  <!-- Face -->
  <ellipse cx="70" cy="70" rx="28" ry="32" fill="#d4a574"/>

  <!-- Visor/Sunglasses -->
  <path d="M40 54 Q70 48 100 54 Q102 62 100 64 Q70 58 40 64 Q38 62 40 54Z" fill="#1a1a2e" stroke="#333" stroke-width="1"/>
  <!-- Glare -->
  <path d="M48 57 Q65 52 80 57" stroke="rgba(74,158,255,0.4)" stroke-width="1.5" fill="none"/>
  <!-- Second glare -->
  <path d="M52 60 Q65 56 78 60" stroke="rgba(74,158,255,0.15)" stroke-width="1" fill="none"/>

  <!-- Cocky smirk -->
  <path d="M50 82 Q58 86 70 88 Q82 86 88 80" stroke="#fff" stroke-width="2.2" fill="none"/>
  <!-- Teeth -->
  <path d="M54 83 Q62 88 74 88 Q82 86 86 81" fill="#fff" opacity="0.85"/>
  <line x1="60" y1="84" x2="60" y2="88" stroke="#d4a574" stroke-width="0.5"/>
  <line x1="65" y1="85" x2="65" y2="88" stroke="#d4a574" stroke-width="0.5"/>
  <line x1="70" y1="86" x2="70" y2="88" stroke="#d4a574" stroke-width="0.5"/>
  <line x1="75" y1="85" x2="75" y2="88" stroke="#d4a574" stroke-width="0.5"/>
  <line x1="80" y1="84" x2="80" y2="86" stroke="#d4a574" stroke-width="0.5"/>

  <!-- Dimple -->
  <path d="M88 78 Q90 80 89 82" stroke="#b8906a" stroke-width="0.8" fill="none" opacity="0.6"/>

  <!-- Nose -->
  <path d="M67 60 Q65 68 63 72 Q67 74 70 74 Q73 74 77 72 Q75 68 73 60" fill="#c49560"/>

  <!-- Sharp jawline -->
  <path d="M46 80 Q52 98 70 100 Q88 98 94 80" stroke="#c49560" stroke-width="1" fill="none"/>
  <!-- Chin dimple -->
  <ellipse cx="70" cy="96" rx="2" ry="1.5" fill="#c09058" opacity="0.4"/>

  <!-- Stubble -->
  <circle cx="52" cy="86" r="0.6" fill="#8a6a4a" opacity="0.25"/>
  <circle cx="54" cy="88" r="0.5" fill="#8a6a4a" opacity="0.25"/>
  <circle cx="56" cy="90" r="0.6" fill="#8a6a4a" opacity="0.2"/>
  <circle cx="50" cy="90" r="0.5" fill="#8a6a4a" opacity="0.2"/>
  <circle cx="88" cy="84" r="0.6" fill="#8a6a4a" opacity="0.25"/>
  <circle cx="86" cy="86" r="0.5" fill="#8a6a4a" opacity="0.25"/>
  <circle cx="84" cy="88" r="0.6" fill="#8a6a4a" opacity="0.2"/>
  <circle cx="90" cy="88" r="0.5" fill="#8a6a4a" opacity="0.2"/>
  <circle cx="62" cy="94" r="0.5" fill="#8a6a4a" opacity="0.2"/>
  <circle cx="66" cy="96" r="0.5" fill="#8a6a4a" opacity="0.15"/>
  <circle cx="74" cy="96" r="0.5" fill="#8a6a4a" opacity="0.15"/>
  <circle cx="78" cy="94" r="0.5" fill="#8a6a4a" opacity="0.2"/>

  <!-- Hair poking out -->
  <path d="M40 48 Q36 42 33 44 Q30 48 36 50" fill="#3a2a1a"/>
  <path d="M38 46 Q34 40 32 42" stroke="#3a2a1a" stroke-width="2" fill="none" stroke-linecap="round"/>
  <path d="M100 48 Q104 42 107 44 Q110 48 104 50" fill="#3a2a1a"/>
  <path d="M102 46 Q106 40 108 42" stroke="#3a2a1a" stroke-width="2" fill="none" stroke-linecap="round"/>

  <!-- Cap -->
  <path d="M34 48 Q38 24 70 22 Q102 24 106 48 Q70 42 34 48Z" fill="#1a3060"/>
  <!-- Flat brim -->
  <path d="M28 48 Q70 54 112 46 Q108 38 70 38 Q32 38 28 48Z" fill="#122448"/>
  <!-- Star logo -->
  <path d="M70 30 L72 36 L78 36 L73 40 L75 46 L70 42 L65 46 L67 40 L62 36 L68 36Z" fill="#4a9eff" opacity="0.7"/>

  <!-- Earpiece -->
  <circle cx="100" cy="62" r="3.5" fill="#222" stroke="#444" stroke-width="0.8"/>
  <circle cx="100" cy="62" r="1.5" fill="#333"/>
  <!-- Cord -->
  <path d="M100 65 Q98 72 96 78 Q94 84 90 88" stroke="#333" stroke-width="1" fill="none"/>

  <!-- Whistle on lanyard -->
  <path d="M70 100 Q68 106 62 108" stroke="#888" stroke-width="1.5" fill="none"/>
  <ellipse cx="60" cy="109" rx="4" ry="3" fill="#bbb" stroke="#999" stroke-width="0.5"/>
  <!-- Whistle hole -->
  <circle cx="58" cy="109" r="1" fill="#888"/>

  <!-- Polo collar -->
  <path d="M50 98 Q56 102 62 100 L64 106 Q58 108 50 104Z" fill="#1a3060"/>
  <path d="M90 98 Q84 102 78 100 L76 106 Q82 108 90 104Z" fill="#1a3060"/>
  <!-- Popped collar -->
  <path d="M90 98 Q92 96 91 94" stroke="#1a3060" stroke-width="2" fill="none"/>
</svg>
```

### Step 2: Wire into TitleScreen

In the `coachCard` function (~line 5168), replace the emoji div:

```javascript
// REPLACE THIS:
React.createElement('div', { style: { fontSize: '28px', marginBottom: '6px' } },
  name === 'joey' ? '\u{1F3C8}' : '\u{1F98A}'
),

// WITH THIS:
React.createElement('div', { style: { marginBottom: '6px' } },
  name === 'joey' ? joeyPortrait() : jimPortrait()
),
```

Where `jimPortrait` and `joeyPortrait` are the functions defined in Step 1.

### Display size

Both SVGs render at `width="100" height="100"` with `viewBox="0 0 140 140"`. The viewBox handles the internal coordinate scaling. Do NOT change the viewBox coordinates.

---

## VERSION BUMP

Update the version string on the title screen.

**Where:** Line ~5204

```javascript
// CHANGE:
'ALPHA v2.1.0'

// TO:
'ALPHA v2.1.1'
```

---

## NOTES.md UPDATE

Append to Notes.md:

```
## Alpha 2.1.1 — Hotfix (2026-02-22)
- Follow camera zoom adjusted from /40 to /55 (less jarring, more field visible)
- WR/TE carriers get "Run Upfield" move (HitTheHole type) — Sprint fatigue no longer a dead end
- Defense tokens (Base 4-3) now visible on field during MENU/PLAY_SELECT
- Jim emoji changed to 🦊 (fox) — overridden by SVG portrait
- Game Over "Play Again" uses RESTART_GAME dispatch instead of page reload
- Sound engine activated: snap, tackle, whistle+crowd on TD, sackBuzz
- Coach SVG portraits on title screen (Jim: fat, walrus mustache, headset; Joey: sunglasses, smirk, earpiece)
```

---

## DONE SUMMARY

When complete, print a plain-English summary:
- What changed (one line per item)
- Where files are saved
- Anything the user still needs to do (e.g., copy to thumb drive, test specific scenarios)

---

## CHECKLIST

- [ ] Backup created (`tacfoot-rewrite-backup-2.1.0.html`)
- [ ] Camera zoom: `/40` → `/55`
- [ ] WR/TE Run Upfield added to getRunnerMoveLabels
- [ ] Fallback defense tokens on MENU/PLAY_SELECT
- [ ] Jim emoji: 🧠 → 🦊 (overridden by SVG but kept as fallback)
- [ ] Game Over: reload → RESTART_GAME dispatch
- [ ] Sound engine: ONE useEffect, four phase checks, `function()` syntax, `sackBuzz` not `sack`
- [ ] Jim SVG portrait: exact coordinates from v4, converted to React.createElement with camelCase attrs
- [ ] Joey SVG portrait: exact coordinates from v4, converted to React.createElement with camelCase attrs
- [ ] Portrait functions wired into coachCard replacing emoji div
- [ ] Version bumped to 2.1.1
- [ ] Notes.md updated
- [ ] All changes use `React.createElement`, no JSX
- [ ] All functions use `function()`, no arrow functions
- [ ] No new files created — everything in index.html
