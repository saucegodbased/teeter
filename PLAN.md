# Plan: Fixed Track with Direct Ball Movement

## Problem Analysis

The current implementation has two related issues that make it feel like the track is tilting rather than the ball moving independently:

### Issue 1: Track visually tilts
**Root cause**: `renderer.js:113-117` contains `updateTrackTilt()` which rotates the `trackGroup` based on head tilt angle. This creates a visual tilting effect that makes the track appear to tilt rather than the ball moving on a fixed surface.

**Current code** (`renderer.js:113-117`):
```js
export function updateTrackTilt(tiltAngle) {
  const maxVisualTilt = 0.15;
  trackGroup.rotation.z = Math.max(-maxVisualTilt, Math.min(maxVisualTilt, tiltAngle * 0.5));
}
```

**Fix**: Remove all track tilting. The `updateTrackTilt` function should be removed, and the `trackGroup` should never have its rotation changed. Remove all calls to `updateTrackTilt` in `main.js`.

### Issue 2: Lateral movement uses gravity-on-slope model
**Root cause**: `physics.js:41` computes lateral acceleration as `GRAVITY * Math.sin(tiltAngle) * SENSITIVITY`, simulating a ball rolling on a tilted surface. Combined with the visual tilt, this reinforces the "tilting track" feel. The task wants **direct** control: head tilt directly controls the ball's lateral velocity.

**Current code** (`physics.js:41-45`):
```js
const ax = GRAVITY * Math.sin(tiltAngle) * SENSITIVITY;
ball.vx += ax * dt;
ball.vx *= (1 - FRICTION * dt);
```

**Fix**: Replace with direct velocity mapping:
```js
const targetVx = tiltAngle * DIRECT_SENSITIVITY;
ball.vx += (targetVx - ball.vx) * RESPONSE_RATE * dt;
```
This gives immediate, proportional control with smooth interpolation. The ball's lateral speed is directly proportional to head tilt angle, with smoothing to prevent jerky movement.

## Changes Required

### File 1: `js/renderer.js`
- Remove `updateTrackTilt()` function body — make it a no-op or remove entirely
- Remove `trackGroup` wrapper — add track meshes directly to `scene` since no rotation is needed
- Remove `updateTrackTilt` from exports

### File 2: `js/physics.js`
- Replace gravity-on-slope lateral model with direct velocity mapping
- Use interpolation toward a target velocity for smooth feel
- Tune `DIRECT_SENSITIVITY` to give responsive but controllable lateral movement

### File 3: `js/main.js`
- Remove `updateTrackTilt` import and all calls to it
- Everything else (pitch wiring, reset logic) stays the same

## Key Design Decisions

1. **Direct velocity with interpolation** vs **direct position mapping**: Using velocity with interpolation (`ball.vx lerps toward tiltAngle * sensitivity`) gives smooth, responsive control while still feeling like a physical ball. Pure position mapping would lose the rolling/momentum feel.

2. **Remove trackGroup entirely**: Since there's no more visual tilt, the group serves no purpose. Simplifying the scene graph by adding track meshes directly to the scene.

3. **Keep existing pitch control**: Forward speed modulation via head pitch already works correctly from the prior PR.

4. **Keep ball rolling visuals and edge-fall physics**: These match the task description ("rolling physics with momentum and friction", "falls off edge → reset").

## Scope Assessment

**Single agent** — all changes are tightly coupled across 3 files with a total diff of ~20 lines. No independent modules to parallelize.
