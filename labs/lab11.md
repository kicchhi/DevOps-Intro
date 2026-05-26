# Lab 11 — Bonus: Reproducible Builds of QuickNotes with Nix

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-Reproducible%20Builds%20%2F%20Nix-blue)
![points](https://img.shields.io/badge/points-10-orange)
![tech](https://img.shields.io/badge/tech-Nix%20Flakes%20%2B%20Go-informational)

> **Goal:** Build QuickNotes reproducibly with a Nix flake. Then build a deterministic OCI image of QuickNotes using `dockerTools.buildImage` and prove that two independent builds produce the same SHA-256 image digest.
> **Deliverable:** A PR from `feature/lab11` to the course repo with `nix/` (or `flake.nix` at root) + `submissions/lab11.md`. Submit the PR link via Moodle.

> 🎁 **This is a bonus lab.** Worth 10 pts, no Bonus row — Task 1 + Task 2 *are* the challenge.

---

## Overview

In Lab 6 you built a Docker image. It's tiny — but try rebuilding it from the same Git SHA on a colleague's laptop and the resulting image's SHA-256 digest will differ. **This lab fixes that.**

By the end:
- A `flake.nix` describes how to build QuickNotes; `nix build` produces a byte-for-byte identical binary every time
- A second flake output builds an OCI image deterministically; the digest is the same across machines
- You've experienced the broader payoff: *anyone* can audit your build by reproducing it

---

## Project State

**Starting point:** Lab 6 Docker image works. QuickNotes builds with `go build` (Lab 1).

**After this lab:** A Nix flake at the repo root; two `nix build` commands produce a binary and a Docker image; reproducibility verified across runs.

---

## Prerequisites

- Install Nix (multi-user) with Flakes enabled:
  - **Recommended:** [Determinate Nix Installer](https://determinate.systems/posts/determinate-nix-installer/)
  - On Ubuntu/Debian: `curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install`
  - On macOS: same installer; works under Rosetta or native ARM
- 8 GB free disk (Nix store can grow)
- One Nix-savvy classmate, or willingness to ssh into a fresh box, for the reproducibility verification

---

## Task 1 — Reproducible Go Build via Nix Flake (6 pts)

### 1.1: Initialize the flake

At the repo root:

```bash
nix flake init
# overwrite the default flake.nix with the content below
```

### 1.2: Write `flake.nix`

```nix
{
  description = "QuickNotes — DevOps-Intro reproducible build";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
  };

  outputs = { self, nixpkgs }:
    let
      system = "x86_64-linux";
      pkgs   = nixpkgs.legacyPackages.${system};
    in {
      packages.${system}.quicknotes = pkgs.buildGoModule {
        pname = "quicknotes";
        version = "0.1.0";
        src = ./app;
        vendorHash = null;          # fill in after first build
        CGO_ENABLED = 0;
        ldflags = [ "-s" "-w" ];
        proxyVendor = true;
      };

      packages.${system}.default = self.packages.${system}.quicknotes;

      devShells.${system}.default = pkgs.mkShell {
        packages = with pkgs; [ go gopls golangci-lint ];
      };
    };
}
```

### 1.3: First build (computes vendor hash)

```bash
git add flake.nix
nix build .#quicknotes
# fails with: hash mismatch ... got: sha256-XXXXXXXX
```

Copy the `got:` hash into `vendorHash` in `flake.nix`. Re-run:

```bash
nix build .#quicknotes
./result/bin/quicknotes &      # smoke test
sleep 1
curl -s http://localhost:8080/health
kill %1
```

### 1.4: Reproduce on a different machine (or fresh sandbox)

The key proof:

```bash
# on machine A
nix-store --query --hash $(readlink result)
# e.g.: sha256:abc123...

# on machine B (or after `nix store gc` and a fresh clone)
git clone YOUR_FORK quicknotes-fresh
cd quicknotes-fresh
nix build .#quicknotes
nix-store --query --hash $(readlink result)
# must match machine A
```

### 1.5: Document

In `submissions/lab11.md`:
- The full `flake.nix`
- The two machines' `nix-store --query --hash` outputs (identical)
- `nix build .#quicknotes` build log excerpt
- `./result/bin/quicknotes --help` (or a curl against it)
- One paragraph: *what gives Nix bit-for-bit determinism that `go build` doesn't?*

---

## Task 2 — Deterministic OCI Image via `dockerTools.buildImage` (4 pts)

### 2.1: Extend `flake.nix`

Add this output:

```nix
packages.${system}.docker = pkgs.dockerTools.buildImage {
  name = "quicknotes";
  tag = "v0.1.0";
  config = {
    Entrypoint = [
      "${self.packages.${system}.quicknotes}/bin/quicknotes"
    ];
    ExposedPorts = { "8080/tcp" = {}; };
    User = "nonroot";
  };
};
```

### 2.2: Build the image

```bash
nix build .#docker
# result is a symlink to the OCI image tarball
ls -lh result
# load into Docker
docker load < result
docker images quicknotes:v0.1.0
docker run --rm -p 8080:8080 quicknotes:v0.1.0 &
sleep 2
curl -s http://localhost:8080/health
docker kill $(docker ps -q --filter ancestor=quicknotes:v0.1.0)
```

### 2.3: Two-clone reproducibility test

```bash
# clone A
nix build .#docker
sha256sum result/*.tar.gz || sha256sum result    # capture digest

# clone B (or fresh sandbox)
git clone YOUR_FORK qn-fresh
cd qn-fresh
nix build .#docker
sha256sum result/*.tar.gz                          # must match
```

Capture both digests. They **must be identical**.

### 2.4: Compare with Lab 6's image

Build the Lab 6 Docker image fresh twice and capture two `docker image ls` digests:

```bash
docker build --no-cache -t qn-d:1 ./app
docker build --no-cache -t qn-d:2 ./app
docker images --no-trunc qn-d
```

Typically the Lab 6 digests **differ** (timestamps in layers).

### 2.5: Document

In `submissions/lab11.md`:
- The extended `flake.nix` snippet
- Image size comparison: Nix-built vs Lab 6 Docker-built
- Two digests proving reproducibility (or differences for Lab 6)
- A 4-5 sentence reflection: *with reproducible builds, what could you prove to a security auditor that you cannot prove today?*

---

## How to Submit

1. `flake.nix` (+ `flake.lock`) at the repo root
2. `submissions/lab11.md` covers both tasks
3. PR from `feature/lab11` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `flake.nix` builds QuickNotes via `nix build .#quicknotes`
- ✅ `./result/bin/quicknotes` runs and serves `/health`
- ✅ Two independent builds (different machines or `nix store gc`-ed clones) produce identical store hashes

### Task 2 (4 pts)
- ✅ `nix build .#docker` produces an OCI image loadable via `docker load`
- ✅ Two independent builds produce identical SHA-256 tarball digests
- ✅ Comparison with non-reproducible Lab 6 image documented

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Nix flake reproducible Go binary | **6** | Flake correct, two-machine hash match, vendorHash pinned |
| **Task 2** — Deterministic OCI image | **4** | Loadable image, two-clone digest match, comparison with Lab 6 |
| **Total** | **10** | (bonus lab — no Bonus row) |

> 📝 **No "Bonus Task" in this lab.** Lab 11 is itself a bonus lab — Task 1 + Task 2 *are* the challenge. The lab's full 10 pts contribute toward your bonus-labs grade weight (see the course README).

---

## Common Pitfalls

- 🪤 **First build fails: `hash mismatch`** — Nix is telling you what the correct `vendorHash` is. Copy/paste the `got:` line, rerun
- 🪤 **`nix: command not found`** after install — open a new terminal (your shell's PATH was updated)
- 🪤 **Different hashes on two machines** — usually means you forgot to `git add flake.lock`. The lockfile pins `nixpkgs` to a specific revision
- 🪤 **Out of disk** — Nix store grows. `nix store gc` reclaims unreferenced paths
- 🪤 **`nix build` requires internet on first run** — downloads cached binaries from cache.nixos.org. Subsequent builds are mostly local
- 🪤 **WSL2 + multi-user Nix can be janky** — use Determinate's installer; or stick to single-user on WSL2

---

## Guidelines

- Pin **both** `nixpkgs` (in `inputs`) and your binary's `vendorHash` — those two are the entire reproducibility story
- The first build is slow; subsequent are seconds because everything is cached
- For "two independent machines" you can also use Docker: `docker run --rm -it -v "$PWD:/repo" nixos/nix bash` provides a clean environment
- Once you have this, you can move toward Cachix or Attic to share the cache across CI runs (out of scope, but worth a follow-up project)

---

## Resources

- 📕 [Nix Pills](https://nixos.org/guides/nix-pills/) — the canonical intro
- 📕 [Zero to Nix](https://zero-to-nix.com/) — Determinate Systems' modern walkthrough
- 📗 [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/)
- 🎥 [Domen Kožar — *Boost your dev env with Nix*](https://www.youtube.com/watch?v=BdF6w3LkkdU)
- 📝 [Reproducible Builds project](https://reproducible-builds.org/)
- 📝 [Eelco Dolstra's PhD thesis (Nix paper)](https://edolstra.github.io/pubs/phd-thesis.pdf)
