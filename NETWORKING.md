# Networking: `script.js` <-> `main.cpp`

This project uses a single WebSocket connection for real-time chat.

## Overview
- Frontend client: `script.js`
- Backend server: `main.cpp` (`cpp-httplib` WebSocket endpoint)
- WebSocket route: `/ws`
- Server listen port: `8080`

`index.html` loads `script.js`, which opens a WebSocket to:
- `ws://localhost:8080/ws` (or `wss://...` if page is served over HTTPS)

## Connection Lifecycle
1. `script.js` reads `chat_name` and optional `chat_color` from `localStorage`.
2. It opens `new WebSocket(.../ws)`.
3. On open, it sends login command:
   - `&u{"username":"<name>","color":"#RRGGBB"}`
4. Server validates username uniqueness and replies:
   - `{"event":"uname-eval","result":"ok"}` or
   - `{"event":"uname-eval","result":"taken"}`
5. If accepted:
   - Server sends stored message history (`event:"msg"` payloads).
   - Server broadcasts `userjoin` to all clients.
   - Client then requests active users list with `&i`.

## Client -> Server Message Types
Non-JSON command frames are prefixed with `&`:

- `&u...`  
  Login / set username (and optionally color).

- `&i`  
  Request current users (`sendusers` response).

- `&t`  
  User started typing.

- `&s`  
  User stopped typing.

Plain text frame (no `&` prefix):
- Treated as a chat message and broadcast as `event:"msg"`.

## Server -> Client Event Payloads (JSON)

- `uname-eval`
  - Example: `{"event":"uname-eval","result":"ok"}`
  - `result` is `ok` or `taken`.

- `sendusers`
  - Example:
    `{"event":"sendusers","users":{"123":{"username":"a","color":"#83c092"}}}`

- `userjoin`
  - Example:
    `{"event":"userjoin","id":"123","username":"alice","color":"#83c092","timestamp":...}`

- `userleft`
  - Example:
    `{"event":"userleft","id":"123"}`

- `msg`
  - Example:
    `{"event":"msg","id":"123","username":"alice","color":"#83c092","timestamp":...,"msg":"hello"}`

- `typing`
  - Example: `{"event":"typing","id":"123"}`

- `stoptyping`
  - Example: `{"event":"stoptyping","id":"123"}`

## State Kept on Server

`main.cpp` stores:
- `clients` map: connection pointer, username, color
- `message_history` deque: recent message payloads (`MAX_HISTORY = 256`)

On disconnect:
- server removes the client
- broadcasts `userleft`
- stores a `" left."` message in history

## State Kept on Client

`script.js` tracks:
- `users` map (id -> username/color)
- `users_typing` list
- `username_ok` gate before processing normal events
- typing debounce timer (`typing_timer = 500ms`)

Typing sync:
- local key activity sets `typing=true`
- periodic interval compares `typing` vs `typing_t`
- sends `&t` or `&s` when state changes

## Failure Behavior
- If `uname-eval` is `taken`, client clears `chat_name` and redirects to `name.html`.
- WebSocket `error` currently posts a red chat message in UI.

