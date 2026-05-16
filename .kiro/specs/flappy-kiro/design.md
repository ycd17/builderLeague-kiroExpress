# Design Document: Flappy Kiro

## Overview

Flappy Kiro is a single-file browser game implemented in pure HTML5, CSS, and JavaScript. It is a Flappy Bird-style endless scroller where the player guides a ghost character through an infinite series of pipe obstacles by tapping Space or clicking the canvas.

The entire game runs inside one `.html` file. All rendering is done on an HTML5 `<canvas>` element using the 2D Canvas API. There are no external libraries, no build steps, and no server requirements — the file opens directly in any modern browser.

The game cycles through three states: **Start Screen → Playing State → Game Over Screen**. Physics, pipe generation, collision detection, scoring, sprite selection, and audio are all handled by small, focused JavaScript modules defined inside the single file.

---

## Architecture

The game follows a **game-loop architecture** driven by `requestAnimationFrame`. Each frame the loop:

1. Updates physics (velocity + position)
2. Scrolls pipes and background
3. Checks collisions
4. Updates score
5. Renders everything to the canvas

All logic lives inside a single `<script>` block. The code is organized into logical modules (plain objects or closures) rather than ES modules, so no server is needed to resolve imports.

```
┌─────────────────────────────────────────────────────┐
│                   Game Loop (rAF)                   │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Physics  │  │  Pipes   │  │    Background    │  │
│  │ Engine   │  │ Spawner  │  │    Scroller      │  │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘  │
│       │              │                 │             │
│  ┌────▼─────────────▼─────────────────▼──────────┐  │
│  │              Collision Detector               │  │
│  └────────────────────┬──────────────────────────┘  │
│                       │                             │
│  ┌────────────────────▼──────────────────────────┐  │
│  │                  Renderer                     │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────┐   ┌──────────────────────┐    │
│  │  Input Handler   │   │    Audio Manager     │    │
│  └──────────────────┘   └──────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**State machine:**

```
         Space / Click
              │
         ┌────▼────┐
         │  START  │
         └────┬────┘
              │ Space / Click
         ┌────▼────┐
         │ PLAYING │◄──────────────┐
         └────┬────┘               │
              │ collision          │ Space / Click
         ┌────▼────┐               │
         │  GAME   ├───────────────┘
         │  OVER   │
         └─────────┘
```

---

## Components and Interfaces

### GameState

A module-level variable (or small object) holding the current state string: `"start"`, `"playing"`, or `"gameover"`.

```js
// Possible values
const STATE = { START: "start", PLAYING: "playing", GAMEOVER: "gameover" };
let gameState = STATE.START;
```

### PhysicsEngine

Responsible for applying gravity and flap impulses to the character.

```js
const PhysicsEngine = {
  gravity: 0.5,          // pixels per frame² downward
  flapImpulse: -9,       // pixels per frame upward (negative = up)
  maxFallSpeed: 12,      // terminal velocity cap (pixels/frame downward)

  applyGravity(character) { /* velocity += gravity, clamp to maxFallSpeed */ },
  applyFlap(character)   { /* velocity = flapImpulse */ },
  update(character)      { /* character.y += character.vy */ },
};
```

### Character

Plain object holding position, velocity, and sprite state.

```js
const character = {
  x: 80,          // fixed horizontal position
  y: 200,         // vertical position (pixels from top)
  vy: 0,          // vertical velocity (negative = up)
  width: 40,
  height: 40,
};
```

### PipeSpawner

Generates pipe pairs at a fixed horizontal interval with randomized gap positions.

```js
const PipeSpawner = {
  spawnInterval: 90,   // frames between spawns
  pipeWidth: 60,
  gapHeight: 150,
  minGapY: 60,         // minimum top of gap from canvas top
  maxGapY: 300,        // maximum top of gap from canvas top

  spawnTimer: 0,
  pipes: [],           // array of PipePair objects

  update(canvasWidth) { /* decrement timer, spawn when 0, scroll all pipes */ },
  spawn(canvasWidth)  { /* push new PipePair with random gapY */ },
  scroll(speed)       { /* move all pipes left by speed */ },
  cleanup()           { /* remove pipes that have left the canvas */ },
};
```

### PipePair

Plain object representing one top+bottom pipe pair.

```js
// Created by PipeSpawner.spawn()
{
  x: canvasWidth,   // starts at right edge
  gapY: <random>,   // y-coordinate of gap top
  gapHeight: 150,
  width: 60,
  scored: false,    // true once the character has passed this pipe
}
```

### CollisionDetector

Evaluates AABB (axis-aligned bounding box) overlap between the character and each pipe segment, plus canvas boundary checks.

```js
const CollisionDetector = {
  check(character, pipes, canvasHeight) {
    // returns true if any collision is detected
  },
  rectOverlap(a, b) {
    // returns true if two {x,y,width,height} rects overlap
  },
};
```

### ScoreManager

Tracks the current score and detects pipe-pass events.

```js
const ScoreManager = {
  score: 0,
  update(character, pipes) {
    // for each pipe not yet scored, if character.x > pipe.x + pipe.width → score++, pipe.scored = true
  },
  reset() { this.score = 0; },
};
```

### Renderer

Draws all visual elements to the canvas each frame.

```js
const Renderer = {
  drawBackground(ctx, bgOffset)    { /* tiled/scrolling background */ },
  drawPipes(ctx, pipes)            { /* top and bottom pipe rects */ },
  drawCharacter(ctx, character, sprites) { /* ascending or descending sprite */ },
  drawScore(ctx, score)            { /* live score during play */ },
  drawStartScreen(ctx)             { /* title + instructions */ },
  drawGameOverScreen(ctx, score)   { /* game over + final score + instructions */ },
};
```

### AudioManager

Loads audio assets and plays them on demand, with graceful fallback.

```js
const AudioManager = {
  jumpSound: null,
  gameOverSound: null,

  load() {
    // create Audio objects, catch load errors silently
  },
  playJump()    { /* reset currentTime, play(), catch errors */ },
  playGameOver(){ /* reset currentTime, play(), catch errors */ },
};
```

### InputHandler

Listens for Space keydown and canvas click events, dispatches actions to the game.

```js
const InputHandler = {
  init(canvas, onAction) {
    document.addEventListener("keydown", e => { if (e.code === "Space") onAction(); });
    canvas.addEventListener("click", () => onAction());
  },
};
```

### BackgroundScroller

Tracks the horizontal offset used to create a seamless looping background.

```js
const BackgroundScroller = {
  offset: 0,
  speed: 2,          // pixels per frame
  tileWidth: 400,    // width of one background tile

  update() { this.offset = (this.offset + this.speed) % this.tileWidth; },
};
```

---

## Data Models

### Character State

| Field  | Type   | Description                                      |
|--------|--------|--------------------------------------------------|
| x      | number | Fixed horizontal position (pixels from left)     |
| y      | number | Vertical position of top-left corner (pixels)    |
| vy     | number | Vertical velocity (negative = upward)            |
| width  | number | Sprite/hitbox width in pixels                    |
| height | number | Sprite/hitbox height in pixels                   |

### PipePair State

| Field     | Type    | Description                                          |
|-----------|---------|------------------------------------------------------|
| x         | number  | Horizontal position of left edge (pixels from left)  |
| gapY      | number  | Y-coordinate of the top of the gap                   |
| gapHeight | number  | Height of the gap in pixels (constant)               |
| width     | number  | Width of the pipe in pixels (constant)               |
| scored    | boolean | Whether the character has already passed this pipe   |

### Game Constants

| Constant       | Value  | Description                                    |
|----------------|--------|------------------------------------------------|
| CANVAS_WIDTH   | 480    | Canvas width in pixels                         |
| CANVAS_HEIGHT  | 640    | Canvas height in pixels                        |
| GRAVITY        | 0.5    | Downward acceleration per frame                |
| FLAP_IMPULSE   | -9     | Upward velocity applied on flap                |
| MAX_FALL_SPEED | 12     | Terminal velocity cap (downward)               |
| PIPE_SPEED     | 3      | Pixels per frame pipes scroll left             |
| PIPE_WIDTH     | 60     | Width of each pipe segment                     |
| GAP_HEIGHT     | 150    | Vertical gap between top and bottom pipe       |
| SPAWN_INTERVAL | 90     | Frames between pipe spawns                     |
| BG_SPEED       | 2      | Background scroll speed (pixels/frame)         |

### Sprite Assets

| Asset              | Path                | Used When                          |
|--------------------|---------------------|------------------------------------|
| Ascending Sprite   | assets/inv1.jpg     | character.vy > 0 (moving upward)   |
| Descending Sprite  | assets/inv2.jpg     | character.vy ≤ 0 (moving downward) |

> **Note on velocity sign convention:** The canvas Y-axis increases downward. "Moving upward" means `vy < 0`. The requirements document uses the opposite convention ("greater than zero = moving upward"). The implementation will follow the canvas convention (`vy < 0` = ascending sprite) and the requirements will be interpreted accordingly.

### Audio Assets

| Asset          | Path                  | Trigger                        |
|----------------|-----------------------|--------------------------------|
| Jump sound     | assets/jump.wav       | Player triggers a flap         |
| Game over sound| assets/game_over.mp3  | Transition to Game Over Screen |

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Gravity monotonically increases downward velocity

*For any* character state with vertical velocity below the terminal velocity cap, applying one frame of gravity must increase the downward velocity (i.e., `vy` increases toward `MAX_FALL_SPEED`).

**Validates: Requirements 2.1, 2.5**

---

### Property 2: Flap always produces upward velocity

*For any* character state (regardless of current velocity), applying a flap must set the character's vertical velocity to exactly `FLAP_IMPULSE` (a negative value, meaning upward).

**Validates: Requirements 2.2**

---

### Property 3: Terminal velocity is never exceeded

*For any* sequence of gravity applications (without flap), the character's downward velocity (`vy`) must never exceed `MAX_FALL_SPEED`.

**Validates: Requirements 2.5**

---

### Property 4: Pipe gap position is always within bounds

*For any* generated pipe pair, the gap's top Y-coordinate must satisfy `minGapY ≤ gapY ≤ maxGapY`, ensuring the gap is always reachable.

**Validates: Requirements 3.2**

---

### Property 5: Pipe scoring is monotonically non-decreasing and correct

*For any* sequence of pipe pairs, the score must equal the number of pipes whose right edge the character has passed, and the score must never decrease during a single play session.

**Validates: Requirements 5.1, 5.4**

---

### Property 6: Correct sprite selected for velocity direction

*For any* character state, the sprite selected by the renderer must be the ascending sprite if and only if `vy < 0` (moving upward in canvas coordinates), and the descending sprite otherwise.

**Validates: Requirements 6.1, 6.2, 6.3**

---

### Property 7: AABB collision detection is symmetric and complete

*For any* two axis-aligned bounding boxes, `rectOverlap(a, b)` must return the same result as `rectOverlap(b, a)`, and must return `true` if and only if the rectangles geometrically overlap.

**Validates: Requirements 4.1**

---

### Property 8: Score resets to zero on game restart

*For any* completed game session (any final score), transitioning from Game Over back to Playing must reset the score to exactly 0.

**Validates: Requirements 5.4**

---

## Error Handling

### Audio Failures

The `AudioManager` wraps all `Audio` construction and `.play()` calls in `try/catch` blocks. If an audio file fails to load or the browser blocks autoplay, the error is silently swallowed and gameplay continues unaffected. No unhandled promise rejections are allowed.

```js
playJump() {
  try {
    this.jumpSound.currentTime = 0;
    this.jumpSound.play().catch(() => {});
  } catch (e) { /* silent fallback */ }
}
```

### Asset Loading

Sprite images are loaded via `new Image()`. If an image fails to load, the renderer falls back to drawing a colored rectangle in place of the sprite, so the game remains playable.

### Canvas Not Supported

If `canvas.getContext("2d")` returns `null` (very old browser), the game displays a plain-text fallback message inside the `<canvas>` element's inner HTML.

### Input Edge Cases

- Rapid repeated Space presses during Playing State each trigger a flap — this is intentional and correct behavior.
- Space presses during Start Screen or Game Over Screen trigger the state transition exactly once per press (debounced by state check).

---

## Testing Strategy

### Unit Tests

Unit tests cover specific examples and edge cases for the pure logic functions:

- **PhysicsEngine**: verify gravity increments velocity, flap sets velocity to impulse, terminal velocity clamp works at boundary values.
- **CollisionDetector.rectOverlap**: verify overlapping, touching, and non-overlapping rectangle pairs with concrete coordinates.
- **ScoreManager**: verify score increments exactly once per pipe, does not double-count, resets to 0 on restart.
- **PipeSpawner**: verify gap Y is within bounds for a sample of spawns, verify pipes are removed after leaving canvas.
- **Sprite selection**: verify ascending sprite is chosen when `vy < 0`, descending sprite when `vy >= 0`.

### Property-Based Tests

Property-based tests use [fast-check](https://github.com/dubzzz/fast-check) (or an equivalent PBT library for the target environment) to verify universal properties across randomly generated inputs. Each test runs a minimum of **100 iterations**.

Each test is tagged with a comment in the format:
`// Feature: flappy-kiro, Property N: <property text>`

| Property | Description | Generator Inputs |
|----------|-------------|-----------------|
| 1 | Gravity increases downward velocity | Random `vy` values below cap |
| 2 | Flap always produces upward velocity | Random character states |
| 3 | Terminal velocity never exceeded | Random sequences of gravity applications |
| 4 | Pipe gap always within bounds | Random `gapY` seeds via spawner |
| 5 | Score is monotonically non-decreasing | Random pipe sequences and character positions |
| 6 | Correct sprite for velocity direction | Random `vy` values |
| 7 | AABB overlap is symmetric and correct | Random pairs of rectangles |
| 8 | Score resets to 0 on restart | Random final scores |

### Integration / Smoke Tests

- Open the `.html` file in a browser and verify the Start Screen renders.
- Verify that pressing Space transitions to Playing State.
- Verify that hitting a pipe transitions to Game Over Screen with the correct score displayed.
- Verify audio plays on flap and game over (manual check, as audio autoplay policies vary by browser).

### What Is Not Property-Tested

- **Rendering correctness** (canvas pixel output) — verified by visual inspection and snapshot tests if a headless browser is available.
- **Audio playback** — verified by manual smoke test; audio behavior is non-deterministic across browsers.
- **Background seamless looping** — verified by visual inspection.
