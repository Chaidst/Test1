# Playing Tic-Tac-Toe over the Internet

## Protocol Overview

This protocol defines a client–server application with a concurrent game server for playing Tic-Tac-Toe over TCP. The server hosts up to 5 game rooms, each supporting 2 players. Clients connect, join a room, and take turns marking positions on a 3x3 board. 

Board positions are numbered left-to-right, top-to-bottom:

```
 1 | 2 | 3
-----------
 4 | 5 | 6
-----------
 7 | 8 | 9
```

The design prioritizes simplicity, every message is a single line terminated by `\n`, with a command keyword followed by optional arguments separated by spaces making the game easy to play. The server is the authority on all game state — it validates moves, tracks turns, detects wins/draws, and manages player statistics.

## Protocol Design Requirements

- **How clients request to join rooms**: A client sends `JOIN y` where `y` is a room number from 1 to 5. This command is used both when first connecting and when choosing to play again after a game ends.

- **How the server confirms or rejects room joins**: If the room has space, the server responds with `JOINED_WAIT` (first player, waiting for opponent) or `JOINED` (second player, game about to start). If the room already has 2 players, the server responds with `ROOM_FULL` and the client can try a different room.

- **How game start, turns, and board updates are communicated**: When both players have joined, the server sends `STARTED X` or `STARTED O` to each client so they know their symbol. The server then sends `YOUR_TURN` to the active player. After a valid move, the server sends the updated board to both players as `BOARD <cells>`, a 9-character string representing the 3x3 grid. Turns alternate until someone wins or the board fills up.

- **How invalid actions are reported**: If a client sends a bad command, an out-of-range position, or tries to mark an already-occupied cell, the server responds with `INVALID`. The client is then expected to retry. During a game, only `MARK` commands are accepted — any other command is treated as invalid.

- **How game termination and disconnection are handled**: When a player wins, the server sends `WIN` to the winner and `LOSE` to the loser. If the board fills with no winner, both receive `DRAW`. If a client disconnects or is inactive for more than 15 seconds during their turn, the server closes their socket without sending statistics and notifies the opponent with `OPPONENT_LEFT`, counting it as a win for them.

- **How cross-game statistics are tracked**: Each client connection maintains a running count of games played, won, and drawn. These stats persist across multiple rounds within the same TCP session, so a client who plays 3 games and then quits will see their cumulative results.

- **How post-game statistics are transmitted**: When a client sends `QUIT`, the server responds with `STATS <won> <drawn> <lost>` containing their session totals, then closes the connection. Statistics are not sent to clients who disconnect or time out mid-game.

## Connection Model

The server listens on a TCP port provided as a terminal argument. When a client connects, the server spawns a dedicated thread to handle that client for the duration of their session. This means the server can handle multiple clients concurrently.

The server pre-creates 5 game rooms at startup (rooms 1-5). Clients join a room by sending `JOIN y` where y is 1-5. The first client to join a room becomes Player 1 and is assigned `X`. The second client becomes Player 2 and is assigned `O`. Player 1 always moves first. If a room already has 2 players, the server rejects the join with `ROOM_FULL`.

After a game ends, both clients are prompted to play again. They can rejoin the same or a different room, and their statistics carry over across rounds within the same connection.

## Messages

All messages are single lines terminated by a newline character (`\n`).

### Client → Server

```
| Message | Format | Description |
|---------|--------|-------------|

| JOIN | `JOIN y` | Request to join a room. `y` is an integer from 1 to 5. Sent when the client first connects or after a game ends. |
| MARK | `MARK x` | Place a mark at the given board position. `x` is an integer from 1 to 9 (see board layout below). Only valid during the client's turn. |
| QUIT | `QUIT` | Disconnect from the server. The server responds with the client's game statistics before closing the connection. |
```


### Server → Client

```
| Message | Format | Description |
|---------|--------|-------------|

| PROMPT_JOIN | `PROMPT_JOIN` | Sent when a client first connects, prompting them to join a room. |
| JOINED_WAIT | `JOINED_WAIT` | Confirms the client joined a room and is waiting for a second player. |
| JOINED | `JOINED` | Confirms the client joined a room that already had one player (game will start). |
| ROOM_FULL | `ROOM_FULL` | The requested room already has 2 players. The client should try a different room. |
| STARTED | `STARTED <symbol>` | The game has started. `symbol` is `X` or `O`, telling the client which mark they play as. `X` moves first. |
| YOUR_TURN | `YOUR_TURN` | It is this client's turn to make a move. The client should respond with a `MARK` command. |
| BOARD | `BOARD <cells>` | A 9-character string representing the current board state, read left-to-right, top-to-bottom. Each character is `X`, `O`, or a digit `1`–`9` for empty cells. Sent after each valid move. |
| INVALID | `INVALID` | The client's last action was invalid (bad command, occupied position, out-of-range position, etc.). The client should try again. |
| WIN | `WIN` | The game is over and this client won. |
| LOSE | `LOSE` | The game is over and this client lost. |
| DRAW | `DRAW` | The game is over in a draw. |
| OPPONENT_LEFT | `OPPONENT_LEFT` | The opponent disconnected or timed out. This client wins by default. |
| PLAY_AGAIN | `PLAY_AGAIN` | Sent after a game ends, prompting the client to join another room or quit. |
| STATS | `STATS <won> <drawn> <lost>` | The client's game statistics for this session. Sent in response to `QUIT`. |
```

## Game State Management

The server maintains a `GameRoom` struct for each of the 5 rooms. Each room tracks:

- Pointers to the two player connections (or `NULL` if a slot is empty)
- A `TicTacToe` board struct (3x3 grid, current player, turn count)
- The number of active players in the room
- Whose turn it is (player 1 or player 2)
- Whether a game is currently active
- The game result (which player won, draw, or disconnect)

Access to room data is synchronized using a mutex lock (`pthread_mutex_t`) since multiple threads may access the same room. A condition variable is also used so that threads can efficiently wait for their turn.

Each client connection has a `PlayerConnection` struct that tracks the socket and cumulative statistics (games played, won, drawn). These stats persist across multiple rounds within the same TCP connection.

When a game ends, both players are removed from the room and the room is reset if no players remain. This allows new clients to join the room for a fresh game.

## Error Handling and Timeouts

**Invalid commands**: If a client sends an unrecognized command, a `MARK` with an out-of-range position (not 1–9), or tries to mark an already-occupied cell, the server responds with `INVALID` and waits for the client to retry.

**Mid-game restrictions**: During a game, only `MARK` commands are accepted. If a client sends `JOIN` or any other command during their turn, the server responds with `INVALID`.

**Inactivity timeout**: When it is a client's turn, the server sets a 15-second receive timeout on their socket. If the client does not send a valid move within 15 seconds, the server treats this the same as a disconnection — it closes that client's socket and does not send them any statistics. The opponent is notified with `OPPONENT_LEFT` and the game counts as a win for them.

**Client disconnection**: If a client disconnects mid-game (TCP connection drops), the server detects this when `recv` returns 0 or an error. The server closes the socket, does not send statistics to the disconnected client, and notifies the opponent with `OPPONENT_LEFT`. The game counts as a win for the remaining player.

**Room full**: If a client tries to join a room that already has 2 players, the server responds with `ROOM_FULL` and the client can try joining a different room.

## Time Diagram

The following diagram shows a typical game flow between two clients (C1, C2) and the server (S):

```
  C1                        S                        C2
  |                         |                         |
  |------- connect -------->|                         |
  |<----- PROMPT_JOIN ------|                         |
  |------- JOIN 1 --------->|                         |
  |<----- JOINED_WAIT ------|                         |
  |                         |                         |
  |                         |<------- connect --------|
  |                         |------ PROMPT_JOIN ----->|
  |                         |<------- JOIN 1 ---------|
  |                         |-------- JOINED -------->|
  |                         |                         |
  |<----- STARTED X --------|------- STARTED O ------>|
  |                         |                         |
  |<----- YOUR_TURN --------|                         |
  |------- MARK 5 --------->|                         |
  |<------ BOARD ---------- |-------- BOARD --------->|
  |                         |                         |
  |                         |------ YOUR_TURN ------->|
  |                         |<------- MARK 1 ---------|
  |<------ BOARD ---------- |-------- BOARD --------->|
  |                         |                         |
  |    ... turns will alternate until game ends ...   |
  |                         |                         |
  |<------- BOARD ----------|-------- BOARD --------->|
  |<------- WIN ------------|-------- LOSE ---------->|
  |                         |                         |
  |<---- PLAY_AGAIN --------|------ PLAY_AGAIN ------>|
  |------- QUIT ----------->|                         |
  |<------- STATS ---------|                         |
  |                         |<------- QUIT -----------|
  |                         |-------- STATS --------->|
```
