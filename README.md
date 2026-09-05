# Tic Tac Toe with AI — Minimax & Alpha-Beta Pruning

A Tkinter tic-tac-toe game with two adversarial search opponents — **minimax** and **alpha-beta pruning** — playable against a human, against each other, or human vs human.

Tic-tac-toe is small enough to solve exactly, which makes it a clean setting for the point of the project: both algorithms return the same optimal move, so the difference between them is purely how much work they do to get there. On this game tree, alpha-beta visits **94% fewer nodes**.

---

## Benchmark

Choosing the first move on an empty board (measured on the search functions from this repo, with GUI calls stripped):

| Algorithm | Nodes visited | Time | Move chosen |
|---|---|---|---|
| Minimax | 549,945 | 2.55 s | `(0, 0)` |
| Alpha-beta | 30,709 | 0.15 s | `(0, 0)` |

**94.4% fewer nodes, ~17× faster, identical result.**

The 2.5-second first move is exactly why the alpha-beta version runs its search on a background thread — see *Threading* below.

---

## The two files

Each script is standalone and launches its own window.

| File | Contents |
|---|---|
| `tic_tac_toe_mini_max.py` | Plain minimax, search runs on the main thread |
| `tic_tac_toe_alpha_beta.py` | Both minimax and alpha-beta, search runs on a worker thread |

`tic_tac_toe_alpha_beta.py` is the complete version — start there.

---

## How the search works

Both algorithms share the same structure: `max_value` and `min_value` recurse into each other, alternating perspective at each ply.

**Terminal conditions.** A node is terminal when `check_victory()` is true or `turn > 9`. Crucially, the victory check returns from the perspective of the node being evaluated — `max_value` returns `-1` on a win, `min_value` returns `+1`. That looks backwards until you notice that a win detected *at* a node was caused by the move that led into it, so it's a loss for whoever is to move there.

**Scoring** is the minimal three-value scheme: `+1` win, `0` draw, `-1` loss. No heuristic evaluation function is needed because the tree is small enough to search to the end every time.

**Board copying.** Each candidate move is applied to a fresh `board.copy()` rather than being made and undone. Wasteful compared to make/unmake, but it makes the recursion trivially correct — no state can leak between branches.

**Alpha-beta's cutoffs.** In `max_value_ab`, once a value reaches `beta` the branch returns immediately: the minimizing player above would never allow this line, so the remaining siblings are irrelevant. `min_value_ab` mirrors it against `alpha`. That single early return is the whole 94%.

---

## Threading

Minimax blocks for 2.5 seconds on its first move. Run on the main thread, that freezes the entire Tk window — no repaints, no responding to the OS.

`tic_tac_toe_alpha_beta.py` solves this properly:

1. The search runs in a `Thread`, writing its result into a `Queue`
2. `ai_wait_for_move()` checks the queue; if empty, it reschedules itself with `window.after(100, ...)`
3. Control returns to the Tk event loop between polls, so the window stays alive

This is the correct pattern for long-running work in Tkinter — Tk is not thread-safe, so the worker computes a plain value and only the main thread ever touches widgets.

---

## Interface

- 300×300 canvas, grid drawn as four lines
- Click a cell to place a symbol on your turn
- Two dropdowns select each player independently: `human`, `AI: Min-Max`, `AI: alpha-beta`
- **New game** resets the board; **Quit** closes the window
- The winning three symbols turn red, and a status label tracks turn number and whose move it is

Any pairing works, including AI vs AI — useful for watching two optimal players draw every time.

---

## Running it

```bash
git clone https://github.com/RovshanBayramRB/AI-for-Tic-Tac-Toe.git
cd AI-for-Tic-Tac-Toe
pip install numpy
python tic_tac_toe_alpha_beta.py
```

Requires Python 3 with Tkinter (bundled on Windows and macOS; on Debian/Ubuntu install `python3-tk`). A display is required — these will not run headless.
