# Product Engineering Challenge Submission

## Candidate

- **Name**: Patel Kunj
- **Email**: kunjmpatel254@gmail.com
- **GitHub**: https://github.com/kunjpatel177/Reconnecting-Real-Time-Feed.git
- **Selected problem**: Problem 3 — Reconnecting Real-Time Feed
- **Demo video**: https://drive.google.com/file/d/1pZF-ECkjEebog_myLCN-KvPQWttSjhar/view?usp=drive_link

---

## Run the project

### Prerequisites
- **Node.js**: v18+ or v20+ (tested on Node v24.14.0)
- **npm**: v9+ or v11+
- **MongoDB**: Local MongoDB instance (`mongodb://localhost:27017`) or MongoDB Atlas URI. *(Note: Automated tests run with an isolated in-memory database and require no pre-installed MongoDB instance).*

### Environment Variables
Configure the environment files before starting the services:

1. **Backend (`server/.env`)**:
   ```env
   PORT=5000
   MONGO_URI=mongodb://localhost:27017/incident_feed
   CLIENT_URL=http://localhost:5173
   ```

2. **Frontend (`client/.env`)**:
   ```env
   VITE_API_URL=http://localhost:5000/api
   VITE_SOCKET_URL=http://localhost:5000
   ```

### Setup & Run Commands

**Option 1: Quick start from repository root**
```bash
# 1. Install all dependencies
npm run install:all

# 2. Start Backend (runs on http://localhost:5000)
npm run dev:server

# 3. Start Frontend in a separate terminal (runs on http://localhost:5173)
npm run dev:client
```

**Option 2: Individual directories**
```bash
# Terminal 1 - Backend:
cd server
npm install
npm run dev

# Terminal 2 - Frontend:
cd client
npm install
npm run dev
```

The frontend application will be live at `http://localhost:5173` and communicate with the backend at `http://localhost:5000`.

---

## Successful scenario

To verify real-time live broadcasting without page refreshes:

1. Open two browser windows side-by-side (e.g., Chrome and Chrome Incognito, or two tabs) at `http://localhost:5173`.
2. Observe both windows connecting to `incident-1` and displaying `Status: Connected` (green badge).
3. In **Client A**, type the update message:
   ```
   Database investigation started
   ```
   and click **Send Update**.
4. **Observe Client B**: Without refreshing or any manual interaction, Client B immediately renders update `#1` with the server timestamp and message content.
5. In **Client B**, type a follow-up:
   ```
   Identified memory leak in connection pool
   ```
   and click **Send Update**.
6. **Observe Client A**: Client A immediately receives update `#2` in real time.

---

## Failure/recovery scenario

To verify connection state transitions, missed-update recovery, deduplication, and stable ordering:

1. With **Client A** and **Client B** both open on `incident-1`, disconnect **Client B**:
   - Click the **"Simulate Disconnect"** button on Client B (or use browser DevTools Network tab offline mode).
2. Observe **Client B**'s status immediately change to `Disconnected` (red badge).
3. In **Client A**, publish three consecutive updates while Client B is disconnected:
   - Update `#3`: `"Applied patch v1.4.2"`
   - Update `#4`: `"Restarting worker pods"`
   - Update `#5`: `"Worker pods healthy; latency normal"`
4. Reconnect **Client B**:
   - Click **"Reconnect Socket"** on Client B (or restore DevTools network).
5. **Observe Client B**:
   - The status transitions to `Connected`.
   - The indicator `Recovering missed updates...` flashes with a spinner as Client B queries `GET /api/incidents/incident-1/updates?after=2`.
   - All three missed updates (`#3`, `#4`, `#5`) are recovered from MongoDB and rendered.
   - **No duplicates**: Even if Socket.IO broadcast `#5` arrived concurrently with the HTTP recovery payload, centralized deduplication via stable update IDs (`_id`) ensures each update appears exactly once.
   - **Stable sequence order**: Updates are rendered in strictly ascending sequence order (`#1, #2, #3, #4, #5`).

---

## Run the tests

Run the automated test suite using:

```bash
# From the root directory:
npm test

# Or directly in the server directory:
cd server
npm test
```

### What the automated tests verify:
- **TEST 1 (`server/tests/live-update.test.js`)**: Real-time delivery. Verifies that publishing via `POST /api/incidents/:id/updates` allocates an atomic sequence, persists the document in MongoDB, and triggers a Socket.IO broadcast to subscribed room clients.
- **TEST 2 (`server/tests/recovery.test.js`)**: Cursor-based recovery. Seeds sequences 1 through 5 in `incident-1` and unrelated updates in `incident-2`. Queries `?after=3`, verifying that only sequences 4 and 5 are returned in ascending order and `incident-2` data is completely isolated.
- **TEST 3 (`server/tests/deduplication.test.js`)**: Deduplication resilience. Feeds the exact same update into the centralized client processor across three distinct paths (initial history, live socket broadcast, recovery replay) and verifies it appears only once.
- **TEST 4 (`server/tests/ordering.test.js`)**: Stable ordering. Feeds out-of-order updates (sequences 5, 3, 4) and asserts that the resulting client list is sorted deterministically [3, 4, 5] without corrupting the sequence cursor.

---

## Architecture and data flow

```
[React Frontend]
   │
   ├── REST (Axios) ─────► [Express API] ─────► [MongoDB] (Durable Source of Truth)
   │                           │
   └── Socket.IO Client ◄──────┘ (Transient Real-Time Distribution)
```

### Durable History vs. Transient Real-Time Delivery
A foundational architectural principle of this system is that **MongoDB is the durable source of truth** and **Socket.IO is merely a transient notification pipe**:
- We **never** treat WebSocket broadcasts as the sole copy of an update.
- The publishing lifecycle strictly enforces **Persistence-Before-Broadcast**:
  1. Client sends HTTP POST.
  2. Server validates message payload.
  3. Server atomically increments the monotonic sequence counter.
  4. Server writes the document to MongoDB and awaits confirmation.
  5. *Only after MongoDB confirmation*, Socket.IO emits `incident:update` to the room `incident:<incidentId>`.
  6. Server returns HTTP 201 with the persisted document.
- If MongoDB persistence fails, the update is rejected with an HTTP 500/400 error and **is never broadcast to the room**.

---

## Technology choices

### React & Vite
- **Why**: React's declarative state model combined with `useRef` provides precise control over deduplication sets and cursor tracking across asynchronous events. Vite ensures near-instant HMR and clean ES module bundling.
- **Trade-offs**: React's re-render cycle must be guarded against unnecessary re-renders when managing transient deduplication sets (solved by holding `seenUpdates` in a mutable `useRef` rather than state).

### Node.js & Express
- **Why**: Lightweight, asynchronous event-driven I/O ideal for handling concurrent HTTP requests and managing long-lived WebSocket connections within the same process.
- **Trade-offs**: Single-threaded event loop requires non-blocking database queries and offloading CPU-intensive tasks.

### MongoDB & Mongoose
- **Why**: MongoDB provides atomic, single-roundtrip sequence increments via `findOneAndUpdate` with `$inc` and `upsert: true`. Unique compound indexes (`{ incidentId: 1, sequence: 1 }`) guarantee sequence integrity at the storage layer without distributed lock managers.
- **Trade-offs**: MongoDB document transactions are heavier than relational sequence generators; using a dedicated counter collection with atomic `$inc` achieves the same guarantee without full multi-document ACID overhead.

### Socket.IO
- **Why**: Built-in room isolation (`incident:<id>`), automatic reconnection management with exponential backoff, and transparent transport fallback (WebSocket to HTTP long-polling).
- **Alternatives Considered**:
  - *Raw WebSockets (`ws`)*: More lightweight, but requires hand-rolling reconnection loops, exponential backoff, heartbeat/ping-pong, and custom room subscription protocols.
  - *Server-Sent Events (SSE)*: Native HTTP-based streaming with automatic browser reconnect, but lacks built-in room clustering and bidirectional control frames.

---

## Important decisions

1. **Persistence Before Broadcast**: Broadcasts are never emitted optimistically. If MongoDB write fails, no update is broadcast.
2. **Monotonic Sequence Numbers as Primary Cursor**: Timestamps suffer from NTP adjustments, clock skew, millisecond collisions, and timezone discrepancies. Monotonic sequence numbers ($1, 2, 3...$) provide total, deterministic ordering and unambiguous `sequence > after` recovery.
3. **Centralized Client Deduplication via Stable ID**: All incoming updates—regardless of arrival channel (initial history fetch, live socket broadcast, or reconnection recovery)—funnel through a single client-side `processIncomingUpdates()` function. Stable MongoDB `_id` tracking in a `Set` guarantees that overlapping events are discarded seamlessly.
4. **Bounded Socket.IO Reconnection & Backoff**: Configured with bounded attempts (`8`), initial delay (`1000ms`), and maximum delay (`5000ms`) to avoid thundering-herd problems and tight retry loops during server restarts.

---

## Assumptions and limitations

- **Feed Scope**: The UI focuses on `incident-1` for demonstration purposes, though the backend is fully generalized and supports any arbitrary `incidentId`.
- **Single Server Instance**: The prototype runs on a single Node.js instance. Multi-node horizontal scaling is documented below.
- **No Authentication/Authorization**: Excluded per challenge scope to keep the code focused on real-time reliability.
- **No Offline Message Queue**: If the network is disconnected, new update submissions fail at the HTTP level rather than queueing locally in IndexedDB.
- **Prototype Scale**: Tested for hundreds of concurrent connections and standard incident response feeds; not benchmarked for millions of concurrent connections.

---

## Production and scale

### What happens if a client disconnects immediately after sending an update?
- **Case 1: Server persisted update before disconnect**: The update is safely committed in MongoDB and assigned a sequence number. Although the client might miss the HTTP 201 response or live broadcast, the client's next reconnect will query `GET /updates?after=<lastSeq>` and recover the update seamlessly.
- **Case 2: Network drops before request reaches server**: The update never reached the server; no sequence was allocated and no record was created. The user/client receives a network error and can choose to resubmit.

### How would multiple backend instances share and order events?
In a multi-node production deployment:
1. **Shared Sequence Generation**: Centralize sequence allocation in a shared distributed store (e.g., MongoDB counter with `$inc`, or Redis `INCRBY`), or use partitioned sequence generators.
2. **Socket.IO Adapter**: Deploy the `@socket.io/redis-adapter` or `@socket.io/mongo-adapter` so that updates broadcast on Node Instance A are published to Redis pub/sub and distributed to sockets connected to Node Instance B.
3. **Change Streams**: Alternatively, worker instances can subscribe to MongoDB Change Streams (`watch({ 'operationType': 'insert' })`) to fan out live updates to local Socket.IO rooms reliably.

### How would you prevent an unbounded history replay?
1. **Cursor + Bounded Limit Pagination**: The `/api/incidents/:id/updates` endpoint enforces a strict maximum page limit (`MAX_LIMIT = 100`, default `50`). Clients page through history using `nextCursor` until `hasMore` is false.
2. **Retention Policies (TTL)**: In high-volume systems, archive resolved incidents or implement MongoDB TTL indexes for older historical logs.
3. **Feed Snapshots**: Create periodic feed checkpoints or snapshots for incidents spanning thousands of updates, allowing clients to hydrate from the latest snapshot rather than replaying from sequence 0.

### What would you monitor in production?
- **Socket Metrics**: Active WebSocket connections, connection duration, disconnect reasons, reconnect rates, reconnect failures.
- **Recovery Metrics**: Recovery query latency, average events recovered per reconnect, recovery failure rate.
- **Data Integrity**: Duplicate detection rate (events dropped by client deduplication), out-of-order arrival rate.
- **Persistence & Broadcast**: Database insert latency, sequence allocation latency, persistence error rate, broadcast dispatch latency.
- **Feed Metrics**: Updates per incident, incident room membership count.

---

## AI usage

AI coding tools (specifically Google DeepMind Antigravity Coding Assistant) were utilized during the development of this project for initial boilerplate scaffolding, test structuring, and documentation drafting. All code, architecture, sequence mechanisms, deduplication logic, and edge-case handling were carefully reviewed, verified, and tested by the candidate.
