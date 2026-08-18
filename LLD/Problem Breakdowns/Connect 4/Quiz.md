> [!TIP] 
>Why is a GameState enum preferred over boolean flags like isOver and hasWinner for tracking game state?
> * A) Enums use less memory
> * B) Enums are faster to evaluate
> * C) Enums support serialization better
> * D) Booleans allow impossible states
> 
> <details><summary><b>Answer</b></summary><b>D) .  </b> With two booleans, you get 4 combinations but only 3 valid states (IN_PROGRESS, WON, DRAW). The impossible combination isOver=false + hasWinner=true can't be represented with an enum, preventing bugs from invalid state.</details>

> [!TIP] 
>Naive minimax scores a RED win as +1, no winner as 0, and a YELLOW win as −1. RED is the maximizer. Why does RED block an immediate YELLOW win when it can??
> * A) Allowing the win makes that branch −1, while a block can leave a 0 that RED prefers.
> * B) Blocking changes YELLOW into the maximizer, making its replies score below every RED move.
> * C)   
The blocking move is scored +1 immediately, even though RED has not completed four in a row.
> * D) Minimax skips every branch where the opponent has a winning reply, so those leaves never receive scores.
> 
> <details><summary><b>Answer</b></summary><b>A) .  </b> RED chooses the highest available score. A branch where YELLOW wins is −1, so RED prefers a blocking branch that stays at 0; if RED can force its own win instead, +1 is better still..</details>

> [!TIP] 
>In the placeDisc method, how does the Board determine where a disc lands in a column?
> * A) It checks the middle row first, then expands outward
> * B) It places the disc at the top row
> * C) It scans upward from the bottom to find the first empty cell
> * D) It uses a random empty cell
> 
> <details><summary><b>Answer</b></summary><b>C) .  </b> Gravity means discs fall to the lowest available position. The method scans from the bottom row upward and places the disc in the first empty cell it finds, mimicking how a physical Connect Four board works.</details>

> [!TIP] 
>**Storing simple enum values in a grid instead of full entity objects improves testability by reducing dependencies.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) .  </b> When a grid stores lightweight values (like a color enum) instead of complex entity objects, it can be tested in isolation without constructing those entities. This reduces coupling between the grid logic and the entity classes.</details>

> [!TIP] 
>**In a turn-based game, why should the orchestrator validate game state before delegating to the board?**
> * A) The board would throw unchecked exceptions
> * B) It prevents meaningless operations on a game that's already over
> * C) State validation requires database access
> * D) The board doesn't have access to state
> 
> <details><summary><b>Answer</b></summary><b>B) .  </b> If the game is already won or drawn, there's no point placing another disc. The orchestrator should reject the move early based on game state, before involving the board. This keeps the board focused on grid logic..</details>

> [!TIP] 
>**Why should the Game class (orchestrator) handle turn switching rather than the Board?**
> * A) Player objects with game logic cause memory leaks in long-running applications
> * B) Players with methods are harder for the garbage collector to clean up efficiently
> * C) Minimal classes are required by the Strategy pattern that turn-based games typically use
> * D) It keeps game rules centralized in the orchestrator
> 
> <details><summary><b>Answer</b></summary><b>D) .  </b> A thin Player data holder keeps game rules in the orchestrator (Game class) where they belong. If Player had methods like makeMove() or checkWin(), game logic would be scattered across multiple classes, making it harder to understand and modify the rules..</details>

> [!TIP] 
>Why does the Board validate column bounds rather than the Game?
> * A) The Game is responsible for player validation only
> * B) The Game doesn't know the board dimensions
> * C) The Board owns the grid, so it validates grid constraints
> * D) Column validation requires checking disc colors
> 
> <details><summary><b>Answer</b></summary><b>C) .  </b> Encapsulation means each class validates its own data. The Board owns the grid, so it's responsible for checking whether a column index is valid and whether the column has space. The Game handles game-level rules like turn order..</details>

> [!TIP] 
>In the article's array-backed Board, which optimization makes disc placement O(1) instead of scanning a column?
> * A) Reverse the direction of the column scan
> * B) Track each column's next free row in a heights array
> * C) Track only the most recent move's row
> * D) Sort each column by disc color
> 
> <details><summary><b>Answer</b></summary><b>B) .  </b> The article suggests a heights array with one entry per column, where each entry points to that column's next free row. Reading and updating that entry makes placement O(1); remembering one global move, reversing the scan, or sorting discs does not identify every column's next open cell.</details>