# Flappy Kiro — Visual Design Reference

## Canvas Setup

```css
canvas {
  display: block;
  margin: 0 auto;
  image-rendering: pixelated;
  image-rendering: crisp-edges; /* Firefox */
}

body {
  margin: 0;
  padding: 0;
  background: #1a1a2e;
  font-family: monospace;
}
```

- Always set `image-rendering: pixelated` on the canvas element — this is what gives sprites their crisp, blocky pixel-art look when scaled
- Center the canvas with `margin: 0 auto` and a dark page background so the canvas edges don't bleed into the page

---

## Sprite Rendering Patterns

### Image Loading with Fallback

```js
const ascendingSprite  = new Image();
const descendingSprite = new Image();
let ascendingLoaded  = false;
let descendingLoaded = false;

ascendingSprite.onload  = () => { ascendingLoaded  = true; };
ascendingSprite.onerror = () => { ascendingLoaded  = false; };
descendingSprite.onload  = () => { descendingLoaded  = true; };
descendingSprite.onerror = () => { descendingLoaded  = false; };

ascendingSprite.src  = "assets/inv1.jpg";
descendingSprite.src = "assets/inv2.jpg";
```

- Track load state with boolean flags, not by checking `.complete` — `complete` can be true before `onload` fires in some browsers
- Set `src` **after** attaching `onload`/`onerror` handlers to avoid race conditions

### Drawing a Sprite with Rectangle Fallback

```js
// Renderer.drawCharacter(ctx, character, sprites)
function drawCharacter(ctx, character) {
  const useAscending = character.vy < 0;
  const sprite  = useAscending ? ascendingSprite  : descendingSprite;
  const isLoaded = useAscending ? ascendingLoaded : descendingLoaded;

  ctx.save();
  if (isLoaded) {
    ctx.drawImage(sprite, character.x, character.y, character.width, character.height);
  } else {
    // Fallback: colored rectangle
    ctx.fillStyle = "#a8e6cf";
    ctx.fillRect(character.x, character.y, character.width, character.height);
    // Accent border
    ctx.strokeStyle = "#3d9970";
    ctx.lineWidth = 2;
    ctx.strokeRect(character.x + 1, character.y + 1, character.width - 2, character.height - 2);
  }
  ctx.restore();
}
```

- Always wrap draw calls in `ctx.save()` / `ctx.restore()` to prevent style leakage between modules
- The fallback rectangle must be visually distinct and the same size as the sprite hitbox

### Sprite Selection Rule

| `character.vy` | Sprite used          | Asset             |
|----------------|----------------------|-------------------|
| `< 0`          | Ascending (flapping) | `assets/inv1.jpg` |
| `>= 0`         | Descending (falling) | `assets/inv2.jpg` |

Sprite swaps on the **same frame** the velocity changes — no delay, no interpolation.

---

## Pipe Rendering

```js
// Renderer.drawPipes(ctx, pipes)
function drawPipes(ctx, pipes) {
  for (const pipe of pipes) {
    const topH    = pipe.gapY;
    const botY    = pipe.gapY + pipe.gapHeight;
    const botH    = CANVAS_HEIGHT - botY;

    ctx.save();

    // Pipe body — retro green
    ctx.fillStyle = "#4caf50";
    ctx.fillRect(pipe.x, 0,    pipe.width, topH); // top pipe
    ctx.fillRect(pipe.x, botY, pipe.width, botH); // bottom pipe

    // Dark border for depth
    ctx.strokeStyle = "#1b5e20";
    ctx.lineWidth = 3;
    ctx.strokeRect(pipe.x, 0,    pipe.width, topH);
    ctx.strokeRect(pipe.x, botY, pipe.width, botH);

    // Pipe cap (slightly wider, at the gap edge)
    const capW = pipe.width + 8;
    const capH = 16;
    const capX = pipe.x - 4;
    ctx.fillStyle = "#388e3c";
    ctx.fillRect(capX, topH - capH, capW, capH);  // top cap
    ctx.fillRect(capX, botY,        capW, capH);  // bottom cap
    ctx.strokeStyle = "#1b5e20";
    ctx.strokeRect(capX, topH - capH, capW, capH);
    ctx.strokeRect(capX, botY,        capW, capH);

    ctx.restore();
  }
}
```

- Draw order within a pipe: body first, then cap on top — caps overlap the body edge
- Cap is 8px wider than the pipe body (4px each side) and 16px tall, matching classic Flappy Bird style

---

## Background Rendering

### Seamless Tiled Scroll

```js
// Renderer.drawBackground(ctx, bgOffset)
function drawBackground(ctx, bgOffset) {
  // Sky gradient
  const grad = ctx.createLinearGradient(0, 0, 0, CANVAS_HEIGHT);
  grad.addColorStop(0,   "#1a1a2e");
  grad.addColorStop(0.6, "#16213e");
  grad.addColorStop(1,   "#0f3460");
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // Scrolling star layer — drawn at offset, wraps at tileWidth
  ctx.fillStyle = "#ffffff";
  for (const star of STARS) {
    const sx = ((star.x - bgOffset) % TILE_WIDTH + TILE_WIDTH) % TILE_WIDTH;
    ctx.fillRect(sx, star.y, star.size, star.size);
  }
}
```

- Pre-generate `STARS` once at startup (not inside the loop) as an array of `{x, y, size}` objects
- Use `((x - offset) % tileWidth + tileWidth) % tileWidth` to handle negative modulo correctly in JS
- `createLinearGradient` is called every frame here for simplicity; if profiling shows it's slow, cache the gradient object

### Pre-generating Stars (called once at startup)

```js
const TILE_WIDTH = 400;
const STARS = [];
for (let i = 0; i < 60; i++) {
  STARS.push({
    x:    Math.floor(Math.random() * TILE_WIDTH),
    y:    Math.floor(Math.random() * CANVAS_HEIGHT),
    size: Math.random() < 0.8 ? 1 : 2,  // mostly 1px, some 2px
  });
}
```

---

## Text and UI Overlays

### Font Stack

```js
ctx.font = "bold 32px monospace";
```

- Always use `monospace` — it's the closest to a bitmap/pixel-art font available without loading external assets
- Use `bold` weight for titles and scores; regular weight for instructions

### Centered Text Helper

```js
function drawCenteredText(ctx, text, y) {
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(text, CANVAS_WIDTH / 2, y);
}
```

- Set `textAlign` and `textBaseline` before every text draw call — don't assume they carry over from a previous frame

### Start Screen Layout

```js
// Renderer.drawStartScreen(ctx)
function drawStartScreen(ctx) {
  ctx.save();

  // Title
  ctx.fillStyle = "#ffd700";
  ctx.font = "bold 40px monospace";
  drawCenteredText(ctx, "Flappy Kiro", CANVAS_HEIGHT * 0.35);

  // Subtitle / instruction
  ctx.fillStyle = "#ffffff";
  ctx.font = "18px monospace";
  drawCenteredText(ctx, "Press Space or Click to Begin", CANVAS_HEIGHT * 0.55);

  ctx.restore();
}
```

### Game Over Screen Layout

```js
// Renderer.drawGameOverScreen(ctx, score)
function drawGameOverScreen(ctx, score) {
  ctx.save();

  // Semi-transparent overlay
  ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  // "Game Over" heading
  ctx.fillStyle = "#ff5252";
  ctx.font = "bold 40px monospace";
  drawCenteredText(ctx, "Game Over", CANVAS_HEIGHT * 0.35);

  // Final score
  ctx.fillStyle = "#ffd700";
  ctx.font = "bold 28px monospace";
  drawCenteredText(ctx, `Score: ${score}`, CANVAS_HEIGHT * 0.48);

  // Restart instruction
  ctx.fillStyle = "#ffffff";
  ctx.font = "18px monospace";
  drawCenteredText(ctx, "Press Space or Click to Restart", CANVAS_HEIGHT * 0.62);

  ctx.restore();
}
```

### Live Score Display

```js
// Renderer.drawScore(ctx, score)
function drawScore(ctx, score) {
  ctx.save();
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 28px monospace";
  ctx.textAlign = "center";
  ctx.textBaseline = "top";
  ctx.fillText(score, CANVAS_WIDTH / 2, 16);
  ctx.restore();
}
```

- Score sits at `y=16` (top-center), always rendered last so it appears above pipes and background

---

## Draw Order (Back to Front)

Every frame must draw in this exact order to avoid z-order artifacts:

1. `Renderer.drawBackground(ctx, bgOffset)` — sky and stars
2. `Renderer.drawPipes(ctx, pipes)` — pipe bodies and caps
3. `Renderer.drawCharacter(ctx, character)` — sprite or fallback rect
4. `Renderer.drawScore(ctx, score)` — live score (Playing State only)
5. `Renderer.drawStartScreen(ctx)` — overlay (START state only)
6. `Renderer.drawGameOverScreen(ctx, score)` — overlay (GAMEOVER state only)

Steps 5 and 6 are mutually exclusive. Steps 1–3 always run (background and pipes still scroll on the Game Over screen).

---

## Animation Guidelines

### No Sprite Sheet Animation

Flappy Kiro uses only two static sprites (ascending / descending). There is no frame-by-frame sprite animation — the visual feedback comes entirely from swapping between the two images based on `vy`.

### Smooth Scrolling

- Background and pipes scroll every frame at a constant pixel rate — no easing, no acceleration
- Integer pixel positions (`Math.floor`) prevent sub-pixel blurring on pixel-art assets

```js
pipe.x = Math.floor(pipe.x - PIPE_SPEED);
```

### Screen Flash on Game Over (optional enhancement)

If a brief flash effect is desired on collision, implement it as a simple overlay alpha that decays over ~10 frames:

```js
let flashAlpha = 0;

// On collision:
flashAlpha = 0.6;

// In render loop (after all other draws):
if (flashAlpha > 0) {
  ctx.fillStyle = `rgba(255, 80, 80, ${flashAlpha})`;
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  flashAlpha = Math.max(0, flashAlpha - 0.06);
}
```

- Keep flash state as a module-level variable, not inside the render function
- Reset `flashAlpha = 0` when transitioning back to Playing State

---

## Particle Effects (optional enhancement)

If feather/dust particles are added on flap, follow these rules to stay within the single-file, no-build constraint:

### Particle Object Shape

```js
// Created on flap, pushed into a module-level array
{
  x: character.x + character.width / 2,
  y: character.y + character.height / 2,
  vx: (Math.random() - 0.5) * 3,
  vy: Math.random() * 2 + 1,   // drift downward
  life: 1.0,                    // 1.0 = fully opaque, 0 = dead
  size: Math.floor(Math.random() * 3) + 2,
}
```

### Particle Update and Draw

```js
// Update (called every frame in PLAYING state)
for (let i = particles.length - 1; i >= 0; i--) {
  const p = particles[i];
  p.x += p.vx;
  p.y += p.vy;
  p.life -= 0.05;
  if (p.life <= 0) particles.splice(i, 1);
}

// Draw (after character, before UI overlays)
for (const p of particles) {
  ctx.globalAlpha = p.life;
  ctx.fillStyle = "#a8e6cf";
  ctx.fillRect(Math.floor(p.x), Math.floor(p.y), p.size, p.size);
}
ctx.globalAlpha = 1; // always reset after use
```

- Iterate particles in reverse when splicing to avoid index skipping
- Always reset `ctx.globalAlpha = 1` after drawing particles — failing to do so will make everything else transparent
- Cap particle count at 30 to avoid performance impact: `if (particles.length < 30) particles.push(newParticle)`
- Clear all particles when transitioning to GAMEOVER or back to PLAYING
