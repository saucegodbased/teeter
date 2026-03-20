# Implementation Plan: Obstacles, Coins, and Score Tracking

## Overview

Add wall obstacles, collectible coins, and a score HUD to the Teeter ball game. The ball must navigate around obstacles (collision = reset) and collect coins (proximity = score increment). Score displays on-screen and resets on death.

## Architecture

The game is pure static HTML+JS with Three.js (v0.183.2, CDN). Four modules:
- `index.html` — HTML shell with overlay UI
- `js/main.js` — Game loop, state management
- `js/physics.js` — Ball physics, collision detection
- `js/renderer.js` — Three.js scene (track, ball, camera, lighting)
- `js/tracker.js` — MediaPipe head tracking (DO NOT MODIFY)

All changes stay within this architecture. No new dependencies needed.

## Design Decisions

### Track Width: 3 → 4.5 units
The current track is 3 units wide. Increasing to 4.5 gives enough room for the ball (radius 0.3) to maneuver around obstacles (width ~1.0–1.5 units) while still making avoidance challenging. The `TRACK_WIDTH` constant in `renderer.js` is the source of truth — physics.js reads it via `getTrackConfig()`.

### Obstacle Placement
- Obstacles are box meshes placed on the track surface at semi-random lateral positions
- Spacing: one obstacle every 7–9 Z units (randomized within range)
- Lateral position: random within track width, but always leaving at least 1.5 units of passable gap on one side
- Safe zone: no obstacles in first 5 Z units from `BALL_START_Z` (-20), so nothing until Z ≈ -15
- Obstacle dimensions: ~1.5 wide × 1.0 tall × 0.4 deep — visually distinct red/dark color
- Track length is 50, ball starts at Z=-20, so obstacles span roughly Z=-15 to Z=+25 (about 5-6 obstacles)

### Coin Placement
- Coins are small golden torus meshes that rotate slowly on the Y-axis
- Placed between obstacles: 2-3 coins between each pair of obstacles
- Lateral position: slightly randomized to encourage lateral movement
- Safe zone: same as obstacles, none in first 5 units from start
- Collection radius: 0.8 units (ball radius 0.3 + coin radius 0.3 + small margin)

### Collision Detection (physics.js)
- Obstacle collision: AABB check — if ball center is within obstacle bounds (with ball radius margin), trigger `falling = true`
- Coin collection: distance check — if ball XZ distance to coin center < collection radius, mark coin as collected
- Both checks happen in `updateOnTrack()` each frame
- Physics module needs obstacle/coin positions — passed via extended config

### Data Flow
1. `renderer.js` generates obstacle/coin positions during `initRenderer()` using a deterministic pseudo-random approach
2. Positions are exposed via `getObstacles()` and `getCoins()` exports
3. `physics.js` receives these arrays via `initPhysics(config)` and checks collisions each frame
4. When a coin is collected, physics returns its index in the result object; main.js tells renderer to hide it
5. On reset, main.js tells renderer to show all coins again and resets physics coin tracking

### Score HUD
- Add `<div id="score">Score: 0</div>` to `index.html`, positioned top-left, styled with CSS
- `main.js` owns the score state (simple integer), updates the DOM element
- Score increments when physics reports a coin collection
- Score resets to 0 in the existing reset handler

### Track Wrapping
The current physics wraps the ball from Z=+25 back to Z=-24 when it reaches the end. After wrapping, all coins should reappear (reset visibility) and score keeps accumulating. Obstacles are always present.

## File Changes

### `index.html`
- Add score HUD div with CSS styling (fixed position, top-left, large font, semi-transparent background)

### `js/renderer.js`
- Change `TRACK_WIDTH` from 3 to 4.5
- Add obstacle mesh creation (box geometry with distinct dark red material)
- Add coin mesh creation (torus geometry with golden metallic material)
- Export `getObstacles()`, `getCoins()`, `hideCoin(index)`, `showAllCoins()`, `updateCoinRotation(dt)`
- Update edge positions for new track width (auto from TRACK_WIDTH constant)

### `js/physics.js`
- Accept obstacle and coin position arrays in config via `initPhysics()`
- Add obstacle collision check in `updateOnTrack()` — set `falling = true` and return `obstacleHit: true`
- Add coin proximity check in `updateOnTrack()` — return `coinsCollected: [indices]` for newly collected coins
- Track which coins have been collected (boolean array), reset on `resetBall()`

### `js/main.js`
- Import new renderer functions (`getObstacles`, `getCoins`, `hideCoin`, `showAllCoins`, `updateCoinRotation`)
- Pass obstacle/coin data to physics via config
- Handle `obstacleHit` from physics result (trigger falling state)
- Handle `coinsCollected` from physics result (hide coins via renderer, increment score, update HUD)
- Reset score to 0 and call `showAllCoins()` on ball reset
- Call `updateCoinRotation(dt)` in game loop for coin spin animation

## Performance
- Obstacles: ~6 simple box meshes — negligible GPU/CPU impact
- Coins: ~15-20 small torus meshes — negligible
- Collision detection: simple AABB and distance checks, O(n) with n < 25 — negligible per frame
- No new textures, shaders, or complex geometry
- Coin rotation is a simple Y-axis increment per frame

## Scope Assessment: Single Agent
All changes are tightly coupled — obstacles and coins need coordinated changes across renderer, physics, and main. Splitting into parallel agents would create merge conflicts and interface coordination overhead. Single agent is correct.
