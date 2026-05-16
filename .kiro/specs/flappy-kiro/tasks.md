# Implementation Plan: Flappy Kiro

## Overview

Implement Flappy Kiro as a single self-contained `.html` file using pure HTML5, CSS, and JavaScript. The game follows a game-loop architecture driven by `requestAnimationFrame` and is organized into focused modules: PhysicsEngine, PipeSpawner, CollisionDetector, ScoreManager, Renderer, AudioManager, InputHandler, and BackgroundScroller. All assets are referenced via relative paths.

## Tasks

- [~] 1. Scaffold the single HTML file with canvas and constants
  - Create `flappy-kiro.html` with a `<canvas>` element sized to `CANVAS_WIDTH=480` × `CANVAS_HEIGHT=640`
  - Define all game constants (`GRAVITY`, `FLAP_IMPULSE`, `MAX_FALL_SPEED`, `PIPE_SPEED`, `PIPE_WIDTH`, `GAP_HEIGHT`, `SPAWN_INTERVAL`, `BG_SPEED`) inside a `<script>` block
  - Define the `STATE` object (`START`, `PLAYING`, `GAMEOVER`) and initialize `gameState = STATE.START`
  - Add a canvas-not-supported fallback message as inner HTML of the `<canvas>` element
  - Apply pixel-art-appropriate CSS (monospace/bitmap font, no margin/padding on body, `image-rendering: pixelated` on canvas)
  - _Requirements: 1.1, 1.5, 7.3, 9.1, 9.3_

- [ ] 2. Implement asset loading (sprites and audio)
  - [~] 2.1 Load sprite images with fallback
    - Create `ascendingSprite` and `descendingSprite` via `new Image()`, set `src` to `assets/inv1.jpg` and `assets/inv2.jpg`
    - Track load success with `onload`/`onerror` flags so the Renderer can fall back to a colored rectangle if loading fails
    - _Requirements: 6.4, 9.2_

  - [~] 2.2 Implement AudioManager with graceful fallback
    - Create `AudioManager` object with `load()`, `playJump()`, and `playGameOver()` methods
    - Wrap all `new Audio()` construction and `.play()` calls in `try/catch`; swallow errors silently so gameplay continues
    - Load `assets/jump.wav` and `assets/game_over.mp3` via relative paths
    - _Requirements: 8.1, 8.2, 8.3, 9.2_

- [ ] 3. Implement PhysicsEngine
  - [~] 3.1 Implement gravity, flap, and update methods
    - Create `PhysicsEngine` object with `applyGravity(character)`, `applyFlap(character)`, and `update(character)` methods
    - `applyGravity` increments `character.vy` by `GRAVITY` and clamps to `MAX_FALL_SPEED`
    - `applyFlap` sets `character.vy = FLAP_IMPULSE`
    - `update` increments `character.y` by `character.vy`
    - _Requirements: 2.1, 2.2, 2.4, 2.5_

  - [~] 3.2 Write property test: gravity monotonically increases downward velocity
    - **Property 1: Gravity monotonically increases downward velocity**
    - For random `vy` values below `MAX_FALL_SPEED`, one `applyGravity` call must increase `vy` toward `MAX_FALL_SPEED`
    - **Validates: Requirements 2.1, 2.5**

  - [~] 3.3 Write property test: flap always produces upward velocity
    - **Property 2: Flap always produces upward velocity**
    - For any random character state, `applyFlap` must set `vy` to exactly `FLAP_IMPULSE` (negative)
    - **Validates: Requirements 2.2**

  - [~] 3.4 Write property test: terminal velocity is never exceeded
    - **Property 3: Terminal velocity is never exceeded**
    - For any sequence of `applyGravity` calls (no flap), `character.vy` must never exceed `MAX_FALL_SPEED`
    - **Validates: Requirements 2.5**

- [ ] 4. Implement Character object and InputHandler
  - [~] 4.1 Define Character object
    - Create `character` plain object with fields `x=80`, `y=200`, `vy=0`, `width=40`, `height=40`
    - Add a `reset()` helper that restores `y=200` and `vy=0` for use on game restart
    - _Requirements: 2.3, 2.4_

  - [~] 4.2 Implement InputHandler
    - Create `InputHandler` object with `init(canvas, onAction)` method
    - Listen for `keydown` with `e.code === "Space"` on `document` and `click` on the canvas
    - Call `onAction()` on each event; the game-loop callback will guard against invalid state transitions
    - _Requirements: 1.2, 1.4, 2.2_

- [ ] 5. Implement PipeSpawner and pipe scrolling
  - [~] 5.1 Implement PipeSpawner with spawn, scroll, and cleanup
    - Create `PipeSpawner` object with `pipes` array, `spawnTimer`, `update(canvasWidth)`, `spawn(canvasWidth)`, `scroll(speed)`, and `cleanup()` methods
    - `spawn` pushes a new `PipePair` object with `x=canvasWidth`, random `gapY` in `[minGapY, maxGapY]`, constant `gapHeight=GAP_HEIGHT`, `width=PIPE_WIDTH`, `scored=false`
    - `cleanup` removes pipes whose `x + width < 0`
    - `update` decrements `spawnTimer`, calls `spawn` when it reaches 0 (reset to `SPAWN_INTERVAL`), then calls `scroll` and `cleanup`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [~] 5.2 Write property test: pipe gap position is always within bounds
    - **Property 4: Pipe gap position is always within bounds**
    - For any random seed passed to `PipeSpawner.spawn`, the resulting `gapY` must satisfy `minGapY ≤ gapY ≤ maxGapY`
    - **Validates: Requirements 3.2**

- [ ] 6. Implement CollisionDetector
  - [~] 6.1 Implement AABB overlap and full collision check
    - Create `CollisionDetector` object with `rectOverlap(a, b)` and `check(character, pipes, canvasHeight)` methods
    - `rectOverlap` returns `true` if two `{x, y, width, height}` rects overlap (strict overlap, not just touching)
    - `check` tests character against top and bottom segments of every pipe, plus canvas top (`character.y < 0`) and bottom (`character.y + character.height > canvasHeight`) boundaries
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [~] 6.2 Write property test: AABB collision detection is symmetric and complete
    - **Property 7: AABB overlap is symmetric and complete**
    - For random pairs of rectangles, `rectOverlap(a, b) === rectOverlap(b, a)` must always hold, and the result must match geometric overlap
    - **Validates: Requirements 4.1**

- [ ] 7. Implement ScoreManager
  - [~] 7.1 Implement score tracking and reset
    - Create `ScoreManager` object with `score=0`, `update(character, pipes)`, and `reset()` methods
    - `update` iterates pipes; for each pipe where `!pipe.scored && character.x > pipe.x + pipe.width`, increment `score` and set `pipe.scored = true`
    - `reset` sets `score = 0`
    - _Requirements: 5.1, 5.4_

  - [~] 7.2 Write property test: score is monotonically non-decreasing and correct
    - **Property 5: Pipe scoring is monotonically non-decreasing and correct**
    - For random sequences of pipe positions and character x-values, score must equal the count of passed pipes and must never decrease within a session
    - **Validates: Requirements 5.1, 5.4**

  - [~] 7.3 Write property test: score resets to zero on game restart
    - **Property 8: Score resets to zero on game restart**
    - For any random final score value, calling `ScoreManager.reset()` must set `score` to exactly `0`
    - **Validates: Requirements 5.4**

- [~] 8. Checkpoint — Ensure all logic modules pass their tests
  - Ensure all tests pass, ask the user if questions arise.

- [~] 9. Implement BackgroundScroller
  - Create `BackgroundScroller` object with `offset=0`, `speed=BG_SPEED`, `tileWidth=400`, and `update()` method
  - `update` advances `offset` by `speed` and wraps with `% tileWidth` to ensure seamless looping
  - _Requirements: 7.1, 7.4_

- [ ] 10. Implement Renderer
  - [~] 10.1 Implement background and pipe drawing
    - Implement `Renderer.drawBackground(ctx, bgOffset)` using a tiled fill that shifts by `bgOffset` and wraps seamlessly
    - Implement `Renderer.drawPipes(ctx, pipes)` drawing top and bottom pipe rectangles using a retro color palette (e.g., green with dark border)
    - _Requirements: 7.1, 7.2, 7.4_

  - [ ] 10.2 Implement character drawing with sprite selection
    - Implement `Renderer.drawCharacter(ctx, character, sprites)` selecting `ascendingSprite` when `character.vy < 0`, `descendingSprite` otherwise
    - Fall back to a colored rectangle if the sprite image failed to load
    - _Requirements: 6.1, 6.2, 6.3_

  - [ ] 10.3 Write property test: correct sprite selected for velocity direction
    - **Property 6: Correct sprite selected for velocity direction**
    - For random `vy` values, the sprite selection logic must return ascending sprite if and only if `vy < 0`
    - **Validates: Requirements 6.1, 6.2, 6.3**

  - [ ] 10.4 Implement score and screen overlays
    - Implement `Renderer.drawScore(ctx, score)` rendering the live score centered at the top of the canvas during Playing State
    - Implement `Renderer.drawStartScreen(ctx)` rendering "Flappy Kiro" title and "Press Space or Click to Begin" instruction, centered
    - Implement `Renderer.drawGameOverScreen(ctx, score)` rendering "Game Over", the final score, and "Press Space or Click to Restart", centered
    - Use pixel-art-appropriate font (monospace) for all text
    - _Requirements: 5.2, 5.3, 7.3, 10.1, 10.2, 10.3_

- [ ] 11. Implement the game loop and state machine
  - [ ] 11.1 Wire the game loop with requestAnimationFrame
    - Implement the main `gameLoop()` function called via `requestAnimationFrame`
    - In `PLAYING` state: call `PhysicsEngine.applyGravity`, `PhysicsEngine.update`, `BackgroundScroller.update`, `PipeSpawner.update`, `ScoreManager.update`, `CollisionDetector.check`; on collision call `AudioManager.playGameOver()` and transition to `GAMEOVER`
    - In `START` and `GAMEOVER` states: skip physics and pipe updates; render the appropriate screen
    - _Requirements: 1.1, 1.3, 2.1, 2.4, 3.3, 4.4, 10.4_

  - [ ] 11.2 Implement state transitions via InputHandler
    - Wire `InputHandler.init` so that `onAction` transitions `START → PLAYING` (reset character and pipes, reset score) and `GAMEOVER → PLAYING` (same reset)
    - During `PLAYING`, `onAction` calls `PhysicsEngine.applyFlap` and `AudioManager.playJump()`
    - Ensure physics is not applied during `START` or `GAMEOVER` states
    - _Requirements: 1.2, 1.4, 2.2, 5.4, 8.1, 10.4_

- [ ] 12. Final checkpoint — Ensure all tests pass and the game runs end-to-end
  - Ensure all tests pass, ask the user if questions arise.
  - Verify the game opens from `flappy-kiro.html` directly in a browser (no server needed)
  - Confirm Start Screen, Playing State, and Game Over Screen all render and transition correctly
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 9.3_

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property-based tests use [fast-check](https://github.com/dubzzz/fast-check) as described in the design's Testing Strategy section
- Each property test is tagged with `// Feature: flappy-kiro, Property N: <property text>`
- All logic is contained in a single `<script>` block inside `flappy-kiro.html` — no ES modules, no build step
- The canvas Y-axis increases downward: `vy < 0` means moving upward (ascending sprite), `vy >= 0` means moving downward (descending sprite)
- Sprite fallback (colored rectangle) ensures the game remains playable if image assets fail to load
