# Requirements Document

## Introduction

Flappy Kiro is a retro-style endless scroller arcade game that runs entirely in the browser as a single HTML file. The player controls a ghost character navigating through an infinite series of pipe obstacles. The game is inspired by Flappy Bird and uses a pixel-art aesthetic with custom sprite assets. No external libraries are required — the game is implemented in pure HTML, CSS, and JavaScript.

## Glossary

- **Game**: The Flappy Kiro browser application running in a single `.html` file.
- **Player**: The human operating the game via keyboard or mouse input.
- **Character**: The ghost sprite controlled by the Player, rendered on the Canvas.
- **Canvas**: The HTML5 `<canvas>` element used to render all game graphics.
- **Pipe**: A vertical obstacle pair (top and bottom) with a gap the Character must pass through.
- **Gap**: The vertical opening between the top and bottom segments of a Pipe pair.
- **Gravity**: The constant downward acceleration applied to the Character each frame.
- **Flap**: The upward velocity impulse applied to the Character when the Player inputs a jump action.
- **Score**: The integer count of Pipe pairs the Character has successfully passed through.
- **Ascending Sprite**: The image asset (`assets/inv1.jpg`) displayed when the Character's vertical velocity is upward (positive).
- **Descending Sprite**: The image asset (`assets/inv2.jpg`) displayed when the Character's vertical velocity is downward (negative or zero).
- **Start Screen**: The initial game state shown before gameplay begins.
- **Playing State**: The active game state during which the Character moves and Pipes scroll.
- **Game Over Screen**: The state shown after the Character collides with a Pipe or boundary.
- **Renderer**: The JavaScript module responsible for drawing all visual elements onto the Canvas.
- **Physics Engine**: The JavaScript logic responsible for applying Gravity and Flap impulses to the Character.
- **Collision Detector**: The JavaScript logic responsible for detecting contact between the Character and Pipes or screen boundaries.
- **Pipe Spawner**: The JavaScript logic responsible for generating new Pipe pairs at regular intervals.
- **Audio Manager**: The JavaScript logic responsible for loading and playing sound effects.
- **Background**: The scrolling retro-style visual layer rendered behind all other game elements.

---

## Requirements

### Requirement 1: Game State Management

**User Story:** As a Player, I want the game to progress through distinct screens, so that I have a clear start, play, and end experience.

#### Acceptance Criteria

1. THE Game SHALL display the Start Screen when first loaded in the browser.
2. WHEN the Player presses the Space key or clicks the Canvas on the Start Screen, THE Game SHALL transition to the Playing State.
3. WHEN the Collision Detector detects a collision during the Playing State, THE Game SHALL transition to the Game Over Screen.
4. WHEN the Player presses the Space key or clicks the Canvas on the Game Over Screen, THE Game SHALL reset all game state and transition to the Playing State.
5. THE Game SHALL run entirely within a single `.html` file without requiring any external libraries or network requests beyond loading local asset files.

---

### Requirement 2: Character Physics

**User Story:** As a Player, I want the character to fall under gravity and jump when I press a button, so that I can control its vertical position through the pipes.

#### Acceptance Criteria

1. WHILE in the Playing State, THE Physics Engine SHALL apply a constant downward Gravity acceleration to the Character on every animation frame.
2. WHEN the Player presses the Space key or clicks the Canvas during the Playing State, THE Physics Engine SHALL apply an upward Flap velocity impulse to the Character.
3. THE Character SHALL maintain a fixed horizontal position on the Canvas while Pipes scroll toward it from the right.
4. WHILE in the Playing State, THE Physics Engine SHALL update the Character's vertical position each frame based on its current velocity.
5. THE Character's vertical velocity SHALL be capped at a maximum downward speed to prevent uncontrollable acceleration.

---

### Requirement 3: Pipe Obstacle Generation and Scrolling

**User Story:** As a Player, I want an endless series of pipe obstacles to scroll toward me, so that the game presents a continuous challenge.

#### Acceptance Criteria

1. WHILE in the Playing State, THE Pipe Spawner SHALL generate a new Pipe pair at a fixed horizontal interval.
2. THE Pipe Spawner SHALL randomize the vertical position of the Gap within defined minimum and maximum bounds on each spawn.
3. WHILE in the Playing State, THE Renderer SHALL scroll all active Pipe pairs leftward at a constant speed each frame.
4. WHEN a Pipe pair moves entirely off the left edge of the Canvas, THE Game SHALL remove that Pipe pair from the active set.
5. THE Gap height SHALL remain constant across all generated Pipe pairs.

---

### Requirement 4: Collision Detection

**User Story:** As a Player, I want the game to end when I hit a pipe or the screen edge, so that the game has meaningful consequences for poor navigation.

#### Acceptance Criteria

1. WHEN the Character's bounding box overlaps with any Pipe segment's bounding box, THE Collision Detector SHALL signal a collision.
2. WHEN the Character's vertical position moves above the top boundary of the Canvas, THE Collision Detector SHALL signal a collision.
3. WHEN the Character's vertical position moves below the bottom boundary of the Canvas, THE Collision Detector SHALL signal a collision.
4. THE Collision Detector SHALL evaluate collision conditions on every animation frame during the Playing State.

---

### Requirement 5: Scoring

**User Story:** As a Player, I want my score to increase as I pass through pipes, so that I can track my progress and compete with myself.

#### Acceptance Criteria

1. WHEN the Character's horizontal position passes the right edge of a Pipe pair's Gap during the Playing State, THE Game SHALL increment the Score by 1.
2. THE Renderer SHALL display the current Score on the Canvas during the Playing State.
3. THE Game Over Screen SHALL display the final Score achieved in the round.
4. WHEN the Game transitions to the Playing State from the Game Over Screen, THE Game SHALL reset the Score to 0.

---

### Requirement 6: Two-State Character Sprite

**User Story:** As a Player, I want the character's appearance to change based on its movement direction, so that the game feels visually responsive and alive.

#### Acceptance Criteria

1. WHILE the Character's vertical velocity is greater than zero (moving upward), THE Renderer SHALL display the Ascending Sprite for the Character.
2. WHILE the Character's vertical velocity is less than or equal to zero (moving downward or stationary), THE Renderer SHALL display the Descending Sprite for the Character.
3. THE Renderer SHALL swap between the Ascending Sprite and Descending Sprite on the same frame that the Character's velocity changes direction.
4. THE Game SHALL load the Ascending Sprite from `assets/inv1.jpg` and the Descending Sprite from `assets/inv2.jpg`.

---

### Requirement 7: Retro Visual Style

**User Story:** As a Player, I want the game to have a retro pixel-art aesthetic, so that it feels like a classic arcade experience.

#### Acceptance Criteria

1. THE Renderer SHALL render a continuously scrolling Background behind all other game elements during the Playing State and Game Over Screen.
2. THE Renderer SHALL style Pipes using a retro color palette consistent with classic arcade visuals.
3. THE Game SHALL use pixel-art-appropriate fonts (such as a monospace or bitmap-style font) for all on-screen text.
4. THE Background SHALL loop seamlessly so that no visible seam appears during continuous scrolling.

---

### Requirement 8: Audio Feedback

**User Story:** As a Player, I want sound effects to play on key game events, so that the game feels engaging and responsive.

#### Acceptance Criteria

1. WHEN the Player triggers a Flap during the Playing State, THE Audio Manager SHALL play the sound effect from `assets/jump.wav`.
2. WHEN the Game transitions to the Game Over Screen, THE Audio Manager SHALL play the sound effect from `assets/game_over.mp3`.
3. IF the browser does not support audio playback or the audio file fails to load, THEN THE Audio Manager SHALL allow the game to continue without audio and SHALL NOT throw an unhandled error.

---

### Requirement 9: Single-File Delivery

**User Story:** As a Player, I want to open the game by simply opening one HTML file in my browser, so that there is no setup or installation required.

#### Acceptance Criteria

1. THE Game SHALL be fully contained within a single `.html` file.
2. THE Game SHALL reference asset files (`assets/inv1.jpg`, `assets/inv2.jpg`, `assets/jump.wav`, `assets/game_over.mp3`) using relative paths from the `.html` file's location.
3. THE Game SHALL run correctly in any modern browser (Chrome, Firefox, Safari, Edge) without requiring a web server, build step, or external dependencies.

---

### Requirement 10: Start and Game Over Screens

**User Story:** As a Player, I want clear instructions on the start and game over screens, so that I know how to begin and restart the game.

#### Acceptance Criteria

1. THE Start Screen SHALL display the game title "Flappy Kiro" and an instruction indicating the Player should press Space or click to begin.
2. THE Game Over Screen SHALL display a "Game Over" message, the final Score, and an instruction indicating the Player should press Space or click to restart.
3. THE Renderer SHALL render Start Screen and Game Over Screen content centered on the Canvas.
4. WHILE the Game is on the Start Screen or Game Over Screen, THE Physics Engine SHALL NOT apply Gravity or Flap impulses to the Character.
