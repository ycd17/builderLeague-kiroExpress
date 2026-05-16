# JavaScript Game Coding Standards

## Module Organization

- Organize game logic into focused plain objects (not ES modules) so the game runs without a build step or server
- Each module has a single responsibility: physics, rendering, input, audio, etc.
- Expose only the methods needed by the game loop; keep internal helpers as local variables or closures
- Define all game constants at the top of the script block, grouped and commented

```js
// ── Constants ──────────────────────────────────────────────────────────────
const CANVAS_WIDTH   = 480;
const CANVAS_HEIGHT  = 640;
const GRAVITY        = 0.5;   // px/frame²
const FLAP_IMPULSE   = -9;    // px/frame (negative = upward)
const MAX_FALL_SPEED = 12;    // terminal velocity cap
const PIPE_SPEED     = 3;     // px/frame
```

## Naming Conventions

| Concept | Convention | Example |
|---|---|---|
| Constants | SCREAMING_SNAKE_CASE | `CANVAS_WIDTH`, `FLAP_IMPULSE` |
| Module objects | PascalCase | `PhysicsEngine`, `CollisionDetector` |
| Plain data objects | camelCase | `character`, `pipePair` |
| Methods | camelCase verbs | `applyGravity()`, `drawPipes()` |
| State enum values | SCREAMING_SNAKE_CASE on a `STATE` object | `STATE.PLAYING` |
| Boolean flags | `is`/`has` prefix | `isLoaded`, `hasScored` |

## Game Loop Pattern

- Drive the loop exclusively with `requestAnimationFrame` — never `setInterval` or `setTimeout`
- Keep the loop function thin: delegate to modules, do not inline logic
- Guard state transitions with explicit state checks before applying physics or input

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
  }
  Renderer.draw(ctx, gameState, character, PipeSpawner.pipes, ScoreManager.score, BackgroundScroller.offset);
  requestAnimationFrame(gameLoop);
}
```

## Physics Conventions

- The canvas Y-axis increases **downward**: `vy < 0` means moving upward
- Always clamp velocity after applying gravity: `vy = Math.min(vy + GRAVITY, MAX_FALL_SPEED)`
- Flap sets velocity directly (not additive): `character.vy = FLAP_IMPULSE`
- Apply velocity to position after clamping, not before

## Collision Detection

- Use strict AABB (axis-aligned bounding box) overlap — touching edges do not count as a collision
- The overlap test must be symmetric: `rectOverlap(a, b) === rectOverlap(b, a)`
- Check boundary collisions (top and bottom of canvas) in the same pass as pipe collisions
- Evaluate collisions every frame during the Playing State only

```js
rectOverlap(a, b) {
  return a.x < b.x + b.width  &&
         a.x + a.width  > b.x &&
         a.y < b.y + b.height &&
         a.y + a.height > b.y;
}
```

## Rendering

- Clear the canvas at the start of each frame before drawing anything
- Draw in back-to-front order: background → pipes → character → UI overlays
- Use `ctx.save()` / `ctx.restore()` around any transform or style changes
- Prefer `ctx.fillRect` and `ctx.drawImage` over paths for performance in tight loops
- Use `image-rendering: pixelated` on the canvas element for crisp pixel art scaling

```css
canvas {
  image-rendering: pixelated;
  image-rendering: crisp-edges; /* Firefox fallback */
}
```

## Asset Loading

- Load images with `new Image()` and track success via `onload`/`onerror` flags
- Always provide a colored-rectangle fallback when a sprite fails to load
- Load audio with `new Audio()` inside a `try/catch`; swallow errors silently
- Wrap every `.play()` call in `try/catch` and chain `.catch(() => {})` on the returned Promise

```js
playJump() {
  try {
    this.jumpSound.currentTime = 0;
    this.jumpSound.play().catch(() => {});
  } catch (_) { /* silent fallback */ }
}
```

## Performance Guidelines

- Avoid allocating objects inside the game loop (no `new`, no array literals, no object literals per frame)
- Reuse arrays by mutating them (splice, push) rather than creating new ones
- Cache `canvas.getContext("2d")` once at startup; never call it inside the loop
- Use integer pixel coordinates where possible — sub-pixel rendering is slower and blurry on pixel-art games
- Limit `ctx.font` and `ctx.fillStyle` assignments; set them once per draw call group, not per character

## Property-Based Testing

- Tag every property test with a comment: `// Feature: <name>, Property N: <description>`
- Use [fast-check](https://github.com/dubzzz/fast-check) for all PBT; run a minimum of 100 iterations per property
- Keep property tests co-located with the module they test (same file, clearly separated section)
- Properties must be deterministic given the same seed — avoid `Math.random()` inside property tests
- Each property should validate exactly one correctness invariant; do not bundle multiple invariants per test

## Error Handling

- Never let audio or asset errors propagate as unhandled exceptions — always catch and swallow
- Check `canvas.getContext("2d")` for `null` at startup and display a plain-text fallback if unsupported
- Validate that constants are sane (e.g., `GAP_HEIGHT > 0`, `SPAWN_INTERVAL > 0`) with assertions during development
