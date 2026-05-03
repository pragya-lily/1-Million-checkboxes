[README.md](https://github.com/user-attachments/files/27317705/README.md)
# ☑ Million Checkboxes

A real-time collaborative web app where **1,000,000 checkboxes** are shared across all connected users. Toggle any checkbox and every other user sees the change instantly — powered by WebSockets, Redis Pub/Sub, and OAuth 2.0 / OIDC authentication.

---

##  Screenshots / Demo

> Open two browser windows side-by-side and toggle checkboxes — watch them update in real time.

---

##  Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vanilla HTML, CSS, JavaScript (Virtual Scroll) |
| Backend | Node.js + Express |
| Real-time | WebSockets (`ws` library) |
| State Store | Redis (BITFIELD — 1M bits = 125 KB) |
| Pub/Sub | Redis Pub/Sub (multi-server broadcast) |
| Auth | OIDC / OAuth 2.0 — Authorization Code Flow |
| Rate Limiting | Custom Redis-based fixed-window counter |

---

##  Features

- **1,000,000 checkboxes** — stored compactly as a Redis bitfield (125 KB total)
- **Real-time sync** — WebSocket updates broadcast to all connected clients
- **Virtual scrolling** — only renders visible rows, handles 1M checkboxes smoothly
- **OIDC / OAuth 2.0** — full Authorization Code Flow with a local mock auth server
- **Custom rate limiting** — no external packages; Redis-based fixed-window counters
- **Redis Pub/Sub** — broadcasts updates across multiple server instances (horizontal scale)
- **Anonymous read access** — guests can view but not toggle
- **Optimistic UI updates** — checkbox toggles feel instant; server state reconciles
- **Auto-reconnect** — WebSocket reconnects with exponential back-off
- **Live stats** — connected users, checked count updated in real time

---

##  Project Structure

```
million-checkboxes/
├── auth-server/
│   └── index.js          # Mock OIDC / OAuth 2.0 provider (port 3001)
├── server/
│   ├── index.js          # Express app + HTTP server entry point
│   ├── redis.js          # Redis client singleton (main + subscriber)
│   ├── rateLimiter.js    # Custom rate limiting (no external packages)
│   ├── websocket.js      # WebSocket server + Redis Pub/Sub handler
│   ├── middleware/
│   │   └── auth.js       # JWT session middleware
│   └── routes/
│       ├── auth.js       # OAuth 2.0 routes (login, callback, logout, me)
│       └── checkboxes.js # REST checkbox routes (state, stats, reset)
├── public/
│   ├── index.html        # Single-page app shell
│   ├── style.css         # Dark terminal aesthetic
│   └── app.js            # Virtual scroll + WebSocket client
├── package.json
├── .env.example
└── README.md
```

---

##  How to Run Locally

### Prerequisites

- Node.js 18+
- Redis 6+ running locally (`redis-server`)

### Steps

```bash
# 1. Clone the repo
git clone <your-repo-url>
cd million-checkboxes

# 2. Install dependencies
npm install

# 3. Configure environment
cp .env.example .env
# Edit .env if needed (defaults work for local dev)

# 4. Start both servers concurrently
npm run dev:simple

# OR start them separately in two terminals:
# Terminal 1:
npm run start:auth    # Auth server → http://localhost:3001
# Terminal 2:
npm start             # Main app   → http://localhost:3000
```

Visit `http://localhost:3000` in your browser.

---

##  Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Main app port |
| `AUTH_PORT` | `3001` | Auth server port |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection URL |
| `OIDC_ISSUER` | `http://localhost:3001` | URL of the auth server |
| `CLIENT_ID` | `million-checkboxes-client` | OAuth client ID |
| `CLIENT_SECRET` | *(see .env.example)* | OAuth client secret |
| `REDIRECT_URI` | `http://localhost:3000/auth/callback` | OAuth callback URL |
| `OIDC_SECRET` | *(see .env.example)* | Secret to sign OIDC tokens |
| `JWT_SECRET` | *(see .env.example)* | Secret to sign session tokens |
| `TOTAL_CHECKBOXES` | `1000000` | Number of checkboxes |

---

##  Redis Setup

```bash
# macOS (Homebrew)
brew install redis
brew services start redis

# Ubuntu / Debian
sudo apt install redis-server
sudo systemctl start redis

# Docker
docker run -d -p 6379:6379 redis:alpine

# Verify
redis-cli ping   # → PONG
```

### How checkbox state is stored in Redis

```
Key: "checkboxes"  (Redis String / Bitfield)

SETBIT checkboxes 0 1        # Check checkbox 0
GETBIT checkboxes 0          # Read checkbox 0
BITFIELD checkboxes INCRBY u1 42 1   # Atomically toggle checkbox 42
BITCOUNT checkboxes          # Count all checked boxes
GET checkboxes               # Get all 125,000 bytes at once
```

1,000,000 bits = **125,000 bytes = 125 KB** — stored in a single Redis key.

---

##  Auth Flow Explanation

This app implements **OAuth 2.0 Authorization Code Flow** with a local mock **OIDC provider**.

```
Browser                   Main App (3000)           Auth Server (3001)
  |                             |                           |
  |  GET /                      |                           |
  |─────────────────────────────>                           |
  |  (no session cookie)        |                           |
  |  render page w/ Sign In btn |                           |
  <─────────────────────────────|                           |
  |                             |                           |
  |  Click "Sign In"            |                           |
  |  GET /auth/login            |                           |
  |─────────────────────────────>                           |
  |  302 → /authorize?...state  |                           |
  <─────────────────────────────|                           |
  |                                                         |
  |  GET /authorize?client_id=...&state=...                 |
  |─────────────────────────────────────────────────────────>
  |  Show login form                                        |
  <─────────────────────────────────────────────────────────|
  |                                                         |
  |  POST /authorize (username/password)                    |
  |─────────────────────────────────────────────────────────>
  |  302 → /auth/callback?code=ABC&state=XYZ                |
  <─────────────────────────────────────────────────────────|
  |                             |                           |
  |  GET /auth/callback?code=ABC|                           |
  |─────────────────────────────>                           |
  |                             |  POST /token (code=ABC)   |
  |                             |─────────────────────────→|
  |                             |  { access_token, id_token}|
  |                             |←─────────────────────────|
  |                             |                           |
  |                             |  Verify id_token         |
  |                             |  Create session JWT      |
  |                             |  Set httpOnly cookie     |
  |  302 → /  (with cookie)     |                           |
  <─────────────────────────────|                           |
  |                             |                           |
  |  GET /  (with session cookie)|                          |
  |─────────────────────────────>                           |
  |  Serve app (logged in)      |                           |
  <─────────────────────────────|                           |
```

**Security highlights:**
- `state` parameter prevents CSRF attacks
- Session token stored in `httpOnly` cookie (not accessible from JS)
- Auth codes are one-time use and expire in 5 minutes

**Demo users:** `alice / password123`, `bob / password123`, `charlie / password123`

---

## 🔌 WebSocket Flow Explanation

```
Client                    Server                    Redis
  |                          |                        |
  |  WS Upgrade /ws          |                        |
  |──────────────────────────>                        |
  |  Verify session cookie   |                        |
  |  Add to connectedClients |                        |
  |  INCR connected_users    |──────────────────────>|
  |  Broadcast stats update  |                        |
  <──────────────────────────|                        |
  |                          |                        |
  |  { type:"toggle", index:N }                       |
  |──────────────────────────>                        |
  |  1. Rate-limit check     |──────────────────────>|
  |  2. Atomic toggle        |  BITFIELD INCRBY u1 N 1
  |                          |──────────────────────>|
  |  3. Publish update       |  PUBLISH checkbox-updates {...}
  |                          |──────────────────────>|
  |                          |                        |
  |  All servers receive the publish via SUB:         |
  |  { type:"update", index:N, state:1, updatedBy }   |
  |  Broadcast to all connected clients               |
  <──────────────────────────|                        |
```

---

##  Rate Limiting Logic

Rate limiting is implemented **from scratch** using Redis — no external packages.

### Strategy: Fixed-Window Counter

```
Key format: rl:{type}:{identifier}:{windowIndex}

windowIndex = Math.floor(Date.now() / windowMs)
```

For every request:
1. `INCR rl:{type}:{id}:{window}` — atomic increment
2. On first increment, `PEXPIRE` sets the key to auto-delete after `2 × windowMs`
3. If `count > limit` → reject with 429

### Configured Limits

| Type | Limit | Window | Keyed by |
|------|-------|--------|----------|
| `http_api` | 120 req | 60 s | IP or userId |
| `ws_toggle` | 15 toggles | 1 s | userId or IP |
| `ws_connect` | 10 connects | 60 s | IP |

### Why Redis?
- `INCR` is **atomic** — no race conditions even across multiple server instances
- Keys **auto-expire** — no cleanup needed, no memory leak
- Works **across all server instances** (unlike in-memory Maps)

---

##  Design Decisions

### How are 1M states stored?

As a **Redis bitfield** — 1 bit per checkbox, 1M bits = 125 KB in a single key.
This is ~8000× more compact than storing one Redis key per checkbox.

### How does toggling work without race conditions?

`BITFIELD checkboxes INCRBY u1 #N 1` is **atomic**. A 1-bit unsigned field wraps 0→1→0, which is a perfect toggle. No read-modify-write race.

### How do multiple servers stay in sync?

Redis **Pub/Sub**: when any server processes a toggle, it publishes `{index, state}` to the `checkbox-updates` channel. Every server instance subscribes to this channel and broadcasts to its own connected clients.

### How is the grid rendered without crashing the browser?

**Virtual scrolling**: only rows currently visible in the viewport (± 8 buffer rows) are in the DOM. A scroll event listener re-renders as the user scrolls. Total DOM nodes at any time: ~(viewportRows + 16) × COLS.

---

##  API Reference

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `GET` | `/api/checkboxes/state` | Optional | Full bitfield as base64 |
| `GET` | `/api/checkboxes/stats` | Optional | Checked count, online users |
| `POST` | `/api/checkboxes/reset` | Required | Reset all to unchecked |
| `GET` | `/auth/login` | — | Start OAuth flow |
| `GET` | `/auth/callback` | — | OAuth callback handler |
| `GET` | `/auth/logout` | — | Clear session |
| `GET` | `/auth/me` | Required | Current user info |
| `WS` | `/ws` | Cookie | WebSocket endpoint |
