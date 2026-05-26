# Lab 12 — Bonus: WebAssembly Containers — Port a QuickNotes Endpoint to WASM

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-WebAssembly-blue)
![points](https://img.shields.io/badge/points-4%2B4%2B2-orange)
![tech](https://img.shields.io/badge/tech-Spin%20%2B%20TinyGo-informational)

> **Goal:** Build a single QuickNotes-style HTTP endpoint as a WASM module served by Fermyon Spin's WAGI executor. Compare cold-start and image size against the Lab 6 Docker container. Bonus: run the same `.wasm` under a second WASI runtime (`wasmtime`) and compare.
> **Deliverable:** A PR from `feature/lab12` to the course repo with `wasm/` + `submissions/lab12.md`. Submit the PR link via Moodle.

> 🎁 **Bonus lab.** 10 pts total, structured as Task 1 (4) + Task 2 (4) + Bonus Task (2). This lab is the bonus; its full 10 pts count toward the bonus-labs grade weight.

---

## Overview

You will not be handed a `main.go` or a `spin.toml`. Read [Reading 12](../lectures/reading12.md) first, then write the module from requirements + docs.

By the end:
- A Go program compiled to WASM via TinyGo
- Spin runs the module behind HTTP at `localhost:3000`
- A measured comparison: cold-start, RSS, image size — WASM-Spin vs Docker-Lab 6

---

## Project State

**Starting point:** Lab 6 image works (baseline for comparison). QuickNotes source in `app/`.

**After this lab:** A `wasm/` directory with the WASM module + Spin manifest + reproducible perf numbers.

---

## Prerequisites

- Read [Reading 12](../lectures/reading12.md)
- TinyGo **0.34+** (`tinygo version`)
- Fermyon Spin **3.x** ([developer.fermyon.com/spin](https://developer.fermyon.com/spin))
- `hyperfine` (or `wrk`) for benchmarking
- *(Optional)* `wabt` for `wasm2wat` inspection

---

## Task 1 — Build a WASM Endpoint with Spin/WAGI (4 pts)

### 1.1: Requirements

Write a Go program in `wasm/main.go` that:

1. Implements the **WAGI** request/response contract — reads request info from environment variables and stdin, writes the HTTP response (headers + blank line + body) to stdout
2. Responds to a `GET /time` request with a **JSON** body containing the current **Moscow time** with at least:
   - `unix` (epoch seconds)
   - `iso` (RFC3339 timestamp)
   - `hour_minute` (e.g. `"15:42"`)
3. Sets a `Content-Type: application/json` response header
4. Builds successfully with **TinyGo** targeting **WASI**:
   ```
   tinygo build -o main.wasm -target=wasi ./main.go
   ```
5. Final `main.wasm` weight: **≤ 2 MB** (TinyGo + WASI keeps this trivial)

Write a `wasm/spin.toml` manifest that:
- Declares an HTTP trigger at the `/time` route
- Points the component at `main.wasm`
- Uses the **WAGI** executor (not the Spin-SDK executor — keeps the Go code minimal)
- Restricts `allowed_outbound_hosts` to `[]` (no outbound network — least privilege)

### 1.2: Design questions

- a) **Browser WASM vs server WASM:** `go build -o m.wasm -target=js/wasm` produces a *browser* artifact requiring the Go runtime + JS glue. TinyGo + `-target=wasi` produces a *server* artifact. What's missing in the server target — and what do you gain?
- b) **WAGI vs Spin SDK:** WAGI maps HTTP onto stdin/stdout, like CGI. Spin SDK exposes higher-level types. What did you gain by picking WAGI (which the lab requires)?
- c) **`allowed_outbound_hosts = []`** is the strictest setting. What's the *capability-based* security model behind this? Compare it to Docker's `--network none`
- d) **TinyGo stdlib gaps:** what part of upstream Go's stdlib does TinyGo *not* support that bit you during this lab? (Time zone data is a common one.)

### 1.3: Where to start

- 📖 [Spin docs](https://developer.fermyon.com/spin)
- 📖 [TinyGo with WASI](https://tinygo.org/docs/guides/webassembly/wasi/)
- 📖 [WAGI README](https://github.com/deislabs/wagi)
- 📖 [Spin's WAGI executor guide](https://developer.fermyon.com/spin/v2/wagi-fundamentals)

### 1.4: Run + verify

```bash
cd wasm/
tinygo build -o main.wasm -target=wasi ./main.go
spin up &       # binds on :3000 by default
sleep 1
curl -s http://127.0.0.1:3000/time | python3 -m json.tool
```

### 1.5: Document

In `submissions/lab12.md`:
- `main.go` + `spin.toml` (paste or link)
- TinyGo build output (size)
- `curl` response showing valid Moscow time JSON
- Design questions a-d answered

---

## Task 2 — Perf Comparison vs Lab 6 Container (4 pts)

### 2.1: Required measurements

Boot **both**: Spin + the Lab 6 Docker container. Measure:

1. **Warm latency** — after 5+ requests have already arrived, hit each with `hyperfine --warmup 5 --runs 50 'curl …'`. Capture p50.
2. **Cold latency** — kill the runtime; restart; measure time to first successful response. Repeat 5 times per platform; capture distribution.
3. **Artifact size** — `main.wasm` vs `quicknotes:lab6` image size.
4. **(Optional but appreciated)** RSS at idle — `ps -o rss` on the Spin process vs `docker stats` on the container.

### 2.2: Table

Build this table in `submissions/lab12.md` from real measurements:

| Dimension              | Lab 6 Docker | Lab 12 WASM/Spin |
|------------------------|-------------:|-----------------:|
| Artifact size          |            ? |                ? |
| Cold start (p50)       |            ? |                ? |
| Warm latency p50       |            ? |                ? |
| Warm latency p95       |            ? |                ? |
| RSS at idle *(opt)*    |            ? |                ? |

### 2.3: Design questions

- e) **What dominates each platform's cold start?** (Container: pull, image extract, runtime init. WASM: ?) Be specific.
- f) **For what workloads is WASM clearly the better choice — and where is Docker still right?** (See Reading 12 for trade-offs)
- g) **Multi-tenant safety:** WASM's capability sandbox is stronger than Linux namespaces. What concrete attack does a WASM platform make harder?

### 2.4: Document

In `submissions/lab12.md`:
- The full perf table from your real measurements
- A description of your test rig (machine, OS, region)
- Design questions e, f, g answered (5-6 sentences total is fine)

---

## Bonus Task — Cross-Runtime Portability (2 pts)

### B.1: Goal

WebAssembly's biggest pitch is **"compile once, run anywhere"** — a `.wasm` module should run identically under any WASI-conformant runtime. Prove (or disprove) this with **the same `main.wasm`** under **two** runtimes.

### B.2: Requirements

Run the **exact same `main.wasm`** from Task 1 under:

1. **Spin / WAGI** (already running from Task 1)
2. **`wasmtime`** standalone — the Bytecode Alliance's reference WASI runtime

For the `wasmtime` path you'll need to write a small WAGI-shaped wrapper: `wasmtime` doesn't speak HTTP by default. Two options:
- **Easiest:** Use `wasmtime serve` (if your version supports it) or `wasmtime` + a thin TCP→stdin shim
- **Cleaner:** Implement a 10-line Go HTTP server on your host that, per request, runs `wasmtime run main.wasm` with appropriate env vars (REQUEST_METHOD, PATH_INFO) and forwards stdout to the HTTP client — exactly the WAGI contract by hand

### B.3: Measure both

Use `hyperfine` (or `wrk`) on both endpoints with identical inputs:

| Metric | Spin/WAGI (port 3000) | wasmtime + shim (your port) |
|--------|----------------------:|----------------------------:|
| Cold start (process start → first response) | ? | ? |
| Warm latency p50 | ? | ? |
| Warm latency p95 | ? | ? |
| Process RSS at idle | ? | ? |

### B.4: Design questions

- h) **The same `.wasm` ran under both runtimes — what's the same, what's different?** What does "portable across runtimes" actually buy you in practice?
- i) **Why is `wasmtime` slower (or faster) than Spin** for the same module? What is Spin doing that pure `wasmtime` is not? Be specific (caching, JIT compilation, instance pooling)
- j) **If you were picking a WASM runtime for production tomorrow**, which factors (besides perf) would tip you toward Spin vs `wasmtime` vs `wasmedge` vs Cloudflare Workers? List ≥ 3

### B.5: Document

In `submissions/lab12.md`:
- Both run commands (`spin up …`, `wasmtime …`)
- The full 4-row comparison table with real numbers from your hardware
- Design questions h, i, j answered

---

## How to Submit

1. `wasm/` directory in your fork with `main.go`, `go.mod` (if any), `spin.toml`
2. *(Bonus)* The wasmtime wrapper / script you wrote
3. (Optional) `main.wasm` build artifact gitignored
4. `submissions/lab12.md` covers all attempted tasks
5. PR from `feature/lab12` → course repo's `main`
6. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (4 pts)
- ✅ `main.wasm` builds with TinyGo for WASI
- ✅ `spin up` serves `/time` returning Moscow-time JSON
- ✅ `main.wasm` ≤ 2 MB
- ✅ Design questions a-d answered

### Task 2 (4 pts)
- ✅ Full perf table with real numbers from your hardware
- ✅ Cold + warm + size captured
- ✅ Design questions e, f, g answered

### Bonus Task (2 pts)
- ✅ Same `main.wasm` runs under both Spin and `wasmtime`
- ✅ Comparison table with real measurements
- ✅ Design questions h, i, j answered

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — TinyGo + Spin WAGI module | **4** | main.wasm builds, /time JSON correct, ≤ 2 MB, design questions |
| **Task 2** — Perf comparison vs Lab 6 | **4** | Real table, cold + warm + size, design questions |
| **Bonus** — Cross-runtime portability | **2** | Same .wasm on two runtimes, comparison table, design questions |
| **Total** | **10** | (bonus lab — contributes toward bonus-labs grade weight) |

> 📝 **Lab 12 itself is a bonus lab** — its full 10 pts go into the bonus-labs grade component (20% of the final grade; see the course [README](../README.md)).

---

## Common Pitfalls

- 🪤 **`tinygo build` fails on stdlib features** — TinyGo's stdlib is a *subset* of upstream Go. `net/http` and reflection-heavy code don't fully work in WASI. Stick to `os`, `encoding/json`, `time`, `fmt`
- 🪤 **`go build -o m.wasm -target=js/wasm`** — that's *browser* WASM (needs the Go runtime + JS glue). Server-side requires TinyGo + WASI
- 🪤 **Spin can't find `main.wasm`** — `source =` in `spin.toml` is relative to the toml file's directory
- 🪤 **Cold-start measurement noisy** — first ever Spin start writes caches; warm once before measuring
- 🪤 **`time.LoadLocation("Europe/Moscow")` fails** in TinyGo — tzdata isn't embedded. Use UTC + manual offset (Moscow is UTC+3), or embed tzdata
- 🪤 **Docker cold-start measurement includes image pull** — pre-pull once on every host; or pin the image locally

---

## Guidelines

- Don't try to port *all* of QuickNotes to WASM — TinyGo stdlib gaps will frustrate you. One endpoint is the lab's scope
- Cold-start measurements need clean state on each iteration — `hyperfine --prepare 'cleanup' --setup 'start'`
- Read [Reading 12](../lectures/reading12.md) first — the trade-offs table there is the answer key for design questions f and g
- The "is WASM faster?" answer depends on workload — be specific about cold vs warm

---

## Resources

- 📖 [Spin developer docs](https://developer.fermyon.com/spin)
- 📖 [TinyGo with WASI guide](https://tinygo.org/docs/guides/webassembly/wasi/)
- 📖 [WAGI README](https://github.com/deislabs/wagi)
- 📖 [WASI documentation](https://wasi.dev/)
- 📖 [Bytecode Alliance — WASI Preview 2](https://bytecodealliance.org/articles/wasi-preview-2-launch)
- 🎥 [Lin Clark — *A cartoon introduction to WebAssembly*](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/)
- 📖 [`wasmtime` docs](https://docs.wasmtime.dev/) — for the Bonus runtime
- 🛠️ [`hyperfine`](https://github.com/sharkdp/hyperfine), [`wasm2wat`](https://github.com/WebAssembly/wabt), [`wasmtime`](https://wasmtime.dev/)
