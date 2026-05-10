# LazyFly.me — Codebase Reference

This document maps the full game structure for future edits. The game is a Construct 3 exported HTML5 project hosted on GitHub Pages at https://propel-paradise.github.io/lazyfly/.

---

## File Structure

| File | Size | Purpose |
|------|------|---------|
| `index.html` | tiny | Loads all scripts and styles; the only file safe to add custom JS/CSS |
| `style.css` | tiny | `body` background `#323232`, canvas `touch-action: none` |
| `data.js` | 182KB | JSON: all 88 game objects, 29 layouts, event sheet logic |
| `c2runtime.js` | 347KB | Minified Construct 3 engine; do not modify |
| `start.js` | tiny | Calls `window.cr_createRuntime({exportType:"html5"})` |
| `box2d.asm.js` | large | Physics engine |
| `opus.js` | large | Audio codec |
| `register-sw.js` | tiny | PWA service worker registration |
| `appmanifest.json` | tiny | PWA config: fullscreen, landscape |
| `images/` | - | Sprite sheets |
| `media/` | - | Audio files |
| `icons/` | - | App icons |

---

## Runtime Globals (from c2runtime.js)

| Global | Type | Purpose |
|--------|------|---------|
| `window.g_a` | Object | Root C3 namespace; `g_a.g_b` = plugin classes |
| `window.c3runtime` | Runtime | Live runtime instance; set on game start |
| `window.c3canvas` | Canvas | The game's `<canvas>` element |
| `window.cr_createRuntime(opts)` | Function | Factory; called by start.js |
| `window.cr_sizeCanvas(w, h)` | Function | Resize canvas |
| `window.cr_setSuspended(bool)` | Function | Pause/resume game |
| `window.cr_getSnapshot()` | Function | Canvas snapshot |

### Accessing Game Objects at Runtime

```javascript
const rt = window.c3runtime;

// Get an object type by name
const type = rt.types['Fly'];  // keys are the exact object names from data.js

// Get instances of that type
const instances = type.g_jQ;  // array of live instances

// Read a Text object's text content (e.g. SoundText)
const text = instances[0].text;  // Text plugin stores displayed string in .text

// All layout list
const layouts = rt.g_gD;  // array of layout objects
```

### Reading Global Variables at Runtime

Global variables (like `DeadFlies`, `FlyAlive`, `SoundVar`) are stored in `rt.g_hZ`, keyed by their SID (the large integer in data.js). The value lives in `.data`.

```javascript
// Find a global variable by name (search by name, not SID, for robustness)
function getGlobalVar(name) {
  const rt = window.c3runtime;
  if (!rt || !rt.g_hZ) return null;
  for (const key of Object.keys(rt.g_hZ)) {
    const v = rt.g_hZ[key];
    if (v && v.name === name) return v;  // v.data is the current value
  }
  return null;
}

// Example:
const deadFlies = getGlobalVar('DeadFlies');
console.log(deadFlies.data);  // current death count (number)
```

**Key global variable SIDs (from data.js):**
- `DeadFlies` → SID `871358709387176` — death counter, numeric
- `FlyAlive` → string: `"Alive"`, `"Dead"`, `"Next"`, `"Win"`
- `SoundVar` → string: `"On"`, `"Off"`

**Note:** `DeadFliesNum` (a Text object in data.js) has **zero instances** — it is defined but never placed in any layout and never used. Do not rely on it. Use `rt.g_hZ` to read `DeadFlies` directly.

### Plugin Classes

```javascript
window.g_a.g_b.Text         // Text plugin class
window.g_a.g_b.Audio        // Audio plugin class
window.g_a.g_b.Keyboard     // Keyboard plugin class
window.g_a.g_b.Touch        // Touch plugin class
```

---

## data.js Structure

Single-line minified JSON. Do NOT pretty-print and re-minify — the engine reads it verbatim.

### Top-level keys
- `"project"` → `[name, null, [plugin_flags], [object_types], [layouts], [event_sheets]]`

### Object Type Format
```
[name, plugin_type_id, is_global, families, initial_count, ..., sid, ..., uid]
```

Plugin type IDs:
- `0` = Sprite
- `4` = Keyboard
- `5` = Mouse
- `9` = Text
- `11` = Audio
- `12` = Browser (can execute JavaScript)

### Event Action Format (in event sheets)
```
[action_id, object_uid_or_system, invert, sid, async, null, [[param_type, param_value], ...]]
```

Common action types in events:
- `11` → "Add to global variable": `[[11,"VarName"],[7,[1,amount]]]`
- `7` → expression: `[7,[1, numeric_literal]]` or `[7,[2, string_literal]]`
- `19` → "Set text of object"
- `23` → "Compare global variable" (condition)

---

## Game Object Inventory (all 88 objects)

### Player & Core
| Name | Plugin | SID | Notes |
|------|--------|-----|-------|
| `Fly` | Sprite | — | Player; Physics, ScrollTo, DestroyOutsideLayout behaviors |
| `FlySpawner` | Sprite | — | Invisible spawn point; Fly created here on retry |
| `FlyTrail` | Sprite | — | Trail effect behind Fly |

### Obstacles
| Name | Notes |
|------|-------|
| `Walls` | Static physics walls |
| `GravityBox` | Gravity-flipping zone |
| `Vent` | Air current; pushes Fly |
| `Rise` | Rising obstacle |
| `Bed` | Level-clear objective |
| `FinalBed` | Final level win objective |
| `Wind` | Horizontal wind force |
| `Touch` | Touch-sensitive trigger |
| `FanDetect` | Fan detection zone |
| `Cover` | Overlay cover sprite |

### Death Effects
| Name | Plugin | Notes |
|------|--------|-------|
| `DeadFly` | Sprite | Spawned at death point; Physics + DestroyOutsideLayout; image: `shared-0-sheet3.png` |
| `Splat` | Sprite | 2-frame splat animation; image: `shared-0-sheet2.png` |
| `BloodSpray` | Sprite | Particle effect on death |
| `BloodSplat` | Sprite | Blood stain on wall |
| `FlyGuts` | Sprite | Guts particle |
| `BedBurst` | Sprite | Bed explode effect on level clear |

### Death Counter Objects (key for death tracking)
| Name | Plugin | Notes |
|------|--------|-------|
| `DeadFliesNum` | **Text** | Displays current session death count. `inst.text` = "0", "1", etc. |
| `DeadFliesText` | Text | Label text displayed alongside DeadFliesNum |

### UI Sprites
| Name | Notes |
|------|-------|
| `Retry` | Button shown on death; click to retry |
| `NextLev` | Button shown on level clear; click to advance |
| `PlayAgain` | Button on win/end screen |
| `Tutorial` | Tutorial sprite |
| `Arrow` | Arrow indicator |
| `LazyFlyTitle` | Title screen logo |
| `meTitle`, `bedTitle`, `ventsTitle` | Level-select title elements |
| `Audio`, `SoundOnOff` | Sound toggle button |

### Level Text Objects
`Level1Text` through `Level26Text` — one per level, shown in level select or title.

### Input & System
| Name | Plugin |
|------|--------|
| `Keyboard` | Keyboard input |
| `Mouse` | Mouse input |
| `Touch` | Touch input |
| `Browser` | Browser plugin (can execute JS via "Execute JavaScript" action) |
| `Zparticles` | Particle system |

---

## Layouts (29 total)

All use the `MainEvent` event sheet for game logic.

| Index | Name | Dimensions |
|-------|------|------------|
| 0 | ClickToPlay | 854×480 |
| 1 | Tutorial | 854×480 |
| 2–27 | Level1–Level26 | 854–1540 wide, 480–2160 tall |
| 28 | End | 854×480 |

Level dimensions vary widely (some levels are very tall, requiring vertical scrolling).

---

## Death Mechanic (Event Sheet Flow)

```
Fly collides with Walls/GravityBox/obstacle
  → FlyAlive = "Dead"
  → Create Splat at collision point (2-frame anim)
  → Create BloodSpray, BloodSplat, FlyGuts effects
  → Create DeadFly sprite (physics, falls off screen)
  → Add 1 to global variable DeadFlies
  → Set DeadFliesNum.text to string(DeadFlies)   ← this is how DeadFliesNum updates
  → Stop FanNoise, FanOverNoise, FlyBuzz sounds
  → Show Retry button at HUD position (427, 240)
  → 0.5s fade effect

User clicks Retry button (Event 19):
  → FlyAlive = "Alive"
  → Destroy old Fly instance
  → Create new Fly at FlySpawner position
  → Hide Retry button
  → Resume sounds

Fly reaches Bed (Event 15):
  → FlyAlive = "Next"
  → Show NextLev button
  → Play 'nextlevel' sound

User clicks NextLev (Event 20):
  → Go to next layout
```

### State Variable: FlyAlive
Values: `"Alive"` | `"Dead"` | `"Next"` | `"Win"`

### Counter Variable: DeadFlies
- C3 global variable (event sheet scope)
- Persists across layout changes within a session
- Resets to 0 on page reload
- **Is reflected in `DeadFliesNum` text object**

---

## How to Read Death Count at Runtime

```javascript
// Read the DeadFlies global variable directly from the runtime
var deadFliesVar = null;

function getSessionDeaths() {
  var rt = window.c3runtime;
  if (!rt || !rt.g_hZ) return null;

  if (!deadFliesVar) {
    var keys = Object.keys(rt.g_hZ);
    for (var i = 0; i < keys.length; i++) {
      var v = rt.g_hZ[keys[i]];
      if (v && v.name === 'DeadFlies') { deadFliesVar = v; break; }
    }
  }

  return deadFliesVar && typeof deadFliesVar.data === 'number' ? deadFliesVar.data : null;
}
```

**Why `rt.g_hZ`:** Global variables are stored in `window.c3runtime.g_hZ`, keyed by their SID. Each entry is an object `{ name, data, g_jC, g_oN, ... }` where `.data` is the current value.

**`DeadFliesNum` is unused:** Despite being defined as object type 24 in data.js, it has 0 instances and is never placed in any layout. All death tracking goes through the `DeadFlies` global variable in `rt.g_hZ`.

---

## Sprite Sheets

| File | Contents |
|------|----------|
| `images/shared-0-sheet0.png` | General background/tile sprites |
| `images/shared-0-sheet1.png` | General sprites |
| `images/shared-0-sheet2.png` | Fly animation frames, FlySpawner, Splat |
| `images/shared-0-sheet3.png` | Walls, GravityBox, DeadFly sprite |
| `images/shared-0-sheet4.png` | UI buttons (Retry, NextLev, etc.) |

---

## Adding a New Level

1. Open `data.js`
2. Find the layouts array (search for `"Level26"`)
3. Copy an existing level's JSON block and increment:
   - Layout name: `"Level27"`
   - Layout SID (must be unique — use a large random integer)
   - Layout UID (sequential integer, one past the last)
   - Set dimensions for the new level
4. Add object instances (obstacles, Bed, FlySpawner, etc.) in the layout's objects array
5. Add `"Level27Text"` object type to the project if showing level text
6. All game logic runs automatically via `MainEvent` — no event sheet changes needed for basic levels

**Important:** The JSON is a single line. Use a JSON validator after editing.

---

## Design System

| Property | Value |
|----------|-------|
| Background color | `#323232` |
| Text color | white |
| Font (game canvas) | Set per-object in data.js |
| Overlay font | monospace (for custom HTML overlays) |
| Game resolution | 854×480 base |

---

## Safe Modification Points

| What to modify | File | Risk |
|----------------|------|------|
| Add HTML overlays / custom JS | `index.html` | Low — additive only |
| Change PWA display/orientation | `appmanifest.json` | Low |
| Game objects, layouts, events | `data.js` | Medium — JSON must stay valid |
| Engine behavior | `c2runtime.js` | High — do not touch |
| Canvas styles | `style.css` | Low |

---

## v1.1 Changes (branch: v1.1)

### Attempt Counter (`index.html`)

Added a GD-style attempt counter as a fixed HTML overlay. Key details:

- **Element:** `<div id="lf-attempt">` — `position: fixed`, `z-index: 9999`, `pointer-events: none`
- **Positioning:** Reads `Level1Text` type (uid 35) world-space position from `rt.types['Level1Text'].g_jQ[0]`, converts to screen coords via `worldToScreen()`, places div immediately below the text (2px gap)
- **Font:** Arial, dynamically scaled (`12 * pxPerWorldUnit`), color `rgb(0,0,0)` — matches in-game level text exactly
- **Death tracking:** Reads `DeadFlies` global variable from `rt.g_hZ` (searched by `.name`), increments `attempts` on each death delta
- **Reset:** Resets to "Attempt 1" on layout change (`rt.g_h$.g_jC` SID changes)
- **No persistence:** Counter is session-only; resets on page close/reload

### World-to-Screen Formula

```
lx = (worldX - scrollX) * scale + logW / 2
ly = (worldY - scrollY) * scale + logH / 2
screenX = rect.left + lx * (rect.width  / logW)
screenY = rect.top  + ly * (rect.height / logH)
```

Where `scrollX/scrollY` = `rt.g_h$.scrollX/scrollY`, `scale` = `rt.g_h$.scale`, `logW/logH` = `rt.g_iS/rt.g_iT` (854×480).
