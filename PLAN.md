# Plan: Add Version/Last-Updated Timestamp to UI

## Overview

Add a small, fixed-position text element displaying a version string and last-updated date to the bottom-right corner of the game UI in `index.html`.

## Codebase Analysis

- **Tech stack**: Pure static HTML+JS (ES modules), Three.js v0.183.2 via CDN importmap, served by nginx in Docker
- **UI layout**: `#score` top-left, `#leaderboard-btn` top-right, centered overlays for game-over and leaderboard
- **Styling**: White text on dark backgrounds, `-apple-system` font stack, `rgba()` for opacity, `position: fixed` for HUD elements
- **Bottom-right corner is unused** — ideal location for version stamp

## Technical Approach

### Single file change: `index.html`

1. **Add CSS** for `#version-info` inside existing `<style>` block:
   - `position: fixed; bottom: 8px; right: 12px;`
   - Small font (~0.7em), muted color (`rgba(255,255,255,0.3)`)
   - `pointer-events: none` to avoid interfering with gameplay
   - `z-index: 5` — below all other HUD elements and overlays
   - Same font-family as existing UI

2. **Add HTML element** before `<script>` tag:
   ```html
   <div id="version-info">v1.0.0 · Last updated: 2026-03-20</div>
   ```

No JavaScript changes needed. The version string is a hardcoded value easily updatable by editing the one line.

## Scope Assessment: Single Agent

This is a ~10-line change to a single file. No parallelism needed.

## Sources

No external research needed — pure HTML/CSS addition following existing codebase patterns.
