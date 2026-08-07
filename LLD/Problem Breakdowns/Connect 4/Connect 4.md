# LLD Problem Breakdown: Connect Four

*Free HelloInterview problem — good starting point before Premium. Rated "easy."*

---

## 1. Clarifying Questions to Ask First

Prompt you'd likely get: *"Build the object-oriented design for a two-player Connect Four game. Players take turns dropping discs into a 7-column, 6-row board. First to align four of their own discs vertically, horizontally, or diagonally wins."*

Work through the same four question themes from the Delivery Framework:

| Theme | Question | Typical answer |
|---|---|---|
| Core actions | How do players interact — column number, disc drops automatically? | Yes — player picks column 0-6, disc falls to lowest open spot |
| Rules & completion | What ends the game? | Four in a row (any direction) = win; full board, no winner = draw |
| Error handling | Full column? Out-of-turn move? | Reject clearly — return `false`/error, don't corrupt state |
| Scope boundaries | Single game or concurrent? Backend-only or UI too? | Single game only; backend logic only (no rendering) |
| Future extensions | Move history/undo? Configurable board size? | No — always 7x6, no undo needed |

**Final requirements to write down:**
```
Requirements:
1. Two players take turns dropping discs into a 7-column, 6-row board
2. A disc falls to the lowest available row in the chosen column
3. Game ends when: a player connects 4 (any direction) → win,
   OR board fills with no winner → draw
4. Invalid moves rejected clearly: full column, out-of-turn move,
   move after game over

Out of scope: UI, concurrent games, move history, undo, configurable board size
```

**Why "backend only" matters as a question:** if UI support were in scope, you'd want methods like `getBoardState()`/`getValidMoves()` for something to render against. Backend-only means you can keep the API minimal and focused purely on game rules — asking this question up front shapes your whole interface design, not just a detail.

---

## 2. Identify Entities

| Entity | Responsibility |
|---|---|
| **Game** | Orchestrator. Holds the board, tracks whose turn it is, manages game state, enforces turn-based rules. Validates each move, delegates placement to Board, checks for a win, switches turns. |
| **Board** | The 7x6 grid. Owns grid state, disc placement, column-full checks, and win detection. Doesn't know or care whose turn it is. |
| **Player** | Pure data — name/ID and disc color. No game logic at all. |

**Deliberately kept small:** three entities, each with one clear job. A common mistake here is over-splitting (a separate class per win-direction, for instance — see Section 4) or under-splitting (cramming board logic into Game). Board manages grid state and placement rules; Game orchestrates turns and win-checking; Player is just data — that division is the whole design.

---

## 3. Class Design

### Game — deriving state from requirements

| Requirement | What Game must track |
|---|---|
| "Two players take turns..." | The two players, whose turn it is, and the board |
| "Game ends when a player wins or board is full" | Game state (in progress / won / draw) |
| "A player gets four in a row" | Who won, if anyone |

**Key design decision — avoid the boolean-flag trap.** A common mistake is tracking `isOver`, `hasWinner`, `isDraw` as three separate booleans. This lets you represent *impossible* states your domain doesn't actually allow — e.g. `isOver=false, hasWinner=true` (someone won but the game isn't over?), or `hasWinner=true, isDraw=true` (both a win and a draw?). Nothing in the type system stops you from getting these out of sync.

**The fix: a single `GameState` enum** — `IN_PROGRESS`, `WON`, `DRAW`. One field, three actual states, invalid combinations become impossible by construction rather than something you have to remember to prevent. This is the general principle of **making invalid states unrepresentable** — worth having ready as a phrase in interviews, since it signals you're thinking about type design, not just "get it working."

*(One remaining gap: `winner` is still a separate nullable field, so in theory `state=IN_PROGRESS` with a non-null `winner` is still representable. Languages with tagged/sum types — Rust, Swift, Kotlin sealed classes, TypeScript discriminated unions — can attach `winner` directly to the `WON` variant and eliminate this too. Java can't do this elegantly, so a plain enum + nullable `winner` is the right practical call — just be aware you're trusting yourself to keep them in sync, and mention the ideal exists if asked.)*

```java
class Game {
    private Board board;
    private Player player1, player2;
    private Player currentPlayer;
    private GameState state;      // IN_PROGRESS, WON, DRAW
    private Player winner;        // null if no winner yet or draw
}
```

**Deriving Game's methods from requirements:**

| Need | Method |
|---|---|
| "Players take turns dropping discs" | `makeMove(player, column)` — the core action |
| "Reject moves out of turn" | `getCurrentPlayer()` |
| "Game ends when..." | `getGameState()` |
| "Four in a row" | `getWinner()` |

It's normal to discover methods that weren't explicit in requirements as you go (e.g. `getCurrentPlayer()` for a UI to display whose turn it is) — that's expected iterative refinement, not a sign you missed something the first time.

### Board — deriving state from requirements

| Requirement | What Board must track |
|---|---|
| "7-column, 6-row board" | Fixed `rows`/`cols` |
| "Disc falls to lowest available row" | Current occupancy of each column (the grid) |
| "Board is full → draw" | Whether any empty cell remains |
| "Four discs in a row" | Enough grid info to check contiguous discs for a color |

```java
class Board {
    private static final int ROWS = 6;
    private static final int COLS = 7;
    private DiscColor[][] grid = new DiscColor[ROWS][COLS]; // null = empty
}
```

**Design decision — store `DiscColor` in the grid, not `Player`.** Keeps `Board` independently testable: a simpler value type, no need to construct/mock a `Player` object just to test grid logic. Either choice works as long as you're consistent, but this is the more testable one.

### Player — deliberately minimal

```java
class Player {
    private String name;
    private DiscColor color;
}
```

No behavior at all — just identity + color. `name` lets `Game` compare/validate whose turn it is; `color` links placed discs back to their owner in the grid.

---

## 4. Full Class Diagram (Java)

```java
enum GameState { IN_PROGRESS, WON, DRAW }
enum DiscColor { RED, YELLOW }

class Player {
    private String name;
    private DiscColor color;

    Player(String name, DiscColor color) { this.name = name; this.color = color; }
    String getName() { return name; }
    DiscColor getColor() { return color; }
}

class Board {
    private static final int ROWS = 6;
    private static final int COLS = 7;
    private DiscColor[][] grid = new DiscColor[ROWS][COLS];

    int getRows() { return ROWS; }
    int getCols() { return COLS; }
    boolean canPlace(int column) { /* ... */ return true; }
    int placeDisc(int column, DiscColor color) { /* returns row, or -1 */ return -1; }
    boolean isFull() { /* ... */ return false; }
    boolean checkWin(int row, int column, DiscColor color) { /* ... */ return false; }
    DiscColor getCell(int row, int column) { return grid[row][column]; }
}

class Game {
    private Board board;
    private Player player1, player2, currentPlayer;
    private GameState state;
    private Player winner;

    Game(Player player1, Player player2) {
        this.board = new Board();
        this.player1 = player1;
        this.player2 = player2;
        this.currentPlayer = player1;
        this.state = GameState.IN_PROGRESS;
    }

    boolean makeMove(Player player, int column) { /* see Section 5 */ return false; }
    Player getCurrentPlayer() { return currentPlayer; }
    GameState getGameState() { return state; }
    Player getWinner() { return winner; }
    Board getBoard() { return board; }
}
```

---

## 5. Implementation — the Three Interesting Methods

Companies vary on whether they want pseudocode, real code, or a talk-through — ask. For each method: happy path first, then edge cases.

### `makeMove` (the core method — encapsulates the whole game flow)

**Logic:** place disc → check win → check draw → switch turn → return true.
**Edge cases (reject before touching state):** game already over; wrong player's turn; invalid/full column (delegated to Board).

```java
boolean makeMove(Player player, int column) {
    if (state != GameState.IN_PROGRESS) return false;
    if (player != currentPlayer) return false;

    int row = board.placeDisc(column, player.getColor());
    if (row == -1) return false; // Board rejected — full or out of bounds

    if (board.checkWin(row, column, player.getColor())) {
        state = GameState.WON;
        winner = player;
    } else if (board.isFull()) {
        state = GameState.DRAW;
    } else {
        currentPlayer = (player == player1) ? player2 : player1;
    }
    return true;
}
```

**Note the separation of concerns:** `Game` never checks column bounds or fullness itself — it delegates entirely to `board.placeDisc()` and just checks for `-1`. Game handles game rules (turns, state); Board handles grid rules (bounds, placement). Don't duplicate that validation in both places.

**Worth mentioning as a tradeoff if asked:** `makeMove` returns `false` on invalid input rather than throwing — simpler and clearer in an interview setting, though throwing is a defensible alternative in some languages/teams. Also worth flagging: `makeMove` takes `player` explicitly rather than implicitly using `currentPlayer` — this matters if you imagine a networked setting where moves arrive tagged with a specific player; for a single local game, implicit would also be fine. Either is reasonable as long as you can explain the choice.

### `placeDisc`

**Logic:** scan upward from the bottom row (`rows - 1`) in the given column until an empty cell is found; place the disc; return the row.
**Edge cases:** out-of-bounds column → `-1`; full column → `-1`.

```java
int placeDisc(int column, DiscColor color) {
    if (column < 0 || column >= COLS) return -1;
    if (!canPlace(column)) return -1;

    for (int row = ROWS - 1; row >= 0; row--) {
        if (grid[row][column] == null) {
            grid[row][column] = color;
            return row;
        }
    }
    return -1;
}
```

All validation lives inside `placeDisc` rather than `Game` calling `canPlace()` first — keeps grid-related validation in one place, and `Game` only needs to check for `-1`.

### `checkWin` — the interesting design decision

**This is the section most likely to trip up a candidate who over-engineers.**

#### ❌ Over-engineered approach: a `WinChecker` interface with 4 implementations
It's tempting to see "4 directions" and think "4 classes implementing a common interface" (Strategy pattern). **Don't.** Connect Four's win directions will never change — there's no future requirement where a 5th direction appears or the check needs to swap at runtime. Building that flexibility is a **YAGNI violation**: you're adding extensibility for a requirement that doesn't exist, and all four "different" checkers would do the literal same thing with different `(dr, dc)` numbers anyway. Strategy earns its keep when behavior genuinely varies and might be swapped later (e.g. payment methods) — not when the "variation" is just four instances of identical logic with different constants.

#### ✅ Right-sized approach: direction vectors + one shared counting method
Recognize that horizontal/vertical/both diagonals are **the same algorithm** — count contiguous same-color discs in a direction — parameterized by a `(dr, dc)` pair, not four different behaviors.

```java
boolean checkWin(int row, int col, DiscColor color) {
    if (row < 0 || row >= ROWS || col < 0 || col >= COLS) return false;
    if (grid[row][col] != color) return false;

    int[][] directions = {{0,1}, {1,0}, {1,1}, {-1,1}};
    for (int[] dir : directions) {
        int count = 1;
        count += countInDirection(row, col, dir[0], dir[1], color);
        count += countInDirection(row, col, -dir[0], -dir[1], color);
        if (count >= 4) return true;
    }
    return false;
}

private int countInDirection(int row, int col, int dr, int dc, DiscColor color) {
    int count = 0;
    int r = row + dr, c = col + dc;
    while (inBounds(r, c) && grid[r][c] == color) {
        count++;
        r += dr;
        c += dc;
    }
    return count;
}

private boolean inBounds(int row, int col) {
    return row >= 0 && row < ROWS && col >= 0 && col < COLS;
}
```

`(0,1)` = horizontal, `(1,0)` = vertical, `(1,1)`/`(-1,1)` = the two diagonals. One method, called twice per direction (forward and its negation) to count both ways from the placed disc. **This is the single best moment in the whole problem to demonstrate you know when *not* to reach for a pattern** — separating data (the direction vectors) from logic (the counting loop) instead of separating behavior into classes.

### Remaining helpers
```java
boolean canPlace(int column) {
    if (column < 0 || column >= COLS) return false;
    return grid[0][column] == null; // top row empty = column has room
}

boolean isFull() {
    for (int c = 0; c < COLS; c++) {
        if (canPlace(c)) return false;
    }
    return true;
}
```

`Player` has no interesting implementation — just getters. Skip walking through it unless explicitly asked.

---

## 6. Verification (trace a scenario — don't skip this)

Quick example: with two RED discs already stacked at the bottom of column 0, placing a third RED disc via `makeMove(player1, 0)`:
1. `placeDisc(0, RED)` → lands at row 2 (third from bottom)
2. `checkWin(2, 0, RED)` → checks vertical direction `(1,0)`: counts down through rows 3, 4, 5 (all RED) → `count = 1 + 3 = 4` → **true**
3. `state = WON`, `winner = player1`
4. A subsequent `makeMove()` call from player2 immediately returns `false` since `state != IN_PROGRESS`

You don't need to write out a full trace this detailed in the interview — talking through 1-2 test cases verbally is usually enough. The goal is catching your own logic errors before the interviewer does.

---

## 7. Extensibility — Common Follow-Ups

| Follow-up | Where the change lives | Why it's clean |
|---|---|---|
| **Configurable board size** | `rows`/`cols` become constructor params on `Board` | Placement/win logic already works for arbitrary dimensions — it only ever uses `rows`, `cols`, `inBounds()`, never hardcoded numbers |
| **Undo / move history** | New `moveHistory` stack in `Game`; a `Move(player, row, col)` value object; `Board.clearCell()` helper | All moves already flow through the single `makeMove()` choke point — natural place to record history without touching Board's core logic |
| **Computer opponent** | New standalone `BotEngine` with `chooseMove(game) -> column`; game loop calls `makeMove(botPlayer, botEngine.chooseMove(game))` | Game rules **don't change at all** — a bot move is just another `makeMove()` call. `Player` stays plain data; making `Player` an interface with `HumanPlayer`/`BotPlayer` would add abstraction without real value, since a human doesn't "do" anything — it's just data being read from elsewhere (input vs. bot logic) |

**Level expectations (per HelloInterview's own framing):**
- **Junior:** working game, correct horizontal/vertical win checks at least (diagonal hints OK), basic error handling
- **Mid-level:** clean separation without guidance, direction-vector win-checking (not 4 separate methods), can discuss 1 extensibility scenario
- **Senior:** proactively justifies decisions (why `GameState` is an enum, why win-checking lives on `Board` not `Game`), catches own edge cases unprompted, can discuss multiple extensibility tradeoffs (e.g. networked multiplayer, spectator support)

---

## 8. Suggested Resources

- HelloInterview — [Connect Four](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/connect-four) — free problem breakdown; source for the framework above (original in Python/pseudocode, translated to Java here).
- Cross-reference: the `GameState` enum vs. boolean-flags decision is a great concrete example of **"make invalid states unrepresentable"** — worth remembering as a phrase for any problem with a small number of mutually exclusive states.
- Cross-reference: the `checkWin` over-engineering trap is the clearest YAGNI violation example in this repo so far — pair with `general-principles-notes.md` Section 3.

---

## 9. Self-Test — Multiple Choice

<details>
<summary>Q1: Why is a single GameState enum better than three separate booleans (isOver, hasWinner, isDraw)?</summary>

**A)** Enums are faster to evaluate at runtime
**B)** The enum makes impossible state combinations (e.g. hasWinner=true and isDraw=true at once) unrepresentable, while three independent booleans allow 8 combinations for only 3 valid states
**C)** Booleans use more memory than enums
**D)** Enums are easier to serialize to JSON

**Answer: B** — this is "make invalid states unrepresentable" in action. The type system itself prevents bugs instead of relying on you to keep three flags manually in sync.
</details>

<details>
<summary>Q2: Why is building 4 separate WinChecker classes (Horizontal, Vertical, DiagonalUp, DiagonalDown) considered over-engineering here?</summary>

**A)** Java doesn't support multiple interfaces
**B)** Connect Four's win directions are fixed forever — there's no real variation to swap at runtime, and all four "checkers" would be identical logic with different constants, so Strategy adds ceremony without solving a real extensibility need (a YAGNI violation)
**C)** Interfaces are slower than direct method calls
**D)** WinChecker should be an abstract class instead of an interface

**Answer: B** — Strategy pattern earns its keep when behavior genuinely varies or might be swapped later. Here the "variation" is really just parameterization (different `(dr,dc)` values), so a single method handles all four cases correctly and more maintainably.
</details>

<details>
<summary>Q3: Why does Board store DiscColor in its grid rather than storing Player references directly?</summary>

**A)** DiscColor uses less memory than a Player object
**B)** Player objects can't be compared with ==
**C)** It keeps Board independently testable — you can test grid/win logic with simple color values instead of needing to construct or mock full Player objects
**D)** Java enums can't be stored in 2D arrays

**Answer: C** — this is a testability-driven design choice, not a technical requirement. Either approach works, but decoupling Board from the Player type keeps its tests simpler.
</details>

<details>
<summary>Q4: In makeMove, why does Game never check if a column is full or out of bounds directly?</summary>

**A)** It's an oversight in the design
**B)** Board.placeDisc() already handles all grid-related validation and returns -1 on failure — Game just checks for -1, keeping grid rules entirely inside Board and game rules (turns, state) entirely inside Game
**C)** Column validation isn't actually needed in Connect Four
**D)** Because Player tracks which columns are valid

**Answer: B** — this is separation of concerns in practice: each class owns validation for the state it's responsible for, and callers don't duplicate checks the callee already performs.
</details>

<details>
<summary>Q5: If asked "how would you add a computer opponent?", what's the strongest answer?</summary>

**A)** Rewrite Player as an interface with HumanPlayer and BotPlayer implementations
**B)** Add an isBot boolean field to Player and branch on it inside makeMove
**C)** Add a separate BotEngine component that picks a column; feed its output into the existing makeMove() unchanged — Game and Board require zero changes
**D)** Create a new BotGame subclass that extends Game

**Answer: C** — the key insight is that game rules don't change at all; a bot is just a different source for the column argument to an already-existing method. This is the cleanest answer because it changes nothing about the core design — it only adds a new component alongside it.
</details>
