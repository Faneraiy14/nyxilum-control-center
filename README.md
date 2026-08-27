# Nyxilum Control Center

*[Українською](README.uk.md)*

Live system monitor written entirely in [NyxilumLang](https://github.com/Faneraiy14/NyxilumLang) — a single file (`main.nx`), no other language or library. At the same time, it's the ecosystem's flagship project: it deliberately exercises nearly the whole language in one coherent scenario, not as an artificial feature checklist.

## What it does

Run one file — get a web dashboard that:

- **Shows live metrics** — process memory, CPU count, the VM's own memory usage (`gc_stats()`) — and updates them in the browser WITHOUT a page reload, via a WebSocket push every 2 seconds.
- **Draws a memory graph** right in the browser (canvas + JS) from the last ~2 minutes of history.
- **Remembers history across restarts** — metrics are written to a persistent database (`NyxilumDb`), so `/api/history` still returns previous data after the server restarts.
- **Pings a list of hosts** — add addresses in the dashboard (each one is saved to the database and survives a restart), hit "Ping all" — the server pings them IN PARALLEL (each on its own thread) and shows the latency next to each one.
- **Shows the OS's top 15 processes** by memory along with live %CPU — a mini-`htop` right in the browser.
- **Sends a full-screen notification when something's wrong** — a system push (the same kind of bubble as in messaging apps) fires if process memory or free disk space crosses a threshold; it won't spam again until the metric returns to normal and crosses the threshold once more.
- **Exports the whole metrics history** as a single archive (`zipCreate`) with one click — the server packs `history.json` into a zip and returns its path on disk.
- **Doesn't crash from any single failure.** Ping timeout, GC error, corrupted history, unreachable notification — each data source is wrapped in `try/catch` and won't take down the whole server.
- **On Windows, additionally opens a native window** (Windows Forms canvas) with the same live memory graph, separate from the browser — updated via a channel (`newChannel`/`channelSend`/`channelReceive`) from the background collector, not by polling. On Linux/Mac the app detects this itself and simply runs as a web server, without crashing.

Endpoints:

| Route | What it does |
|---|---|
| `GET /` | the dashboard itself (HTML + JS) |
| `GET /api/history` | metrics history as JSON |
| `GET /api/gc` | current GC stats of the VM itself |
| `GET /api/processes` | OS's top 15 processes by memory, with %CPU |
| `GET /api/hosts` | list of hosts being monitored |
| `POST /api/hosts` | add a host (request body is the address itself) |
| `DELETE /api/hosts?host=...` | remove a host from the list |
| `GET /api/ping-all` | pings all hosts in the list in parallel |
| `GET /api/export` | packs history into a zip on the server's disk, returns the path |
| `WS /live` | live stream of new metrics, checks once a second and pushes if there's something new |

## What language features this shows off

- **`httpServer` + `NyxilumDb`** — the web dashboard, metrics history, and host list all survive a restart
- **WebSocket server** (`httpServer(port, handler, wsHandler)`) — live push to the browser without client-side polling
- **`spawn`/`workerJoin`** — the background metrics collector and every `ping` check (including pinging all hosts at once) run on separate threads, without blocking the server
- **`newChannel`/`channelSend`/`channelReceive`** — live updates for the GUI window (Windows-only)
- **`createCanvas` + 2D graphics** — a native window with a live memory graph (Windows)
- **`procRun` + `regex`** — running `ping` against an arbitrary host, parsing latency from the output
- **`osProcessList()`** — list of all OS processes with memory and %CPU
- **`osDiskFree()`** — free/total disk space
- **`notify()`** — system push notifications when memory/disk crosses a threshold
- **`zipCreate`** — exporting the metrics history as an archive
- **`gc_stats()`** — the VM's own memory usage is visible right in the dashboard
- **`try/catch`** — no data source can take down the rest of the server on failure

`osProcessList()`, `osDiskFree()`, and `notify()` didn't exist in NyxilumLang before this project — they were added SPECIFICALLY because Control Center needed them. The language literally grows together with the app.

## Running it

```
nx main.nx
```

The dashboard will be at `http://localhost:8090/` (the port can be changed via `PORT`). On Windows, a native window with the live graph will also open; on Linux/Mac the app detects this itself and just runs as a web server, without crashing.

### Environment variables

- `PORT` — the web server's port (default `8090`)
- `NCC_DATA` — folder for the metrics database (default `./data`)
- `NCC_NO_GUI=1` — force-disable the GUI window even on Windows

## Why it's built this way

On thread safety: the worker collector and the HTTP/WS handlers run on different threads, each with its own VM (`spawn`), but they all write to/read from the SAME `NyxilumDb` database — database handles (as well as WebSocket and channel handles) are deliberately NOT copied between threads (`ConcurrencyModule.DeepCopy`), but passed by reference, so everyone sees the same data with no races.

## License

MIT, see [LICENSE](LICENSE).
