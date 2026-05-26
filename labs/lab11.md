# Lab 11 — Bonus: Reproducible Builds of QuickNotes with Nix

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-Reproducible%20Builds%20%2F%20Nix-blue)
![points](https://img.shields.io/badge/points-10-orange)
![tech](https://img.shields.io/badge/tech-Nix%20Flakes%20%2B%20Go-informational)

> **Goal:** Write a Nix flake that builds QuickNotes reproducibly. Extend it to build a deterministic OCI image. Prove that two independent builds produce the same SHA-256 image digest.
> **Deliverable:** A PR from `feature/lab11` to the course repo with `flake.nix` (+ `flake.lock`) + `submissions/lab11.md`.

> 🎁 **Bonus lab.** 10 pts total, no Bonus row — Tasks 1 + 2 *are* the challenge.

---

## Overview

You will not be handed a flake. Read [Reading 11](../lectures/reading11.md) first; then write the flake from requirements + docs.

By the end:
- A `flake.nix` at the repo root builds the QuickNotes binary
- A second flake output builds a deterministic OCI image
- Two independent builds (different machines or `nix store gc`-ed clones) produce **identical** SHA-256 image digests

---

## Project State

**Starting point:** Lab 6 Docker image works. QuickNotes builds with `go build` (Lab 1).

**After this lab:** A flake at the repo root; reproducibility verified across two independent runs.

---

## Prerequisites

- Read [Reading 11](../lectures/reading11.md)
- Install Nix with Flakes enabled:
  - [Determinate Nix Installer](https://determinate.systems/posts/determinate-nix-installer/) (recommended)
- ≥ 8 GB free disk
- A second machine, fresh Docker container (`docker run -it nixos/nix bash`), or a colleague — for verifying reproducibility

---

## Task 1 — Reproducible Go Build via Nix Flake (6 pts)

### 1.1: Requirements

Your `flake.nix` at the **repo root** MUST:

1. Pin **nixpkgs** to a specific channel revision in `inputs:` (e.g. `nixos-24.11`)
2. Expose a package `quicknotes` (and `default`) that **builds the QuickNotes Go source from `app/`**
3. Use `buildGoModule` (or `buildGoApplication`, etc. — your choice; document why)
4. Set **`CGO_ENABLED = 0`** so the binary is static
5. Pin **`vendorHash`** (you'll get the value from the first failed build — paste it in)
6. Use **`-ldflags = [ "-s" "-w" ]`** for size + reproducibility (carried from Lab 6)
7. Expose a `devShell` with `go`, `gopls`, and `golangci-lint` so collaborators can `nix develop` into the project

The flake MUST commit cleanly together with **`flake.lock`** (auto-generated) so anyone cloning gets the exact same nixpkgs revision.

### 1.2: Verify reproducibility

The proof for Task 1 is that **two independent builds produce identical store hashes**:

```bash
# machine A (or first sandbox)
nix build .#quicknotes
nix-store --query --hash $(readlink result)
# e.g. sha256:abc123...

# machine B (or `docker run -it nixos/nix bash`, fresh clone)
git clone YOUR_FORK qn-fresh
cd qn-fresh
nix build .#quicknotes
nix-store --query --hash $(readlink result)
# MUST match machine A
```

### 1.3: Design questions

- a) **Why does `go build` not produce bit-identical outputs** on two machines, even from the same Git SHA? (Hint: timestamps, vendor resolution, build IDs.)
- b) **`vendorHash`** is a SHA over what, exactly? What happens if you set `vendorHash = null;`?
- c) **`flake.lock`** pins nixpkgs. Why is this the single most important file for reproducibility? What happens if you delete it before the second build?
- d) **`buildGoModule` vs `buildGoApplication`** — what's the difference? Which would you pick for QuickNotes and why?

### 1.4: Where to start

- 📖 [Nix Pills](https://nixos.org/guides/nix-pills/) — chapter 1-5 cover the model
- 📖 [Zero to Nix](https://zero-to-nix.com/) — Determinate's modern walkthrough
- 📖 [`buildGoModule` reference](https://ryantm.github.io/nixpkgs/languages-frameworks/go/) (nixpkgs section)
- 📖 [Flakes reference](https://nixos.wiki/wiki/Flakes)

### 1.5: Document

In `submissions/lab11.md`:
- Your `flake.nix` (paste; flake.lock can be linked)
- `nix build .#quicknotes` log excerpt
- Two `nix-store --query --hash` outputs from two independent environments — identical
- `./result/bin/quicknotes &` + `curl /health` proof it runs
- Design questions a-d answered

---

## Task 2 — Deterministic OCI Image (4 pts)

### 2.1: Requirements

Extend `flake.nix` to expose a `docker` (or similar) package using **`pkgs.dockerTools.buildImage`** that:

1. Produces an OCI image tarball containing the QuickNotes binary from Task 1
2. Sets the binary as `Entrypoint` (exec form)
3. Sets `ExposedPorts` to include `8080/tcp`
4. Runs as a `nonroot` user (carry forward Lab 6's discipline)
5. The image is built **without Docker** — only Nix tooling

### 2.2: Verify reproducibility

The proof for Task 2 is that **two independent builds produce identical SHA-256 image digests**:

```bash
# environment A
nix build .#docker
sha256sum result            # capture digest

# environment B
nix build .#docker
sha256sum result            # MUST match
```

### 2.3: Compare with Lab 6's Dockerfile build

Build the Lab 6 image fresh **twice** with `--no-cache`:

```bash
docker build --no-cache -t qn-lab6:run1 ./app
docker build --no-cache -t qn-lab6:run2 ./app
docker images --no-trunc qn-lab6
```

Typically the Lab 6 digests **differ** (timestamps in the layers).

### 2.4: Design questions

- e) **`dockerTools.buildImage` produces a deterministic image. What does Docker's `docker build` do** that introduces non-determinism, even from the same Dockerfile + Git SHA?
- f) **For a security auditor**, what can you prove with a reproducible image that you *cannot* prove with a signed-but-non-reproducible image?
- g) **What's the trade-off** of Nix's reproducibility? Why is `docker build` still the default for most teams?

### 2.5: Document

In `submissions/lab11.md`:
- The extended `flake.nix` snippet
- Image-size comparison: Nix-built vs Lab 6 Docker-built
- Two `sha256sum` outputs proving identical Nix digests
- The two `docker images --no-trunc` digests proving Lab 6 differs
- Design questions e, f, g answered

---

## How to Submit

1. `flake.nix` + `flake.lock` at the repo root
2. `submissions/lab11.md` covers both tasks
3. PR from `feature/lab11` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Flake builds QuickNotes via `nix build .#quicknotes`
- ✅ `./result/bin/quicknotes` runs and serves `/health`
- ✅ Two independent builds produce **identical** store hashes
- ✅ `flake.lock` committed
- ✅ Design questions a-d answered

### Task 2 (4 pts)
- ✅ `nix build .#docker` produces an OCI image, loadable via `docker load`
- ✅ Two independent builds produce identical SHA-256 tarball digests
- ✅ Comparison with non-reproducible Lab 6 image documented
- ✅ Design questions e, f, g answered

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Reproducible Go build | **6** | Flake correct, two-environment hash match, design questions |
| **Task 2** — Deterministic OCI image | **4** | Loadable image, two-environment digest match, vs-Lab 6 comparison |
| **Total** | **10** | (bonus lab — no Bonus row) |

> 📝 **No "Bonus Task" in this lab.** Lab 11 is itself a bonus lab — Task 1 + Task 2 *are* the challenge. The lab's full 10 pts contribute toward your bonus-labs grade weight (see the course [README](../README.md)).

---

## Common Pitfalls

- 🪤 **First build fails: `hash mismatch`** — that's Nix telling you the *correct* `vendorHash`. Paste the `got:` line, rerun
- 🪤 **`nix: command not found`** after install — open a new terminal so PATH refreshes
- 🪤 **Different hashes on two machines** — usually means `flake.lock` is not committed. The lockfile pins nixpkgs to a specific revision
- 🪤 **Out of disk** — Nix store grows. `nix store gc` reclaims unreferenced paths
- 🪤 **`nix build` requires internet on first run** — downloads pre-built artifacts from cache.nixos.org. Subsequent builds are mostly local
- 🪤 **WSL2 multi-user Nix is finicky** — use the Determinate installer; or single-user on WSL2

---

## Guidelines

- The reproducibility proof is the deliverable; the flake is just how you got there
- Pin everything: nixpkgs revision (via `flake.lock`), `vendorHash`, Go version
- For "two independent environments" the easiest path is `docker run --rm -it -v "$PWD:/repo" -w /repo nixos/nix bash`
- Once you have this, the natural next step is Cachix (shared binary cache) — out of scope but worth a follow-up project

---

## Resources

- 📕 [Nix Pills](https://nixos.org/guides/nix-pills/) — canonical intro
- 📕 [Zero to Nix](https://zero-to-nix.com/)
- 📗 [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/)
- 🎥 [Domen Kožar — *Boost your dev env with Nix*](https://www.youtube.com/watch?v=BdF6w3LkkdU)
- 📝 [Reproducible Builds project](https://reproducible-builds.org/)
- 📝 [Eelco Dolstra — original Nix paper (PhD thesis, 2004)](https://edolstra.github.io/pubs/phd-thesis.pdf)
