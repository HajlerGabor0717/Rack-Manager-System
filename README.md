[README.md](https://github.com/user-attachments/files/27311555/README.md)
# RMS — Rack Manager System

> A real-time, file-based coordination tool for a server hardware testing facility. Battle-tested in production, used daily by a multi-shift technician team.

![Status](https://img.shields.io/badge/status-in%20production-success)
![Tech](https://img.shields.io/badge/tech-HTA%20%2F%20Vanilla%20JS-blue)
![Dependencies](https://img.shields.io/badge/dependencies-zero-brightgreen)
![Year](https://img.shields.io/badge/built-2024-lightgrey)

![RMS overview](docs/screenshot-overview.png)
*The main board: rack lifecycle stages with color-coded claims, cabling status, team palette and live chat. Internal test names anonymized as TEST_ALPHA / TEST_BETA / TEST_GAMMA / TEST_DELTA.*

---

## A note on this repository

This is the **real tool** that runs on our floor every day. Internal test names, rack number ranges, and workflow specifics have been anonymized (e.g. `TEST_ALPHA`, `TEST_BETA`) out of respect for my employer's confidentiality. The architecture, code patterns, and engineering decisions are unchanged from the production version.

UI strings are currently in English. The original production deployment uses Hungarian (the language of the team running it).

---

## The Problem

In our server testing facility, dozens of server racks are physically moved in and out of test stages every shift. Multiple technicians work in parallel across several test types (burn-in, validation, error-recovery, and transfer/relocation flows).

The reality on the floor was chaotic:

- **No visibility** of who was working on which rack
- **Cross-work conflicts** — two technicians unknowingly running tests on the same hardware
- **Lost work** when a rack got moved without anyone tracking it
- **No persistent log** of which units had passed which stage

Management hadn't prioritized building an internal tool, and bringing in a vendor solution would have taken months of procurement.

I built RMS to fix it. Today, my entire team uses it daily.

---

## The Solution

RMS is a **single-file desktop app** (HTML Application / `.hta`) that runs on Windows without installation. It lives on a shared network drive, and every technician launches the same file from their workstation.

State is synchronized through plain JSON files on the shared drive — **no server, no database, no install, no admin rights required**. This was a hard constraint: the facility's IT policy doesn't allow installing custom software on testing workstations.

### Core features

- **Live rack board** per test stage (TEST_ALPHA, TEST_BETA, TEST_GAMMA, TEST_DELTA) plus a transfer/staging flow
- **Claim system** — a technician picks a color identity and claims racks they're actively working on; everyone else sees who has what
- **Real-time team chat** built directly into the tool
- **Conflict-free concurrent writes** using optimistic file locks and per-user write-isolated state files
- **State persistence** — survives crashes, reboots, and the daily chaos of a multi-shift environment

![Claims and transfer flow](docs/screenshot-claims.png)
*Color-coded claims in action. Rack 210 is claimed by "Bryan" (blue), the transfer entry `106 → 102 [BETA]` shows the full from/to/type metadata with a DONE marker, and rack 506 is claimed by a different technician (green).*

---

## Architecture Highlights

The interesting engineering problem here was: **how do you build a real-time, multi-user coordination tool with no server, no database, and no central authority?**

The answer is a hybrid of three patterns:

### 1. Per-user write-isolated state files

The biggest race-condition risk in shared-filesystem coordination is two clients writing the same file at the same time. RMS sidesteps this by giving every user their own state file:

```
shared-drive/
├── data.json              ← rack lists (rarely written, lock-protected)
├── data.lock              ← short-lived lock file
├── chat.json              ← team chat (lock-protected, short writes)
├── chat.lock
└── colors/
    ├── slot_0.json        ← user 0's claims & identity (only user 0 writes)
    ├── slot_1.json        ← user 1's claims & identity (only user 1 writes)
    └── ...
```

User-specific data (which racks they've claimed, their display name, color identity) lives in `slot_X.json` files that **only that user ever writes to**. No race condition is possible by design.

Shared data (rack lists, chat) uses a different strategy:

### 2. Optimistic file locking with timeout

For the rare writes to `data.json` (adding/removing a rack) and the more frequent writes to `chat.json`, RMS uses a classic optimistic lock pattern:

1. Write a `*.lock` file containing a timestamp
2. If a lock already exists and is younger than `LOCK_TIMEOUT` (3s), back off and retry
3. Stale locks (older than the timeout) are ignored — assumed to be a crashed client
4. After the write, delete the lock

Bounded retries (`MAX_RETRY=15`, `RETRY_DELAY=150ms`) keep the UI responsive even under contention.

### 3. Hash-diff polling instead of full re-render

Every 4 seconds the client polls the shared state files. To avoid expensive re-renders and flicker, each file's content is hashed and compared against the last known hash. If unchanged → no work. If changed → re-render only the affected section.

```javascript
function simpleHash(s) {
  var h = 0;
  for (var i = 0; i < s.length; i++) {
    h = ((h << 5) - h) + s.charCodeAt(i);
    h = h & h;
  }
  return String(h);
}
```

This pattern keeps the tool responsive even when 8+ technicians are connected simultaneously.

### 4. Defense-in-depth validation

Every mutating action validates twice — once on the client for instant UX feedback, and again inside the lock-protected write path to defeat TOCTOU races between concurrent users:

```javascript
// Client-side check (fast UX feedback)
var conflict = findStationConflict(D, n, section);
if (conflict) { toast('Station ' + n + ' is already in ' + conflict, 'warn'); return; }

// Server-side recheck under lock (race protection)
function dataWrite(change) {
  acquireLock(...);
  var disk = loadData();             // re-read latest state under the lock
  var conflict = findStationConflict(disk, change.station, change.section);
  if (conflict) { /* abort */ }
  // ...
}
```

This pattern is applied to station uniqueness, name uniqueness, and the login/logout flow.

![Validation feedback](docs/screenshot-validation.png)
*Inline validation: rejected station numbers explain which ranges are valid, instead of just throwing them away.*

---

## Tech Stack & Constraints

| Choice | Why |
|---|---|
| **HTML Application (`.hta`)** | The facility's IT policy forbids installing software on test workstations. HTA runs on stock Windows with zero install. |
| **Vanilla JavaScript (ES5)** | HTA uses Internet Explorer's rendering engine. No ES6+, no frameworks, no build step. |
| **JSON files on shared drive** | No server infrastructure approval needed. Every workstation already has access to the network drive. |
| **`ActiveXObject('Scripting.FileSystemObject')`** | The only way to do filesystem I/O in HTA. |

These constraints were not a choice — they were the reality of deploying inside a regulated industrial environment. Working within them is what made the project interesting.

---

## Running it

1. Clone the repo
2. Drop `RMS.hta` into a shared folder (or just open it locally to try it)
3. Double-click `RMS.hta` on a Windows machine

The app will create `data.json`, `chat.json`, and the `colors/` directory on first run.

---

## What I'd do differently

If I were building v2 today, with fewer constraints:

- **Move to a small Node + WebSocket backend** to eliminate polling and get true real-time updates
- **SQLite instead of JSON files** for atomic transactions and easier querying
- **Replace the file-lock dance** with proper transactional writes
- **TypeScript + a real component framework** (Svelte or React) for maintainability
- **Audit log** — currently the tool tracks current state, not history. A "who claimed what when" log would help post-shift analysis

That said, every one of those decisions would have required IT approval and infrastructure I didn't have. The constraint-driven design is part of why this tool actually got deployed and used — instead of sitting in a backlog.

---

## License

MIT — see [LICENSE](LICENSE).
