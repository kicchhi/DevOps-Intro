# Lab 6 — Containers: Dockerize QuickNotes

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Containers-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Docker%2028.x-informational)

> **Goal:** Write a multi-stage `Dockerfile` that produces a ≤ 25 MB image of QuickNotes. Write a `compose.yaml` that runs it with healthcheck + persistent volume. Bonus: apply the 6 security defaults from Lecture 6 and prove they work.
> **Deliverable:** A PR from `feature/lab6` to the course repo with `app/Dockerfile` + `compose.yaml` + `submissions/lab6.md`. Submit the PR link via Moodle.

---

## Overview

You will not be handed the Dockerfile or compose.yaml. The skill of this lab is **writing them** from requirements + the Lecture 6 patterns + docs.

By the end you have:
- `app/Dockerfile` — multi-stage build, final image ≤ 25 MB
- `compose.yaml` at the repo root running QuickNotes
- A named volume that survives `docker compose down && up`
- *(Bonus)* The hardening defaults applied + proof the container can't escape its sandbox

---

## Project State

**Starting point:** QuickNotes builds with `go build` (Lab 1); CI runs (Lab 3); VM works (Lab 5).

**After this lab:** `docker compose up` is the one-command way to run the project.

---

## Prerequisites

- Docker **28.x** (Compose v2 built in)
- ≥ 4 GB free disk for image cache

---

## Task 1 — Multi-Stage Dockerfile, ≤ 25 MB (6 pts)

### 1.1: Requirements

Your `app/Dockerfile` MUST:

1. Be **multi-stage** — at least one *builder* stage and one *runtime* stage
2. The **builder** stage uses an official Go image pinned to `1.24` (any minor patch; not `:latest`)
3. The **runtime** stage uses a **distroless** or **scratch** base — no shell, no `apt`, no package manager
4. Build a **static** Go binary (`CGO_ENABLED=0`) — distroless static needs no dynamic linker
5. Strip the binary with `-ldflags='-s -w'` and use `-trimpath` for reproducibility
6. Run as **`nonroot`** (UID 65532 if you're using distroless's nonroot tag) — never as root
7. Set `ENTRYPOINT` (exec form: `["..."]`, not shell form), declare `EXPOSE 8080`
8. Final image weight: **≤ 25 MB** (verify with `docker images`)
9. Order Dockerfile lines to **maximize layer cache reuse** — `go.mod / go.sum` before the source

### 1.2: Design questions — answer in your submission

- a) **Why does layer-order matter?** Show before/after rebuild times for two strategies: `COPY . . && go mod download && go build` vs `COPY go.mod go.sum ./ && go mod download && COPY . . && go build`
- b) **Why `CGO_ENABLED=0`?** What happens in distroless-static if you forget it?
- c) **What is `gcr.io/distroless/static:nonroot`?** What's in it, what isn't, and why does that matter for CVEs?
- d) **`-ldflags='-s -w'` and `-trimpath`:** what does each flag do, and what's the cost?

### 1.3: Build + verify

After your Dockerfile is in place:

```bash
cd app/
docker build -t quicknotes:lab6 .
docker images quicknotes:lab6     # confirm ≤ 25 MB
docker run --rm -p 8080:8080 -v "$PWD/data:/data" quicknotes:lab6 &
sleep 2
curl -s http://localhost:8080/health
docker stop $(docker ps -q --filter ancestor=quicknotes:lab6)
```

### 1.4: Where to start

- 📖 [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- 📖 [Docker — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- 📖 [Google Distroless images](https://github.com/GoogleContainerTools/distroless) — read the README; pay attention to the **nonroot** tag and the `static` variant
- 📖 [Go cmd documentation — build flags](https://pkg.go.dev/cmd/go) (search for `-ldflags`, `-trimpath`)

### 1.5: Document

In `submissions/lab6.md`:
- Your Dockerfile (paste it or link)
- `docker images quicknotes:lab6` output
- `docker inspect quicknotes:lab6 | jq '.[0].Config'` excerpt (User, ExposedPorts, Entrypoint)
- The `golang:1.24-alpine` (or whatever you used) base image size for comparison
- Written answers to all 4 design questions in 1.2

---

## Task 2 — Compose + Healthcheck + Persistent Volume (4 pts)

### 2.1: Requirements

Your `compose.yaml` (at the **repo root**) MUST:

1. Define a service `quicknotes` that builds from `./app` and tags as `quicknotes:lab6`
2. Publish port `8080`
3. Define a **named volume** that mounts at `/data` inside the container — this is where QuickNotes' `notes.json` lives
4. Define a **healthcheck** for the `quicknotes` service. Distroless has no shell, so think about how a healthcheck even works in this image — your options are limited and you have to pick one
5. Pass the required env vars: `ADDR`, `DATA_PATH`, `SEED_PATH`
6. Include `restart: unless-stopped`

### 2.2: Design questions

- e) **Distroless has no shell. How do you healthcheck it?** Pick a strategy; explain. (Options: HTTP via a separate sidecar; `wget`-only debug image; rely on Docker's default behavior of just checking the process is alive; use a binary that's already in the image.)
- f) **Why does `volumes: [quicknotes-data:/data]` survive `docker compose down`?** And what *does* destroy it?
- g) **`depends_on` without `condition: service_healthy`** — what does it actually wait for? What's the bug it can cause?

### 2.3: Persistence test

```bash
docker compose up --build -d
sleep 3
curl -X POST -H 'Content-Type: application/json' \
  -d '{"title":"durable","body":"survive a restart"}' \
  http://localhost:8080/notes
curl -s http://localhost:8080/notes | grep durable    # exists

docker compose down                 # NOT `down -v`
docker compose up -d
sleep 3
curl -s http://localhost:8080/notes | grep durable    # must STILL exist ✅

docker compose down -v              # NOW the volume dies
docker compose up -d
sleep 3
curl -s http://localhost:8080/notes | grep durable    # gone
```

### 2.4: Document

In `submissions/lab6.md`:
- Your `compose.yaml`
- The 3-step persistence test output (note present → down → up → present → down -v → up → absent)
- Design questions e, f, g answered

---

## Bonus Task — The 6 Security Defaults (2 pts)

Lecture 6 listed six container-security defaults. Apply **all six** to your `quicknotes` service in `compose.yaml`, then **prove each works**.

### B.1: The 6 defaults

You must apply (and verify) at least each of:

1. **`USER nonroot`** in the Dockerfile (already from Task 1 — explicitly confirm here)
2. **Distroless or `scratch`** base (already from Task 1)
3. **Drop all Linux capabilities**, add only what you need (which for QuickNotes is **nothing**)
4. **Read-only root filesystem**, with a `tmpfs` mount for anywhere the app needs to write (other than your `/data` volume)
5. **`no-new-privileges`** security option
6. **Image scanned by Trivy in CI** (you'll wire this fully in Lab 9; for here, just run Trivy once and document)

### B.2: Verify each

For each of 1-5, produce evidence the constraint is *enforced*:

- **`USER nonroot`** → `docker inspect quicknotes:lab6 --format '{{ .Config.User }}'`
- **No shell available** → `docker compose exec quicknotes sh` should **fail**
- **Capabilities dropped** → `docker inspect <container> --format '{{ .HostConfig.CapDrop }}'` shows `[ALL]`
- **Read-only root** → `docker compose exec quicknotes touch /etc/test 2>&1` should fail (no shell to execute it; or via a debug image variant — describe how you tested)
- **`no-new-privileges`** → inspect `SecurityOpt`

### B.3: Run Trivy

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.59.1 image --severity HIGH,CRITICAL --no-progress \
  quicknotes:lab6
```

Capture the output. With distroless-static, the count is often **zero HIGH/CRITICAL** — that's the value of using a minimal base.

### B.4: Document

In `submissions/lab6.md`:
- Your hardened `compose.yaml` snippet (the `services.quicknotes` block)
- The 5 verification outputs from B.2
- The Trivy summary
- 3-4 sentences: *which of the 6 defaults gives you the most security per line of YAML?*

---

## How to Submit

1. `app/Dockerfile` + `compose.yaml` in your fork
2. `submissions/lab6.md` covers all attempted tasks
3. PR from `feature/lab6` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Multi-stage Dockerfile present
- ✅ Final image ≤ 25 MB
- ✅ Runs as nonroot, has `EXPOSE 8080`, distroless or scratch base
- ✅ `docker run` serves `/health` and `/notes`
- ✅ All 4 design questions answered

### Task 2 (4 pts)
- ✅ `compose.yaml` defines named volume, healthcheck, env, restart policy
- ✅ POSTed note survives `down && up`
- ✅ Design questions e, f, g answered

### Bonus Task (2 pts)
- ✅ All 6 security defaults applied
- ✅ Each verified with a specific command + captured output
- ✅ Trivy ran; output documented

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Multi-stage Dockerfile | **6** | ≤ 25 MB, distroless+nonroot, layer cache-friendly, design questions |
| **Task 2** — Compose + volume + healthcheck | **4** | Persistence verified, healthcheck strategy explained, design questions |
| **Bonus** — 6 hardening defaults | **2** | All 6 applied, each verified, Trivy summary |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **`CGO_ENABLED=1` (the default) → non-static binary** → distroless won't run it (`no such file or directory` because the dynamic linker is missing)
- 🪤 **Distroless has no shell** → `docker exec ... sh` fails. That's the feature, not a bug. For one-off debug use the `:debug` tag temporarily
- 🪤 **Image is 200 MB** → you forgot multi-stage; the toolchain leaked into the runtime
- 🪤 **`USER nonroot` but `data/` is owned by root** on the host → container can't write. Use named volumes (Docker handles ownership) or fix permissions
- 🪤 **`HEALTHCHECK` shell-form in a no-shell image** → check never runs. Use exec form or pick a tool that exists in the image
- 🪤 **Layer cache miss every rebuild** → you put `COPY . .` before `go mod download`. Order matters

---

## Guidelines

- Image size is a real cost (egress, registry storage, cold-start latency). Be ruthless
- Healthcheck must be *cheap* and *side-effect free*
- Treat the 6 defaults as a checklist for every production container you'll ever ship — they're not lab-only theater

---

## Resources

- 📖 [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)
- 📖 [Compose specification](https://compose-spec.io/)
- 📖 [Distroless images repo](https://github.com/GoogleContainerTools/distroless)
- 📖 [Docker — Security hardening](https://docs.docker.com/engine/security/)
- 📖 [`HEALTHCHECK` reference](https://docs.docker.com/reference/dockerfile/#healthcheck)
- 🛠️ [`dive`](https://github.com/wagoodman/dive) — interactive layer explorer
- 🛠️ [Trivy](https://trivy.dev/)
