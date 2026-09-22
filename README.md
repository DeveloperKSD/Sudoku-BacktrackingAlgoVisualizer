# Sudoku GUI Solver 

A pygame Sudoku app with an animated backtracking solver, random puzzle
generation, notes, hints, undo, and best-time tracking.

## Setup & Run

```bash
pip install pygame
```
```bash
python GUI_enhanced.py
```

Requires a display (won't run over plain SSH without X forwarding). On first
launch you'll pick a difficulty; `best_times.json` is created automatically
next to the script once you finish a puzzle.

## Controls

| Key | Action |
|---|---|
| `1`–`9` / numpad | Pencil in a digit (or toggle a candidate note, in Notes mode) |
| `ENTER` | Confirm the pencilled digit into the cell |
| `DELETE` | Clear the pencil mark / notes in the selected cell |
| `SPACE` | Auto-solve with animation |
| `TAB` | Toggle Notes mode (multi-candidate pencil marks) |
| `N` | New game (choose difficulty again) |
| `R` | Restart current puzzle from its original clues |
| `U` | Undo last confirmed placement or hint |
| `H` | Hint — reveal the correct value for the selected cell |
| Mouse click | Select a cell |

## Basic Flow

1. **Menu** — `choose_difficulty()` blocks on a key event loop until the
   player picks Easy/Medium/Hard.
2. **Puzzle generation** — `generate_puzzle()` builds a full solved board,
   then removes cells one at a time, re-solving after each removal to check
   the puzzle still has exactly one solution, until it hits the clue count
   for the chosen difficulty.
3. **Game loop** (`main()`) — a standard pygame loop: poll events → mutate
   state (`Grid`/`Cube`) → redraw. Placing a digit re-validates the whole
   board with the same backtracking solver used for generation; if the
   partially-filled board is no longer solvable, the move is rejected and
   counted as a strike.
4. **End states** — filling the board ends the round with a "Solved!"
   overlay and a best-time check; reaching the strike limit shows "Game
   Over". Both are cleared by `N` (new puzzle) or `R` (retry).

## The Core Logic (unchanged from the original)

Three functions do all the actual Sudoku reasoning, and are used both to
**solve** a board and to **generate** one:

- **`find_empty(bo)`** — scans row by row for the first cell with value `0`.
  Returns `None` when the board is full.
- **`valid(bo, num, pos)`** — checks `num` isn't already present in `pos`'s
  row, column, or 3×3 box.
- **`solve(bo)`** — recursive backtracking:
  1. Find an empty cell. If there isn't one, the board is complete → `True`.
  2. Try digits 1–9 in that cell; for each one that's `valid`, place it and
     recurse.
  3. If the recursive call fails, undo the digit (`0`) and try the next one.
  4. If no digit works, return `False` and let the caller backtrack further.

This is exhaustive but fast enough for 9×9 because `valid` prunes almost
every branch immediately — the search rarely goes more than a few cells deep
before hitting a contradiction.

`solve_gui()` is the same algorithm with a `pygame.display.update()` +
`delay()` after every placement/undo, so the recursion is visible on screen
instead of running silently.

**Generation reuses the same two primitives:**
- Filling a full board is `solve()`'s logic with a *shuffled* digit order
  per cell, so each run produces a different valid, complete grid.
- Digging holes removes a cell, then runs a solution-counting variant of
  `solve()` (`_count_solutions`, capped at 2) on the result — if it finds
  more than one solution, the removal is undone. This guarantees every
  generated puzzle has exactly one answer.

## What Changed / What We Learned

Improving this project came down to finding places where new features could
sit **on top of** the existing algorithm rather than inside it:

- **Puzzle generation is the same recursion as solving**, just with random
  digit ordering — no new solving logic was needed to add "New Game".
- **Uniqueness checking** (`_count_solutions`) is a one-line variant of
  `solve()` — stop early once 2 solutions are found instead of 1. This is
  the standard trick for generating valid Sudoku puzzles.
- A subtle real bug in the original code: `Grid.board` was a **class**
  attribute (shared across all instances), not an instance attribute. It
  went unnoticed with a single hardcoded puzzle, but would've caused every
  `Grid()` to share state once multiple boards existed — fixed by moving it
  into `__init__`.
- Features like notes, undo, hints, and highlighting only needed new state
  (a `set()` of candidates, a history list, a `selected` peer calculation)
  layered onto the existing `Grid`/`Cube` classes — the solving/validation
  core never had to change.

## File

- `GUI_enhanced.py` — the entire app (generation, GUI, game loop) in one
  self-contained file.
