# Lab 12 — Bonus: WebAssembly Containers — Port a QuickNotes Endpoint to WASM

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-WebAssembly-blue)
![points](https://img.shields.io/badge/points-10-orange)
![tech](https://img.shields.io/badge/tech-Spin%20%2B%20TinyGo-informational)

> **Goal:** Build a tiny WASM module that serves a single QuickNotes-style endpoint via Fermyon Spin (WAGI executor). Compare cold-start and image size against the Lab 6 Docker container.
> **Deliverable:** A PR from `feature/lab12` to the course repo with `wasm/` + `submissions/lab12.md`. Submit the PR link via Moodle.

> 🎁 **This is a bonus lab.** Worth 10 pts, no Bonus row — Task 1 + Task 2 *are* the challenge.

---

## Overview

By the end:
- A Go program compiled to WASM via TinyGo
- Spin runs the module behind HTTP at `localhost:3000`
- The endpoint returns Moscow time as JSON (a single hot endpoint — keeping the WASM port minimal)
- A measured comparison: cold-start, RSS, image size for WASM-Spin vs Docker-Lab 6

---

## Project State

**Starting point:** Lab 6 image works (for comparison baseline). QuickNotes source in `app/`.

**After this lab:** A `wasm/` directory with `main.go`, `spin.toml`, build artifacts, and reproducible perf numbers.

---

## Prerequisites

- TinyGo 0.34+ (`tinygo version`)
- Fermyon Spin 3.x (`spin --version`) — install via [developer.fermyon.com/spin](https://developer.fermyon.com/spin)
- `wabt` (optional, for `wasm2wat` inspection)
- `hyperfine` or `wrk` for benchmarking

---

## Task 1 — Build a WASM Endpoint with Spin/WAGI (6 pts)

### 1.1: Lay out `wasm/`

```text
wasm/
├── main.go
├── go.mod
├── spin.toml
└── README.md
```

### 1.2: `wasm/main.go` — Moscow time as JSON

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"time"
)

func main() {
	// WAGI: read the request from the environment + stdin, write to stdout
	method := os.Getenv("REQUEST_METHOD")
	path   := os.Getenv("PATH_INFO")
	if method == "" { method = "GET" }
	if path == ""   { path   = "/" }

	moscow, _ := time.LoadLocation("Europe/Moscow")
	now := time.Now().In(moscow)

	resp := map[string]any{
		"endpoint":    path,
		"method":      method,
		"city":        "Moscow",
		"unix":        now.Unix(),
		"iso":         now.Format(time.RFC3339),
		"hour_minute": now.Format("15:04"),
	}

	body, _ := json.Marshal(resp)

	// WAGI response: headers, blank line, body
	fmt.Println("Content-Type: application/json")
	fmt.Println()
	fmt.Println(string(body))

	// suppress unused
	_ = http.StatusOK
}
```

### 1.3: `wasm/go.mod`

```text
module quicknotes-wasm

go 1.24
```

### 1.4: Build with TinyGo (WASI target)

```bash
cd wasm/
tinygo build -o main.wasm -target=wasi -no-debug ./main.go
ls -lh main.wasm     # expect ~500 KB - 2 MB
```

### 1.5: `wasm/spin.toml`

```toml
spin_manifest_version = 2

[application]
name = "quicknotes-wasm"
version = "0.1.0"

[[trigger.http]]
route = "/time"
component = "moscow-time"

[component.moscow-time]
source = "main.wasm"
executor = { type = "wagi" }
allowed_outbound_hosts = []
```

### 1.6: Run it

```bash
spin up   # binds on :3000 by default
# in another terminal:
curl -s http://127.0.0.1:3000/time | python3 -m json.tool
```

### 1.7: Document

In `submissions/lab12.md`:
- `main.go` + `spin.toml`
- `tinygo build` output (size)
- `curl` against `/time` showing the JSON response
- One paragraph: *what's the WAGI executor doing? compare with a traditional HTTP framework*

---

## Task 2 — Performance Comparison: WASM vs Lab 6 Container (4 pts)

### 2.1: Boot both

In one terminal:

```bash
cd wasm/
spin up &
SPIN_PID=$!
```

In another:

```bash
docker run -d --name qn-lab12 -p 8080:8080 quicknotes:lab6
```

### 2.2: Cold + warm latency

```bash
# warm: hit it a few times to make sure both are alive
for i in {1..5}; do curl -s -o /dev/null http://localhost:3000/time; done
for i in {1..5}; do curl -s -o /dev/null http://localhost:8080/health; done

# now benchmark warm
hyperfine --warmup 3 --runs 50 \
  'curl -s -o /dev/null http://localhost:3000/time' \
  'curl -s -o /dev/null http://localhost:8080/health'
```

### 2.3: Cold-start (kill + restart)

```bash
# WASM cold
kill $SPIN_PID
sleep 1
hyperfine --warmup 0 --runs 5 --shell=bash \
  'spin up --listen 127.0.0.1:3000 &  sleep 0.3 && curl -s http://localhost:3000/time && kill %1'

# Docker cold
hyperfine --warmup 0 --runs 5 --shell=bash \
  'docker run -d --name qn-cold -p 8081:8080 quicknotes:lab6 > /dev/null && \
   until curl -fs http://localhost:8081/health > /dev/null; do :; done && \
   docker rm -f qn-cold > /dev/null'
```

### 2.4: Sizes

```bash
du -h wasm/main.wasm
docker images quicknotes:lab6 --format '{{.Size}}'
```

### 2.5: Document

In `submissions/lab12.md`:
- Hyperfine results: warm p50/p95/p99 for each
- Cold-start results
- Sizes: WASM module vs Docker image
- A small comparison table:

  | Dimension       | Lab 6 Docker | Lab 12 WASM/Spin |
  |-----------------|--------------|------------------|
  | Image size      | XX MB        | XX MB            |
  | Cold start      | XX ms        | XX ms            |
  | Warm latency p50| XX ms        | XX ms            |
  | RSS at idle     | XX MB        | XX MB            |

- 5-6 sentences: *for what workloads is the WASM model clearly better? what's the catch?* Reference Reading 12 if helpful.

---

## How to Submit

1. `wasm/` directory in your fork with `main.go`, `go.mod`, `spin.toml`, `main.wasm` (optional — gitignored if large)
2. `submissions/lab12.md` covers both tasks
3. PR from `feature/lab12` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `main.wasm` built with TinyGo (WASI target)
- ✅ `spin up` serves `/time` returning Moscow time JSON
- ✅ Brief explanation of WAGI

### Task 2 (4 pts)
- ✅ Warm + cold latencies measured (hyperfine)
- ✅ Sizes of both artifacts captured
- ✅ Comparison table + written analysis

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — TinyGo + Spin module serving /time | **6** | main.wasm built, spin.toml correct, curl response works |
| **Task 2** — Perf comparison vs Lab 6 | **4** | Cold + warm + size table, written trade-off |
| **Total** | **10** | (bonus lab — no Bonus row) |

> 📝 **No "Bonus Task" in this lab.** Lab 12 is itself a bonus lab — Task 1 + Task 2 *are* the challenge. The lab's full 10 pts contribute toward your bonus-labs grade weight (see the course README).

---

## Common Pitfalls

- 🪤 **`tinygo build` fails on stdlib features** — TinyGo's stdlib is a *subset* of upstream Go. Things like `net/http` don't fully work in WASI. Stick to `os`, `encoding/json`, `time`, `fmt`
- 🪤 **`go build -o main.wasm -target=js/wasm`** — that's *browser* WASM with the Go runtime + JS glue. Use TinyGo + `-target=wasi` for server-side
- 🪤 **Spin can't find `main.wasm`** — `source =` in `spin.toml` is relative to the toml file
- 🪤 **Cold-start measurement noisy** — the first ever Spin start writes caches; warm it once, then measure
- 🪤 **`time.LoadLocation` fails in TinyGo** — embed tzdata or use UTC + manual offset (Moscow is UTC+3)
- 🪤 **Docker cold-start measurement includes image pull** — pre-pull on every host; or pin the image as a local-only tag

---

## Guidelines

- Don't try to port *all* of QuickNotes to WASM — TinyGo's stdlib gaps will frustrate you. One hot endpoint is the lab's scope
- Cold-start measurements need clean state on each iteration — use `--prepare` in hyperfine if needed
- The "is WASM-Spin faster?" answer depends on workload — be specific about cold-start vs warm
- Read [Reading 12](../lectures/reading12.md) first if you haven't

---

## Resources

- 📖 [Spin docs](https://developer.fermyon.com/spin)
- 📖 [TinyGo with WASI](https://tinygo.org/docs/guides/webassembly/wasi/)
- 📖 [WAGI README](https://github.com/deislabs/wagi)
- 📖 [Bytecode Alliance — WASI](https://wasi.dev/)
- 🎥 [Lin Clark — A cartoon intro to WebAssembly](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/)
- 🛠️ [`hyperfine`](https://github.com/sharkdp/hyperfine), [`wasm2wat`](https://github.com/WebAssembly/wabt)
