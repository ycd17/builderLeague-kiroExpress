# Flappy Kiro — Game Mechanics Reference

## Physics Constants

These values are defined once at the top of the `<script>` block and must not be redefined elsewhere.

| Constant        | Value | Unit         | Description                                      |
|-----------------|-------|--------------|--------------------------------------------------|
| `GRAVITY`       | 0.5   | px/frame²    | Downward acceleration applied every frame        |
| `FLAP_IMPULSE`  | -9    | px/frame     | Upward velocity set on flap (negative = up)      |
| `MAX_FALL_SPEED`| 12    | px/frame     | Terminal velocity cap (downward)                 |
| `PIPE_SPEED`    | 3     | px/frame     | Leftward scroll speed for all pipes              |
| `PIPE_WIDTH`    | 60    | px           | Width of each pipe segment                       |
| `GAP_HEIGHT`    | 150   | px           | Vertical opening between top and bottom pipe     |
| `SPAWN_INTERVAL`| 90    | frames       | Frames between consecutive pipe spawns           |
| `BG_SPEED`      | 2     | px/frame     | Background scroll speed                          |
| `CANVAS_WIDTH`  | 480   | px           | Canvas element width                             |
| `CANVAS_HEIGHT` | 640   | px           | Canvas element height                            |

### Pipe Gap Bounds

```js
const MIN_GAP_Y = 60;   // minimum y of gap top (px from canvas top)
const MAX_GAP_Y = 300;  // maximum y of gap top (px from canvas top)
```

These bounds ensure the gap is always fully visible and reachable by the player.

---

## Y-Axis Convention

The canvas Y-axis increases **downward**:

- `vy < 0` → character is moving **upward** → show **ascending sprite** (`assets/inv1.jpg`)
- `vy >= 0` → character is moving **downward** → show **descending sprite** (`assets/inv2.jpg`)

This is the opposite of typical math coordinates. All physics logic must follow this convention.

---

## Movement Algorithms

### Gravity Application (every frame, Playing State only)

```js
// PhysicsEngine.applyGravity(character)
character.vy = Math.min(character.vy + GRAVITY, MAX_FALL_SPEED);
```

- Increment `vy` by `GRAVITY` first, then clamp — never clamp before adding
- Clamping uses `Math.min` because downward velocity is positive and must not exceed `MAX_FALL_SPEED`

### Position Update (every frame, after gravity)

```js
// PhysicsEngine.update(character)
character.y += character.vy;
```

- Always apply velocity to position **after** clamping, not before
- `character.x` is fixed at `80` — never modified by physics

### Flap Impulse (on Space / click, Playing State only)

```js
// PhysicsEngine.applyFlap(character)
character.vy = FLAP_IMPULSE; // -9, sets directly — not additive
```

- Flap **sets** velocity to `FLAP_IMPULSE`, it does not add to current velocity
- This ensures consistent jump height regardless of current fall speed

### Pipe Scrolling (every frame, Playing State only)

```js
// PipeSpawner.scroll(speed)
for (const pipe of this.pipes) {
  pipe.x -= speed; // move left by PIPE_SPEED each frame
}
```

### Background Scrolling (every frame, Playing State and Game Over Screen)

```js
// BackgroundScroller.update()
this.offset = (this.offset + this.speed) % this.tileWidth;
```

- `tileWidth = 400` — the background tile repeats every 400px
- Modulo wrapping ensures seamless looping with no visible seam

### Pipe Spawn Timer

```js
// PipeSpawner.update(canvasWidth)
this.spawnTimer--;
if (this.spawnTimer <= 0) {
  this.spawn(canvasWidth);
  this.spawnTimer = SPAWN_INTERVAL;
}
this.scroll(PIPE_SPEED);
this.cleanup();
```

### Pipe Cleanup

```js
// PipeSpawner.cleanup()
this.pipes = this.pipes.filter(p => p.x + p.width >= 0);
```

Remove pipes only after their right edge has fully exited the left side of the canvas.

---

## Collision Detection Patterns

### AABB Overlap (strict — touching edges do not collide)

```js
// CollisionDetector.rectOverlap(a, b)
// a, b: { x, y, width, height }
return a.x < b.x + b.width  &&
       a.x + a.width  > b.x &&
       a.y < b.y + b.height &&
       a.y + a.height > b.y;
```

- Uses strict `<` / `>` — rectangles that only share an edge are **not** colliding
- Must be symmetric: `rectOverlap(a, b) === rectOverlap(b, a)` for all inputs

### Pipe Segment Bounding Boxes

Each `PipePair` has two implicit segments derived from its fields:

```js
// Top pipe segment
const topPipe = {
  x: pipe.x,
  y: 0,
  width: pipe.width,
  height: pipe.gapY,           // from canvas top down to gap opening
};

// Bottom pipe segment
const bottomPipe = {
  x: pipe.x,
  y: pipe.gapY + pipe.gapHeight,
  width: pipe.width,
  height: CANVAS_HEIGHT - (pipe.gapY + pipe.gapHeight),
};
```

### Full Collision Check

```js
// CollisionDetector.check(character, pipes, canvasHeight)
// Returns true if any collision is detected

// 1. Canvas boundary — top
if (character.y < 0) return true;

// 2. Canvas boundary — bottom
if (character.y + character.height > canvasHeight) return true;

// 3. Pipe segments
for (const pipe of pipes) {
  const topPipe    = { x: pipe.x, y: 0,                          width: pipe.width, height: pipe.gapY };
  const bottomPipe = { x: pipe.x, y: pipe.gapY + pipe.gapHeight, width: pipe.width, height: canvasHeight - (pipe.gapY + pipe.gapHeight) };
  if (rectOverlap(character, topPipe) || rectOverlap(character, bottomPipe)) return true;
}

return false;
```

- Boundary checks come first (cheaper than iterating pipes)
- Both top and bottom segments of every pipe are checked each frame

---

## Scoring Logic

```js
// ScoreManager.update(character, pipes)
for (const pipe of pipes) {
  if (!pipe.scored && character.x > pipe.x + pipe.width) {
    this.score++;
    pipe.scored = true;
  }
}
```

- A pipe is scored when the **left edge of the character** passes the **right edge of the pipe**
- `pipe.scored` flag prevents double-counting on subsequent frames
- Score is reset to `0` on every transition to Playing State

---

## State Machine Transitions

```
START  ──(Space / Click)──►  PLAYING  ──(collision)──►  GAMEOVER
                               ▲                             │
                               └────────(Space / Click)──────┘
```

### On transition to PLAYING (from either START or GAMEOVER):

```js
character.y  = 200;
character.vy = 0;
PipeSpawner.pipes      = [];
PipeSpawner.spawnTimer = SPAWN_INTERVAL;
ScoreManager.reset();   // score = 0
```

### Physics guard — only apply during PLAYING:

```js
if (gameState !== STATE.PLAYING) return; // skip gravity, pipes, collision
```

---

## Sprite Selection Rule

| Condition     | Sprite shown         | Asset            |
|---------------|----------------------|------------------|
| `vy < 0`      | Ascending (flapping) | `assets/inv1.jpg`|
| `vy >= 0`     | Descending (falling) | `assets/inv2.jpg`|

Selection happens inside `Renderer.drawCharacter` on every frame — no caching needed.

---

## Correctness Properties Summary

These invariants must hold at all times and are verified by property-based tests:

1. After `applyGravity`, `vy` is strictly greater than before (if below cap)
2. After `applyFlap`, `vy === FLAP_IMPULSE` regardless of prior state
3. After any number of `applyGravity` calls, `vy <= MAX_FALL_SPEED`
4. Every spawned pipe satisfies `MIN_GAP_Y <= gapY <= MAX_GAP_Y`
5. Score equals the count of pipes whose right edge the character has passed; never decreases
6. `Renderer` selects ascending sprite iff `vy < 0`
7. `rectOverlap(a, b) === rectOverlap(b, a)` for all rectangle pairs
8. `ScoreManager.reset()` always sets `score` to exactly `0`
