# Tetris Test Agent Guide

This repository is specification-first. The current goal is to let a new engineer or coding agent implement the game without needing prior conversation context.

## Read First

Before planning or editing code, read these files in order:

1. `README.md`
2. `docs/DEVELOPMENT_FRAMEWORK.md`

Treat `docs/DEVELOPMENT_FRAMEWORK.md` as the product and architecture specification. Do not silently change its scope or rules.

## Complete Project Mission

Implement the complete playable first version described in `docs/DEVELOPMENT_FRAMEWORK.md`. Execute phases 1 through 7 in order, beginning with the project baseline:

- Create the `src/tetris/` and `tests/` package layout described in the framework.
- Add `pyproject.toml` and `.gitignore`.
- Add the smallest pygame window that starts through the `tetris` console command and exits cleanly.
- Configure pytest and Ruff.
- During phase 1, do not implement pieces, board rules, scoring, rendering, persistence, sounds, Hold, Ghost Piece, SRS, wall kicks, or lock delay.

After each phase passes its specified validation, continue automatically to the next phase. Do not stop after the baseline unless a real blocker requires user input. The task is complete only when phase 7 is implemented, all automated checks pass, and the full game has been manually played through.

## Architecture Rules

- `board.py`, `piece.py`, `shapes.py`, `randomizer.py`, and `scoring.py` must not import pygame.
- `InputHandler` only maps pygame events to `GameAction`; it does not mutate game state.
- `Game` is the only object that coordinates rule-state changes.
- `Renderer` reads game state but never mutates `Board` or `Piece`.
- Randomness must be injectable so tests can use a fixed seed.
- Movement and rotation use candidate immutable `Piece` values and commit them only when `Board.can_place()` succeeds.
- The first version uses clockwise rotation only. Invalid rotations are cancelled without wall kicks.
- The board is 10×20 visible cells with no hidden rows.
- A blocked spawn means immediate Game Over.
- A piece locks immediately when it cannot move down. There is no lock delay.
- Time-based falling uses a delta-time accumulator and catches up by multiple steps when required.

## Required Validation

Run the complete project checks before reporting a phase complete:

```bash
pytest
ruff check .
ruff format --check .
```

For user-visible phases, also launch `tetris` and manually exercise the workflow added in that phase.

## Delivery Format

Every implementation handoff must report:

1. What changed.
2. Why it changed.
3. Commands run and their results.
4. Manual behavior verified, when applicable.
5. Remaining risks or intentionally deferred work.

Keep implementation work ordered by phase. Do not pull later features into an earlier milestone, but continue through all phases until the complete first version is delivered.
