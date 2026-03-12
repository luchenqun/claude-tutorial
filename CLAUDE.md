# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A browser-based Snake game implemented as a single HTML file (`snake.html`). No build tools, no dependencies, no package manager — open the file directly in a browser.

## Running the Game

```bash
open snake.html
```

## Architecture

The entire game lives in `snake.html` as inline `<style>` and `<script>` blocks. There are no external files or modules.

**Game loop:** `setInterval` drives `update()` at a speed determined by the current level. On collision, `clearInterval` stops the loop and `requestAnimationFrame` takes over for the dizzy animation.

**Rendering pipeline (each frame):**
1. `drawBackground()` — checkered green grid
2. `drawFood()` — apple with leaf/stem
3. `drawSnakeBody()` — thick stroked path (round caps/joins) for smooth tube look
4. `drawSnakeHead()` / `drawDizzyHead()` — head with face or dizzy X-eyes + orbiting stars

**Key state variables:**
- `snake` — array of `{x, y}` grid cells, head at index 0
- `dir` / `nextDir` — current and queued direction (prevents 180° reversal)
- `dizzy` / `dizzyStartTime` — controls the death animation state
- `running` / `paused` — game lifecycle flags

**Collision triggers `startDizzy()`**, which runs a 1.4s `requestAnimationFrame` loop showing the dizzy animation, then calls `gameOver()` to display the overlay.

**Difficulty scaling:** every 50 points increases `level` by 1; speed = `max(60, 150 - (level-1) * 10)` ms per tick.
