# Lab 6 — Containers: Multi-Stage Dockerfile + Compose for QuickNotes

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Containers-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Docker%2028.x-informational)

> **Goal:** Build a ≤ 25 MB multi-stage Docker image of QuickNotes. Run it with Compose alongside a sidecar. Harden the runtime with the 6 security defaults from Lecture 6.
> **Deliverable:** A PR from `feature/lab6` to the course repo with `submissions/lab6.md`. Submit the PR link via Moodle.

---

## Overview

By the end of this lab you have:
- A multi-stage `Dockerfile` at `app/Dockerfile` producing a tiny distroless image
- A `compose.yaml` running QuickNotes with a healthcheck
- A persistent volume for `data/notes.json` (so notes survive `docker compose down`)
- A hardened container (USER nonroot, read-only root, no caps) — Bonus

---

## Project State

**Starting point:** QuickNotes builds with `go build` (Lab 1). CI runs (Lab 3).

**After this lab:** `docker compose up` is the one-command way to run the project. The image is ~15-25 MB.

---

## Prerequisites

- Docker **28.x** (`docker --version`)
- Docker Compose v2 (built into Docker)
- ≥ 4 GB free disk for image cache

---

## Task 1 — Multi-Stage Dockerfile, ≤ 25 MB (6 pts)

### 1.1: Write `app/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7

# ─── builder ───
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux \
    go build -trimpath -ldflags='-s -w' -o /quicknotes .

# ─── runtime ───
FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=builder /quicknotes /app/quicknotes
COPY seed.json /app/seed.json
USER nonroot:nonroot
EXPOSE 8080
ENV ADDR=":8080" DATA_PATH="/data/notes.json" SEED_PATH="/app/seed.json"
ENTRYPOINT ["/app/quicknotes"]
```

### 1.2: Build + smoke test

```bash
cd app/
docker build -t quicknotes:lab6 .
docker images quicknotes:lab6     # confirm size — target ≤ 25 MB

mkdir -p ./data
docker run --rm -p 8080:8080 -v $PWD/data:/data quicknotes:lab6 &
sleep 2
curl -s http://localhost:8080/health
curl -s http://localhost:8080/notes
docker stop $(docker ps -q --filter ancestor=quicknotes:lab6)
```

### 1.3: Document

In `submissions/lab6.md`:
- `docker images quicknotes:lab6` output (size column)
- `docker inspect quicknotes:lab6 | jq '.[0].Config'` excerpt — confirm USER, ExposedPorts, Entrypoint
- The size of the *base* `golang:1.24-alpine` image vs your final image (use `docker images golang:1.24-alpine`) — explain the delta

---

## Task 2 — Compose + Healthcheck + Persistent Volume (4 pts)

### 2.1: Write `compose.yaml` at the repo root

```yaml
services:
  quicknotes:
    build: ./app
    image: quicknotes:lab6
    ports:
      - "8080:8080"
    volumes:
      - quicknotes-data:/data
    environment:
      ADDR: ":8080"
      DATA_PATH: "/data/notes.json"
    healthcheck:
      test: ["CMD", "/app/quicknotes", "--help"]
      interval: 5s
      timeout: 2s
      retries: 3
    restart: unless-stopped

  busybox-probe:
    image: busybox:1.37
    depends_on:
      quicknotes:
        condition: service_healthy
    command: >
      sh -c "wget -qO- http://quicknotes:8080/health && echo OK"

volumes:
  quicknotes-data:
```

> ⚠️ **distroless has no shell** so the healthcheck above uses the binary. If you need an HTTP healthcheck, use a debug image (`debian:stable-slim`) for that purpose only, or rely on Docker's built-in `HEALTHCHECK` semantics with a sidecar.

### 2.2: Persistence test

```bash
docker compose up --build -d
sleep 3

curl -X POST -H 'Content-Type: application/json' \
  -d '{"title":"durable","body":"survive a restart"}' \
  http://localhost:8080/notes
curl -s http://localhost:8080/notes | grep "durable"

docker compose down                 # NOT `down -v` — keep the volume
docker compose up -d
sleep 3

curl -s http://localhost:8080/notes | grep "durable"   # still there ✅
```

### 2.3: Document

In `submissions/lab6.md`:
- The full `compose.yaml`
- The before/after `docker compose down && up` evidence that the note survived
- One sentence: *what would happen with `docker compose down -v`?*

---

## Bonus Task — Apply the 6 Security Defaults (2 pts)

Harden the QuickNotes container in `compose.yaml`:

### B.1: Read-only root + tmpfs

```yaml
services:
  quicknotes:
    # ... existing config ...
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "65532:65532"           # nonroot user from distroless
```

### B.2: Prove the constraints

```bash
docker compose up -d
# the container can't `apt install` (no shell + read-only)
docker compose exec quicknotes sh    # expected: fails (no sh)
# the container can't write outside /data + /tmp
docker compose exec quicknotes touch /etc/test 2>&1 || echo "blocked ✅"
```

### B.3: Run a Trivy scan (preview of Lab 9)

```bash
docker run --rm aquasec/trivy:0.59.1 image \
  --severity HIGH,CRITICAL \
  --no-progress \
  quicknotes:lab6
```

Document any HIGH/CRITICAL CVEs in your submission (often: zero — distroless is *that* small). Lab 9 will dig deeper.

### B.4: Document

In `submissions/lab6.md`:
- The hardened `compose.yaml` snippet
- Evidence the container can't escape (failed `sh`, failed `touch /etc/test`)
- Trivy output (or "0 CVEs found, distroless image")
- One sentence: *which of the 6 defaults gives you the most security per line of YAML?*

---

## How to Submit

1. `app/Dockerfile` and `compose.yaml` exist in your fork
2. `submissions/lab6.md` covers all attempted tasks
3. PR from `feature/lab6` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `app/Dockerfile` uses multi-stage build
- ✅ Final image ≤ 25 MB
- ✅ `docker run -p 8080:8080 quicknotes:lab6` serves `/health` and `/notes`
- ✅ Size analysis in submission

### Task 2 (4 pts)
- ✅ `compose.yaml` defines quicknotes service + named volume + healthcheck
- ✅ Note POSTed survives `down && up`
- ✅ Volume vs `-v` distinction explained

### Bonus Task (2 pts)
- ✅ Container runs as nonroot, read-only root, dropped caps, no-new-privs
- ✅ Failed-shell / failed-write evidence captured
- ✅ Trivy scan run and documented

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Multi-stage Dockerfile | **6** | ≤ 25 MB image, distroless base, USER nonroot, smoke test works |
| **Task 2** — Compose + volume + healthcheck | **4** | Named volume, persistence verified, healthcheck wired |
| **Bonus** — 6 security defaults | **2** | All 6 applied, evidence of escape attempts blocked, Trivy ran |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **`CGO_ENABLED=1` produces a non-static binary** that won't run in distroless — set `CGO_ENABLED=0` explicitly
- 🪤 **Distroless has no shell** → no `docker exec ... sh` for debugging; use a debug image (`gcr.io/distroless/base-debian12:debug`) temporarily
- 🪤 **Volume mount path wrong** → notes don't persist. Verify with `docker volume inspect quicknotes-data`
- 🪤 **`compose.yaml` vs `docker-compose.yml`** — Compose v2 accepts both; prefer `compose.yaml`
- 🪤 **`USER nonroot` but volume owned by root** → container can't write. Either set permissions on the host directory or use a named volume (Docker handles uid)
- 🪤 **Image is 200 MB** → you forgot multi-stage and copied the toolchain in

---

## Guidelines

- Treat the Dockerfile as code: comment any non-obvious flag (`-trimpath`, `-ldflags='-s -w'`)
- Image size is a real cost — egress, registry storage, cold-start latency
- Healthchecks should be cheap and side-effect-free
- The Bonus is genuinely the SOC2-relevant lesson — these 6 defaults are why most container CVEs are non-exploitable

---

## Resources

- 📖 [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- 📖 [Compose specification](https://compose-spec.io/)
- 📖 [Google Distroless images](https://github.com/GoogleContainerTools/distroless)
- 📖 [Docker — Hardening best practices](https://docs.docker.com/engine/security/)
- 🎥 [Solomon Hykes 2013 Docker demo](https://www.youtube.com/watch?v=wW9CAH9nSLs)
- 🛠️ [`dive`](https://github.com/wagoodman/dive) — explore your image's layers interactively
