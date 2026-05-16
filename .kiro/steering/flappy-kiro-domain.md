# Flappy Kiro — Domain Reference

## Game State Management

### State Enum

```js
const STATE = {
  START:    "start",
  PLAYING:  "playing",
  GAMEOVER: "gameover",
};

let gameState = STATE.START;
```

- `gameState` is a single module-level string — never an object, never nested
- All state checks use strict equality: `gameState === STATE.PLAYING`
- No other module may write to `gameState` directly; transitions happen only inside the `onAction` callback and the collision branch of the game loop

### Valid Transitions

| From       | Event              | To         |
|------------|--------------------|------------|
| `START`    | Space / Click      | `PLAYING`  |
| `PLAYING`  | Collision detected | `GAMEOVER` |
| `GAMEOVER` | Space / Click      | `PLAYING`  |

There is no `PLAYING → START` transition. The only way back to `START` is a full page reload.

### Transition Handler (onAction)

```js
function onAction() {
  if (gameState === STATE.START || gameState === STATE.GAMEOVER) {
    // Reset all mutable state
    character.y  = 200;
    character.vy = 0;
    PipeSpawner.pipes      = [];
    PipeSpawner.spawnTimer = SPAWN_INTERVAL;
    ScoreManager.reset();
    gameState = STATE.PLAYING;
  } else if (gameState === STATE.PLAYING) {
    PhysicsEngine.applyFlap(character);
    AudioManager.playJump();
  }
}
```

- A single `onAction` function handles all input — the state check inside it determines the correct behavior
- Reset happens **before** setting `gameState = STATE.PLAYING` so the first frame renders a clean slate
- `AudioManager.playJump()` is called on every flap during Playing State, including the very first one

### Game Loop State Guard

```js
function gameLoop() {
  if (gameState === STATE.PLAYING) {
    PhysicsEngine.applyGravity(character);
    PhysicsEngine.update(character);
    BackgroundScroller.update();
    PipeSpawner.update(CANVAS_WIDTH);
    ScoreManager.update(character, PipeSpawner.pipes);

    if (CollisionDetector.check(character, PipeSpawner.pipes, CANVAS_HEIGHT)) {
      AudioManager.playGameOver();
      gameState = STATE.GAMEOVER;
    }
  } else if (gameState === STATE.GAMEOVER) {
    // Background still scrolls on game over screen
    BackgroundScroller.update();
  }
  // Render runs every frame regardless of state
  render();
  requestAnimationFrame(gameLoop);
}
```

- Physics and pipe updates are **strictly gated** to `STATE.PLAYING`
- Background scrolling continues on the Game Over screen for visual continuity
- `render()` is called unconditionally every frame

---

## Score Management

### ScoreManager Module

```js
const ScoreManager = {
  score: 0,

  update(character, pipes) {
    for (const pipe of pipes) {
      if (!pipe.scored && character.x > pipe.x + pipe.width) {
        this.score++;
        pipe.scored = true;
      }
    }
  },

  reset() {
    this.score = 0;
  },
};
```

### Scoring Rules

- A pipe is scored when `character.x > pipe.x + pipe.width` — the character's **left edge** has cleared the pipe's **right edge**
- `pipe.scored = true` is set immediately to prevent double-counting on subsequent frames
- Score is always a non-negative integer; it never decreases within a session
- `reset()` sets `score` to exactly `0` — not `undefined`, not `null`

### Score Display

- Live score is rendered centered at the top of the canvas (`y = 16`) during `STATE.PLAYING` only
- Final score is shown on the Game Over screen as `Score: N`
- There is no high score persistence in the base implementation (see optional section below)

### Optional: Session High Score

If a best-score display is desired, track it as a separate module-level variable — never inside `ScoreManager`:

```js
let bestScore = 0;

// In the collision branch, before transitioning to GAMEOVER:
if (ScoreManager.score > bestScore) {
  bestScore = ScoreManager.score;
}
```

- `bestScore` resets to `0` on page reload (no `localStorage` in the base spec)
- Display it on the Game Over screen below the final score: `Best: N`

---

## Difficulty Progression

### Base Spec: No Difficulty Scaling

The base implementation uses fixed constants for all difficulty parameters. Pipe speed, gap height, and spawn interval never change. This keeps the implementation simple and the property-based tests deterministic.

### Optional: Time-Based Difficulty Scaling

If progressive difficulty is added, implement it as a read-only computed value derived from `ScoreManager.score` — never by mutating the base constants.

```js
// Difficulty multiplier — increases every 10 pipes passed
function getDifficultyLevel() {
  return Math.floor(ScoreManager.score / 10);
}

// Effective pipe speed — capped to prevent unplayability
function getEffectivePipeSpeed() {
  return Math.min(PIPE_SPEED + getDifficultyLevel() * 0.4, 8);
}

// Effective gap height — shrinks as difficulty increases, minimum 90px
function getEffectiveGapHeight() {
  return Math.max(GAP_HEIGHT - getDifficultyLevel() * 8, 90);
}
```

- Base constants (`PIPE_SPEED`, `GAP_HEIGHT`) remain unchanged — effective values are computed on the fly
- Cap effective values to prevent the game from becoming physically impossible
- Pass effective values into `PipeSpawner.update` and `PipeSpawner.spawn` as arguments, not via globals

### Optional: Spawn Interval Scaling

```js
function getEffectiveSpawnInterval() {
  return Math.max(SPAWN_INTERVAL - getDifficultyLevel() * 5, 50);
}
```

- Minimum spawn interval of 50 frames prevents pipes from overlapping
- Reset difficulty implicitly resets when `ScoreManager.reset()` is called, since level is derived from score

---

## Reset Checklist

Every transition to `STATE.PLAYING` must reset **all** of the following:

| Item                        | Reset value          |
|-----------------------------|----------------------|
| `character.y`               | `200`                |
| `character.vy`              | `0`                  |
| `PipeSpawner.pipes`         | `[]`                 |
| `PipeSpawner.spawnTimer`    | `SPAWN_INTERVAL`     |
| `ScoreManager.score`        | `0`                  |
| `flashAlpha` (if used)      | `0`                  |
| `particles` (if used)       | `[]`                 |

`character.x` is **never** reset — it is always fixed at `80`.

---

## Pipe Lifecycle

```
spawn (x = CANVAS_WIDTH)
  → scroll left each frame (x -= PIPE_SPEED)
    → score check (character.x > pipe.x + pipe.width → scored = true)
      → cleanup (pipe.x + pipe.width < 0 → removed from array)
```

- A pipe is created at `x = CANVAS_WIDTH` (right edge of canvas)
- It scrolls left at `PIPE_SPEED` pixels per frame
- It is scored exactly once, when the character clears its right edge
- It is removed from the `pipes` array when its right edge exits the left side of the canvas
- The `scored` flag is set on the pipe object itself — `ScoreManager` does not maintain a separate set of scored pipe IDs

### PipePair Object Shape

```js
{
  x:          canvasWidth,          // current left edge position
  gapY:       <random in bounds>,   // y of gap top
  gapHeight:  GAP_HEIGHT,           // constant 150px
  width:      PIPE_WIDTH,           // constant 60px
  scored:     false,                // true once character has passed
}
```

---

## Input Handling Rules

- `InputHandler.init` is called **once** at startup — never re-registered on state transitions
- Both `keydown` (Space) and `click` (canvas) call the same `onAction` function
- No debounce is needed — the state check inside `onAction` naturally prevents invalid transitions
- Space presses during `START` or `GAMEOVER` trigger exactly one transition per press because `gameState` changes immediately
- Rapid Space presses during `PLAYING` each trigger a flap — this is intentional

```js
const InputHandler = {
  init(canvas, onAction) {
    document.addEventListener("keydown", e => {
      if (e.code === "Space") {
        e.preventDefault(); // prevent page scroll
        onAction();
      }
    });
    canvas.addEventListener("click", () => onAction());
  },
};
```

- `e.preventDefault()` on Space prevents the browser from scrolling the page during gameplay

---

## Audio Event Map

| Game event                        | Audio call                    |
|-----------------------------------|-------------------------------|
| Player flaps (Playing State)      | `AudioManager.playJump()`     |
| Collision → Game Over transition  | `AudioManager.playGameOver()` |

Audio is never played during `START` or `GAMEOVER` states except for the game-over sound on the transition itself.

---

## Correctness Invariants (Domain Level)

These must hold across all game sessions:

1. `gameState` is always one of `STATE.START`, `STATE.PLAYING`, `STATE.GAMEOVER` — never `undefined` or any other value
2. `character.x` is always `80` — never modified by any module
3. `ScoreManager.score` is always `>= 0` and always an integer
4. Every pipe in `PipeSpawner.pipes` satisfies `MIN_GAP_Y <= gapY <= MAX_GAP_Y`
5. No pipe in `PipeSpawner.pipes` has `x + width < 0` (cleanup removes them before the next frame renders)
6. `pipe.scored` transitions from `false` to `true` exactly once per pipe, never back to `false`
7. Physics (gravity, flap, position update) is never applied outside `STATE.PLAYING`
