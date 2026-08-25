# Tic-Tac-Toe (Game LLD)

## Problem

Design a Tic-Tac-Toe game. Two players take turns. Each player places a symbol (`X` or `O`) on a board. A player wins when they place the same symbol in a full row, a full column, or a full diagonal. If the board fills up and no one wins, the game is a draw.

This looks like a small problem. But interviewers use it to check your object-oriented design skill. They want to see:

- A board that is not hard-coded to 3x3.
- Win and draw detection that runs fast, not by scanning the whole board every turn.
- A clean way to add new player types (human, computer) without changing old code.
- A design that can grow into other board games later.

The goal is not just a working console game. The goal is clean, extensible object design.

## Requirements & Clarifying Questions

**Functional requirements (assumed for this design):**

1. The board size is N x N. Default is 3, but the code must support any N.
2. Exactly two players play one game. Each player has one symbol.
3. Players take turns, picking a cell by row and column.
4. After each move, the game checks: did this move complete a full row, column, or diagonal of the same symbol? If yes, that player wins.
5. If all cells fill and no one won, the game is a draw.
6. A move on an already-filled cell, or outside the board, is invalid. The system must reject it and ask again, without crashing or corrupting game state.
7. Two player types are supported: a **human player** (input from a person) and a **computer player** (input from simple game logic, for example "pick the first empty cell").

**Clarifying questions to ask the interviewer:**

- Is the board always 3x3, or should the design support N x N? (Assumption: N x N, since interviewers usually want this.)
- For N x N, do we need "N in a row," or a smaller count K (Gomoku-style)? (Assumption: full row/column/diagonal of size N. The win-checker is designed so K can be made configurable later.)
- Do we need an undo feature? (Out of scope for the base design, mentioned as an extension.)
- Do we need to persist game state or replay history? (Out of scope. We keep a move list in memory, which makes replay easy to add later.)
- Should the AI player be smart (minimax) or simple? (Assumption: simple for now, pluggable so a smarter one can replace it.)
- Do we need thread-safety for concurrent games? (Assumption: single-threaded per game instance. We note where locks would go for multiplayer-over-network.)

## Design / Approach

### Key design decisions

1. **`Symbol` is an enum**, not a raw `char`. It holds `X`, `O`, and `EMPTY`. This avoids magic characters and keeps the board type-safe.

2. **`Cell` is a small value holder.** It stores a `Symbol` and knows if it is empty. It does not know about rows, columns, or win rules — **Single Responsibility Principle** (SRP).

3. **`Board` owns the grid and does bookkeeping for win detection.** Instead of scanning the full board after every move, it keeps running counters: one per row, one per column, and two for the diagonals. Each move updates only the counters it touches — O(1) per move. This is the key efficiency point interviewers look for.

4. **`Player` is an interface with one method: `makeMove(Board board)`.** `HumanPlayer` reads console input. `ComputerPlayer` picks a cell using its own logic. This is the **Strategy Pattern** — `Game` does not care how a player decides a move. It follows **Open/Closed Principle** (OCP): a smarter AI later means a new class, not edits to `Game` or `Board`.

5. **`WinningStrategy` is a separate interface**, injected into `Game`. The default is `LineWinStrategy` (row/column/diagonal of size N). This is **Dependency Inversion Principle** (DIP): `Board`'s logic does not hard-code the rule. Swapping in "5-in-a-row" or Connect Four's "4-in-a-row, gravity-drop" needs no change to `Board`'s grid logic.

6. **`Game` is the controller (Facade).** It owns the `Board`, the `Player`s, and turn order. It runs the loop: ask for a move, validate, apply, check win/draw, switch turn. Client code only calls `game.play()`. This is SRP plus **Facade Pattern**: `Game` orchestrates, it does not validate cells or decide moves itself.

7. **Input validation lives in `Board.placeSymbol(...)`.** It throws `InvalidMoveException` for out-of-range or already-taken cells. `Game` catches this and asks the same player again — validation stays in one place instead of scattered checks.

8. **`Move` is a small immutable class** (row, column, symbol). Storing moves in a list gives a free move history, useful for undo/replay later, with no change to existing classes.

9. **A `GameResult`** (status enum plus a `winner` field) keeps the "what happened" state explicit, instead of null checks or magic booleans.

### Class relationships (ASCII sketch)

```
                    +----------------------+
                    |      Symbol          |  <<enum>>  X, O, EMPTY
                    +----------------------+

  +--------------+        contains N*N        +----------------+
  |    Board     |---------------------------->|      Cell      |
  +--------------+                             +----------------+
  | grid: Cell[][]|                            | symbol: Symbol |
  | rowCount[]   |                             | isEmpty()      |
  | colCount[]   |                             +----------------+
  | diagCount    |
  | antiDiagCount|        uses to check win     +--------------------------+
  | placeSymbol()|----------------------------->|   WinningStrategy        | <<interface>>
  | isFull()     |                              +--------------------------+
  +------+-------+                              | checkWin(board, move)   |
         ^                                      +-----------^--------------+
         | owns                                             |
         |                                       +-----------+-----------+
  +------+-------+                               |    LineWinStrategy    |
  |    Game      |                               | (row/col/diag == N)   |
  | (Controller, |                               +------------------------+
  |   Facade)    |
  +--------------+        asks for a move         +------------------+
  | board        |------------------------------->|     Player       | <<interface>>
  | players[2]   |                                +------------------+
  | currentTurn  |                                | makeMove(board) |
  | moveHistory  |                                +--------^---------+
  | play()       |                                         |
  +--------------+                          +---------------+---------------+
                                             |                               |
                                    +--------+--------+           +---------+---------+
                                    |  HumanPlayer    |           |  ComputerPlayer   |
                                    | (console input) |           | (simple strategy) |
                                    +------------------+           +--------------------+

  +--------------+
  |    Move      |   (row, col, symbol, player) — stored in Game.moveHistory
  +--------------+
```

**Patterns used:** Strategy (`Player`, `WinningStrategy`), Facade (`Game`). SOLID: SRP (`Cell`, `Board`, `Game` each have one job), OCP (new player types and win rules plug in without editing existing classes), DIP (`Game` and `Board` depend on interfaces, not concrete classes), LSP (any `Player` or `WinningStrategy` implementation can replace another safely).

## Java Solution

```java
import java.util.*;

// ---------- Symbol ----------
enum Symbol {
    X, O, EMPTY;

    public boolean isEmpty() {
        return this == EMPTY;
    }
}

// ---------- Cell ----------
final class Cell {
    private Symbol symbol;

    Cell() {
        this.symbol = Symbol.EMPTY;
    }

    Symbol getSymbol() {
        return symbol;
    }

    void setSymbol(Symbol symbol) {
        this.symbol = symbol;
    }

    boolean isEmpty() {
        return symbol.isEmpty();
    }
}

// ---------- Exceptions ----------
class InvalidMoveException extends RuntimeException {
    InvalidMoveException(String message) {
        super(message);
    }
}

// ---------- Move (history entry) ----------
final class Move {
    final int row;
    final int col;
    final Symbol symbol;

    Move(int row, int col, Symbol symbol) {
        this.row = row;
        this.col = col;
        this.symbol = symbol;
    }

    @Override
    public String toString() {
        return symbol + " -> (" + row + "," + col + ")";
    }
}

// ---------- WinningStrategy (Strategy Pattern) ----------
interface WinningStrategy {
    // Checks if the move at (row, col) completed a win. Must run in O(1).
    boolean checkWin(Board board, int row, int col, Symbol symbol);
}

// Default rule: a full row, column, or diagonal of the same symbol wins.
class LineWinStrategy implements WinningStrategy {
    @Override
    public boolean checkWin(Board board, int row, int col, Symbol symbol) {
        int n = board.getSize();
        return board.getRowCount(row, symbol) == n
                || board.getColCount(col, symbol) == n
                || (isOnMainDiagonal(row, col) && board.getDiagCount(symbol) == n)
                || (isOnAntiDiagonal(row, col, n) && board.getAntiDiagCount(symbol) == n);
    }

    private boolean isOnMainDiagonal(int row, int col) {
        return row == col;
    }

    private boolean isOnAntiDiagonal(int row, int col, int n) {
        return row + col == n - 1;
    }
}

// ---------- Board ----------
class Board {
    private final int size;
    private final Cell[][] grid;
    private int filledCells;

    // Running counters for O(1) win checks instead of rescanning the board.
    private final Map<Symbol, int[]> rowCount = new EnumMap<>(Symbol.class);
    private final Map<Symbol, int[]> colCount = new EnumMap<>(Symbol.class);
    private final Map<Symbol, Integer> diagCount = new EnumMap<>(Symbol.class);
    private final Map<Symbol, Integer> antiDiagCount = new EnumMap<>(Symbol.class);

    Board(int size) {
        if (size < 3) {
            throw new IllegalArgumentException("Board size must be at least 3");
        }
        this.size = size;
        this.grid = new Cell[size][size];
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                grid[r][c] = new Cell();
            }
        }
        for (Symbol s : new Symbol[]{Symbol.X, Symbol.O}) {
            rowCount.put(s, new int[size]);
            colCount.put(s, new int[size]);
            diagCount.put(s, 0);
            antiDiagCount.put(s, 0);
        }
    }

    int getSize() {
        return size;
    }

    boolean isFull() {
        return filledCells == size * size;
    }

    // Places a symbol on the board; throws if invalid. Updates counters used for fast win checks.
    void placeSymbol(int row, int col, Symbol symbol) {
        validateCoordinates(row, col);
        if (!grid[row][col].isEmpty()) {
            throw new InvalidMoveException("Cell (" + row + "," + col + ") is already taken");
        }
        grid[row][col].setSymbol(symbol);
        filledCells++;

        rowCount.get(symbol)[row]++;
        colCount.get(symbol)[col]++;
        if (row == col) {
            diagCount.put(symbol, diagCount.get(symbol) + 1);
        }
        if (row + col == size - 1) {
            antiDiagCount.put(symbol, antiDiagCount.get(symbol) + 1);
        }
    }

    void validateCoordinates(int row, int col) {
        if (row < 0 || row >= size || col < 0 || col >= size) {
            throw new InvalidMoveException(
                    "Cell (" + row + "," + col + ") is out of range for a " + size + "x" + size + " board");
        }
    }

    boolean isCellEmpty(int row, int col) {
        return grid[row][col].isEmpty();
    }

    int getRowCount(int row, Symbol symbol) {
        return rowCount.get(symbol)[row];
    }

    int getColCount(int col, Symbol symbol) {
        return colCount.get(symbol)[col];
    }

    int getDiagCount(Symbol symbol) {
        return diagCount.get(symbol);
    }

    int getAntiDiagCount(Symbol symbol) {
        return antiDiagCount.get(symbol);
    }

    void print() {
        for (Cell[] row : grid) {
            StringBuilder sb = new StringBuilder();
            for (Cell cell : row) {
                sb.append(cell.isEmpty() ? "." : cell.getSymbol()).append(" ");
            }
            System.out.println(sb.toString().trim());
        }
    }
}

// ---------- Player (Strategy Pattern) ----------
interface Player {
    String getName();

    Symbol getSymbol();

    // Returns the next move as {row, col}. Does not mutate the board; Game applies the move after validating it.
    int[] makeMove(Board board);
}

class HumanPlayer implements Player {
    private final String name;
    private final Symbol symbol;
    private final Scanner scanner;

    HumanPlayer(String name, Symbol symbol, Scanner scanner) {
        this.name = name;
        this.symbol = symbol;
        this.scanner = scanner;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public Symbol getSymbol() {
        return symbol;
    }

    @Override
    public int[] makeMove(Board board) {
        System.out.println(name + " (" + symbol + "), enter row and col (space separated): ");
        int row = scanner.nextInt();
        int col = scanner.nextInt();
        return new int[]{row, col};
    }
}

// Simple AI: picks the first empty cell. A smarter one can replace this without touching Game or Board.
class ComputerPlayer implements Player {
    private final String name;
    private final Symbol symbol;

    ComputerPlayer(String name, Symbol symbol) {
        this.name = name;
        this.symbol = symbol;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public Symbol getSymbol() {
        return symbol;
    }

    @Override
    public int[] makeMove(Board board) {
        int n = board.getSize();
        for (int r = 0; r < n; r++) {
            for (int c = 0; c < n; c++) {
                if (board.isCellEmpty(r, c)) {
                    return new int[]{r, c};
                }
            }
        }
        throw new IllegalStateException("No empty cell left; board is full");
    }
}

// ---------- Game result ----------
enum GameStatus {
    IN_PROGRESS, WIN, DRAW
}

final class GameResult {
    final GameStatus status;
    final Player winner; // null if DRAW or IN_PROGRESS

    GameResult(GameStatus status, Player winner) {
        this.status = status;
        this.winner = winner;
    }
}

// ---------- Game (Controller / Facade) ----------
class Game {
    private final Board board;
    private final List<Player> players;
    private final WinningStrategy winningStrategy;
    private final List<Move> moveHistory = new ArrayList<>();
    private int currentPlayerIndex = 0;

    Game(Board board, List<Player> players, WinningStrategy winningStrategy) {
        if (players.size() != 2) {
            throw new IllegalArgumentException("Tic-Tac-Toe needs exactly 2 players");
        }
        this.board = board;
        this.players = players;
        this.winningStrategy = winningStrategy;
    }

    GameResult play() {
        while (true) {
            Player current = players.get(currentPlayerIndex);
            int row, col;

            // Retry loop: invalid moves do not crash the game or skip a turn.
            try {
                int[] move = current.makeMove(board);
                row = move[0];
                col = move[1];
                board.placeSymbol(row, col, current.getSymbol());
            } catch (InvalidMoveException e) {
                System.out.println("Invalid move: " + e.getMessage() + ". Try again.");
                continue;
            }

            moveHistory.add(new Move(row, col, current.getSymbol()));
            board.print();

            if (winningStrategy.checkWin(board, row, col, current.getSymbol())) {
                return new GameResult(GameStatus.WIN, current);
            }
            if (board.isFull()) {
                return new GameResult(GameStatus.DRAW, null);
            }
            currentPlayerIndex = 1 - currentPlayerIndex;
        }
    }

    List<Move> getMoveHistory() {
        return Collections.unmodifiableList(moveHistory);
    }
}

// ---------- Demo ----------
public class TicTacToeDemo {
    public static void main(String[] args) {
        Board board = new Board(3);
        Player p1 = new HumanPlayer("Alice", Symbol.X, new Scanner(System.in));
        Player p2 = new ComputerPlayer("Bot", Symbol.O);

        Game game = new Game(board, Arrays.asList(p1, p2), new LineWinStrategy());
        GameResult result = game.play();

        if (result.status == GameStatus.WIN) {
            System.out.println("Winner: " + result.winner.getName());
        } else {
            System.out.println("It's a draw!");
        }
    }
}
```

## How It Works

1. **Setup.** `Game` is built with a `Board`, two `Player` objects, and a `WinningStrategy` (constructor injection). A unit test can pass fake `Player`s that return a fixed sequence of moves.

2. **Turn loop.** `Game.play()` asks the current player for a move via `makeMove(board)`. `HumanPlayer` reads console input; `ComputerPlayer` scans for the first empty cell. Neither touches the board directly — they only *decide*, `Game` applies.

3. **Validation happens in one place.** `Board.placeSymbol` checks range, then checks if the cell is filled, throwing `InvalidMoveException` on failure. `Game` catches this and asks the *same* player again — a bad input never silently skips a turn.

4. **Win check is O(1) per move.** `Board` keeps four counters per symbol: row, column, diagonal, anti-diagonal. Each `placeSymbol` call updates at most 4 of them. `LineWinStrategy.checkWin` just reads the counters touched by the last move — an O(N²) scan becomes O(1).

5. **Game ends** when `checkWin` returns true (`GameResult` with a winner) or `board.isFull()` (`GameResult` for a draw). Both return the same type, so callers never guess the outcome from null checks.

6. **Move history** is an immutable list of `Move` objects — a side benefit that makes undo/replay easy to add later, with no change to existing code.

## How to Extend (Follow-ups)

Interviewers almost always ask: "How would you change this for game X?" The answer should be: swap one or two components; the rest does not change.

- **K-in-a-row (Gomoku-style)**: Change `LineWinStrategy` to check K consecutive symbols in any direction, scanning outward from the last move. `Board`, `Game`, `Player` do not change — only a new `WinningStrategy`.

- **Connect Four**: Replace `placeSymbol` with a gravity drop (`dropInColumn`), plus a new `WinningStrategy` for 4-in-a-row. `Player`, `Game`, `GameResult` stay the same, since the turn loop does not depend on how a move lands.

- **Snake & Ladder**: A bigger structural change (a single track of N cells, moves from a dice roll, not a choice), but the same shapes carry over: `Player` (human vs. computer roll), `Game` as the turn-loop controller, a `Board` owning cells, and a `GameResult`. The reusable part is the *pattern* (Strategy, Facade), not the grid code.

- **Chess**: A larger board with special piece rules still fits the skeleton: `Cell` holds a `Piece` instead of a `Symbol`, `Player` picks a move, `Game` runs turns and asks a `WinningStrategy` (checkmate). Identify the stable parts (turn loop, validation, result reporting) versus variable parts (what a move or a cell means), and put interfaces at that boundary.

- **N players**: Change turn switching from `1 - currentPlayerIndex` to `(currentPlayerIndex + 1) % players.size()`, and loosen the 2-player constructor check.

- **Undo move**: Add `undo()` to `Game` that pops the last `Move` from `moveHistory`, clears that `Cell`, and decrements the matching counters.

- **Smarter AI**: Add a `MinimaxPlayer implements Player`. Because `Player` is an interface, `Game` needs zero changes.

## Complexity & Thread-Safety Notes

**Time complexity:**
- `placeSymbol`: O(1) — updates a fixed number of counters, independent of board size.
- `checkWin` (via `LineWinStrategy`): O(1) — reads pre-computed counters, does not scan the grid.
- `isFull`: O(1) — a running counter (`filledCells`), not a grid scan.
- Total game: at most N² moves, each O(1), so a full game runs in O(N²) — optimal, since every cell is visited at most once.

**Space complexity:** O(N²) for the grid, plus O(N) for the row/column counters per symbol, plus O(1) for the two diagonal counters per symbol. Move history adds O(N²) in the worst case.

**Thread-safety:** This design is single-threaded — one `Game` runs one game to completion in one thread, which fits a console app or a single request/response API. For concurrent access (a server running many simultaneous games, or two clients racing to move):

- Put one lock around `Game.play()`'s move-apply-check sequence, so "validate, apply, check win, switch turn" is atomic. Without it, two threads could both pass the "is it my turn" check before either applies a move.
- Guard `Board`'s counters with the same lock, since the grid cell, row counter, column counter, and diagonal counter must update together — one `AtomicInteger` per field is not enough.
- Keep one `Game` object per match; never share a `Board` across two unrelated games.

## Interview Tips & Common Mistakes

- **Do not hard-code 3x3.** A common mistake is writing `char[3][3] board` with literal loops `for i in 0..2`. Parameterize by `size` from the start.

- **Do not rescan the whole board to check a win after every move.** This is the most common inefficiency here. Explain the O(1) counter approach even if you run out of time to code every branch.

- **Separate "decide a move" from "apply a move."** Do not let `Player` write directly into the `Board`. Keep `Player.makeMove()` returning a proposed move; let `Game`/`Board` validate and apply it, keeping `Player` swappable and testable.

- **Use an enum for `Symbol`, not `char` or `String`.** Raw characters lead to typos and a fragile empty-cell check.

- **Name the Strategy pattern for `Player` and `WinningStrategy`,** and explain why it helps: Open/Closed Principle — new player types or win rules plug in without editing existing, tested code.

- **Handle invalid input without crashing or skipping a turn.** Throw from `Board`, catch in `Game`'s loop, instead of letting `main` crash or silently ignoring bad input.

- **Do not put win-checking logic inside `Player`.** A player's only job is to propose a move; whether it wins is the board/game's job.

- **Mention thread-safety even if you do not fully implement it.** State what is and is not thread-safe, and the minimal fix (one lock around the move-apply-check sequence).

- **Keep `GameResult` explicit** (a status enum plus a nullable winner) instead of overloading `null` for "no winner yet" — mixing return meanings is a common code smell.
