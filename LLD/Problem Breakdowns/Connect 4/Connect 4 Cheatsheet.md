# Connect Four LLD Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Default Requirements
| Question | Answer |
|---|---|
| Board size | 7 columns × 6 rows, fixed (not configurable) |
| Move | Player picks column → disc falls to lowest open row |
| Win | 4 in a row — horizontal, vertical, or diagonal |
| Draw | Board full, no winner |
| Invalid moves | Full column, out-of-turn, move-after-game-over → reject (return false) |
| Scope | Single game, backend only — no UI, no concurrency, no undo, no move history |

## Entities
| Entity | Responsibility |
|---|---|
| `Game` | Orchestrator — turns, state, win-checking delegation |
| `Board` | Grid state, placement, win detection |
| `Player` | Pure data — name + color, zero logic |

## Class Diagram (compressed)
```
Game ──has──> Board
Game ──has──> Player (x2, + currentPlayer ref)
Game.state: GameState (IN_PROGRESS | WON | DRAW)
Board.grid: DiscColor[6][7]
```

## Key Design Decisions & One-Line Justifications
| Decision | Why |
|---|---|
| `GameState` enum, not 3 booleans | Makes invalid states (e.g. won+draw both true) unrepresentable — type system enforces correctness |
| `winner` stays a separate nullable field | Java/pseudocode can't attach data to enum variants elegantly (Rust/Swift/Kotlin could) — practical tradeoff, mention the ideal if asked |
| `Board` stores `DiscColor`, not `Player` | Keeps Board independently testable — simpler value type, no mocking needed |
| `checkWin` uses direction vectors `(dr,dc)` + one shared counting method | Avoids 4 separate WinChecker classes — the "variation" is just parameters, not real behavioral difference (YAGNI) |
| Grid validation lives entirely in `Board.placeDisc()` | Game only checks for `-1` — keeps grid rules and game rules cleanly separated |
| `Player` is pure data, no interface | A human doesn't "do" anything — making Player polymorphic adds abstraction with no value |

## The Over-Engineering Trap (know this cold)
❌ **Don't:** `WinChecker` interface + `HorizontalWinChecker`/`VerticalWinChecker`/`DiagonalUpWinChecker`/`DiagonalDownWinChecker` — looks OOP, but win directions are fixed forever, all four "checkers" are literally the same algorithm, and Strategy pattern is for genuine runtime-swappable variation, not fixed geometry.
✅ **Do:** One `checkWin()` + one `countInDirection(row, col, dr, dc, color)` helper, called with 4 direction pairs: `(0,1)` horizontal, `(1,0)` vertical, `(1,1)` and `(-1,1)` diagonals.

## Method Signatures (Java)
```java
class Game {
    boolean makeMove(Player player, int column);
    Player getCurrentPlayer();
    GameState getGameState();
    Player getWinner();
    Board getBoard();
}

class Board {
    boolean canPlace(int column);
    int placeDisc(int column, DiscColor color);  // returns row, or -1
    boolean isFull();
    boolean checkWin(int row, int column, DiscColor color);
    DiscColor getCell(int row, int column);
}

class Player {
    String getName();
    DiscColor getColor();
}
```

## makeMove — Order of Operations
1. Reject if `state != IN_PROGRESS`
2. Reject if wrong player's turn
3. `board.placeDisc()` → reject if `-1`
4. `board.checkWin()` → set `WON` + `winner`
5. else `board.isFull()` → set `DRAW`
6. else switch `currentPlayer`

## Extensibility Follow-Ups
| Ask | Answer shape |
|---|---|
| Configurable board size | `rows`/`cols` become `Board` constructor params — logic already generic since it never hardcodes numbers |
| Undo / move history | `moveHistory` stack in `Game` + `Move(player,row,col)` value object + `Board.clearCell()` — single choke point (`makeMove`) makes this easy |
| Computer opponent | New standalone `BotEngine.chooseMove(game)` — feeds into existing `makeMove()` unchanged; **Game/Board require zero changes** |

## Level Expectations
- **Junior:** working game, correct horiz/vert win check, basic error handling (false on invalid)
- **Mid:** clean separation unprompted, direction-vector win-check (not 4 classes), 1 extensibility discussion
- **Senior:** justifies decisions proactively (enum vs. booleans, win-check placement), catches own edge cases, multiple extensibility tradeoffs unprompted
