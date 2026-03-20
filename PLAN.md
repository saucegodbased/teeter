# Plan: Randomize Obstacle and Coin Layout Each Run

## Problem
`seededRandom(42)` in `renderer.js` produces the same layout every run. Players memorize the course and replayability drops.

## Approach
Replace the hardcoded seed with `Date.now()` and introduce a `regenerateLevel()` function that tears down existing obstacle/coin meshes and creates new ones with a fresh seed. Call this during the reset flow in `main.js`.

## Changes

### 1. renderer.js — `regenerateLevel()` function

- Extract the obstacle/coin mesh creation code (lines 177–212) into a new exported function `regenerateLevel()`.
- `regenerateLevel()` will:
  1. Remove all existing obstacle meshes from `scene` via `scene.remove(mesh)`.
  2. Remove all existing coin meshes from `scene` similarly.
  3. Clear `obstacleMeshes`, `obstacleData`, `coinMeshes`, `coinData` arrays.
  4. Create a new RNG with `seededRandom(Date.now())`.
  5. Regenerate `obstacleData` and `coinData` using existing `generateObstacles()` / `generateCoins()`.
  6. Create new meshes and add them to the scene (same code as current init).
  7. Return `{ obstacles: getObstacles(), coins: getCoins() }` so the caller can update physics.
- In `initRenderer()`, replace the inline generation block with a call to `regenerateLevel()` (DRY).

**Geometry/Material reuse**: Obstacle and coin geometries/materials are identical across all instances. Store them as module-level variables created once in `initRenderer()`, reuse in `regenerateLevel()`. Only dispose meshes on teardown, not shared geo/mat.

### 2. physics.js — `updateLevel()` function

- Add exported function `updateLevel(obstacles, coins)` that:
  1. Replaces the `obstacles` and `coins` arrays.
  2. Resets `coinsCollected` to a fresh `Array(coins.length).fill(false)`.
- This allows main.js to update physics collision data without a full `initPhysics()` call.

### 3. main.js — Call during reset

- Import `regenerateLevel` from renderer.js and `updateLevel` from physics.js.
- In the reset timer callback (line 122–133), after `resetBall()`:
  1. Call `const level = regenerateLevel()`.
  2. Call `updateLevel(level.obstacles, level.coins)`.
- This replaces the current `showAllCoins()` call (no longer needed since `regenerateLevel()` creates fresh visible coins).

### 4. main.js — Initial load

- `initRenderer()` calls `regenerateLevel()` internally for the initial layout.
- The init flow in main.js already calls `getObstacles()` and `getCoins()` after `initRenderer()` — this continues to work unchanged.

## Memory Leak Prevention
- Each `regenerateLevel()` call removes old meshes from scene.
- Shared geometry and material are created once and reused — never disposed during gameplay.
- Only mesh objects are created/destroyed per regeneration.

## Validation
- `generateObstacles()` and `generateCoins()` are unchanged — spacing rules, safe zone, gap requirements remain intact.
- Ball spawn position is unchanged — `resetBall()` always uses `trackConfig.ballStartZ`.
- `Date.now()` guarantees a different seed each run (millisecond precision).

## Testing
- `docker build -t teeter .` must succeed (no build step — just static file copy).
- Visual: obstacle/coin layout differs after each death/reset.
- Gameplay: collisions, coin collection, score reset all still work.

## Scope Assessment: Single Agent
All changes are tightly coupled across 3 files with shared interfaces. No parallel decomposition is beneficial.

## Sources
No external libraries or APIs needed. All changes use existing Three.js patterns already in the codebase.
