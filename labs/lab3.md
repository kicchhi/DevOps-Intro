# Lab 3 — CI/CD: GitHub Actions for QuickNotes (Bonus: GitLab CI Mirror)

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-CI%2FCD-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-GH%20Actions%20%2B%20GitLab%20CI-informational)

> **Goal:** Wire `vet + test + lint` as a required PR gate for QuickNotes. Cache the Go module download so subsequent runs are fast. Mirror the same pipeline to GitLab CI (Bonus).
> **Deliverable:** A PR from `feature/lab3` to the course repo with `.github/workflows/ci.yml` + `submissions/lab3.md`. Submit the PR link via Moodle.

---

## Overview

By the end of this lab, every PR you open against your fork:
- Runs `go vet ./...`
- Runs `go test -race -count=1 ./...`
- Runs `golangci-lint run`
- Blocks the PR if any of the three fail

You'll also pin third-party actions by full commit SHA (not by tag) — the tj-actions/changed-files lesson from Lecture 3.

---

## Project State

**Starting point:** QuickNotes runs locally; `app/` ships with `go.mod`, `Makefile`, `.golangci.yml`.

**After this lab:** A working GH Actions pipeline (and optionally a GitLab CI mirror) running on every PR to `main`. Branch protection requires CI to pass before merge.

---

## Prerequisites

- Lab 1 + Lab 2 complete
- Your fork's `main` is up to date with upstream
- (For Bonus) A GitLab account on `gitlab.pg.innopolis.university` or `gitlab.com`

---

## Task 1 — GitHub Actions: PR-Gated Build/Test/Lint (6 pts)

### 1.1: Write `.github/workflows/ci.yml`

In your fork, on `feature/lab3`:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read   # ✅ least privilege

jobs:
  test:
    name: vet + test
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<PIN_BY_SHA>      # v4.x.x
      - uses: actions/setup-go@<PIN_BY_SHA>      # v5.x.x
        with:
          go-version: '1.24'
          cache: true
      - working-directory: app
        run: go vet ./...
      - working-directory: app
        run: go test -race -count=1 ./...

  lint:
    name: golangci-lint
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<PIN_BY_SHA>
      - uses: actions/setup-go@<PIN_BY_SHA>
        with: { go-version: '1.24', cache: true }
      - uses: golangci/golangci-lint-action@<PIN_BY_SHA>  # v6.x.x
        with:
          version: 'v2.5.0'
          working-directory: app
```

**Replace `<PIN_BY_SHA>`** with the actual commit SHA of the version you want. Find the SHA on the action's GitHub releases page, copy the full 40-char hash. Add a trailing comment with the human-readable tag:

```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.2.2
```

### 1.2: Push + observe

```bash
git switch -c feature/lab3
git add .github/workflows/ci.yml
git commit -S -s -m "ci(lab3): vet + test + lint on every PR"
git push -u origin feature/lab3
```

Open a draft PR from `feature/lab3` → your fork's `main`. Watch CI run.

### 1.3: Force a failure, fix it

Deliberately break a test (e.g., change an expected value in `app/store_test.go`). Push. Confirm the PR is **red and blocked**. Restore the test. Push. Confirm green.

### 1.4: Enable branch protection

In your fork: **Settings → Branches → Add rule for `main`**:
- ☑️ Require status checks to pass before merging
- ☑️ Require branches to be up to date before merging
- ☑️ Required checks: `vet + test` and `golangci-lint`

### 1.5: Document

In `submissions/lab3.md`:
- Link to the green CI run
- A screenshot or log of the *failed* run from 1.3 + the fix commit
- The SHA-pinned `uses:` lines from your YAML

---

## Task 2 — Caching, Matrix, and Pipeline Performance (4 pts)

### 2.1: Measure baseline run-time

Run the CI on a fresh push **without** `cache: true`. Note the wall-clock time for the `test` job in the Actions UI.

### 2.2: Add a Go-version matrix

Update `test` to run against Go 1.23 and 1.24 in parallel:

```yaml
test:
  strategy:
    fail-fast: false
    matrix:
      go: ['1.23', '1.24']
  runs-on: ubuntu-24.04
  steps:
    - uses: actions/checkout@<SHA>
    - uses: actions/setup-go@<SHA>
      with:
        go-version: ${{ matrix.go }}
        cache: true
    - working-directory: app
      run: go vet ./... && go test -race -count=1 ./...
```

Commit, push. Now CI runs two jobs in parallel.

### 2.3: Path filter for docs-only PRs

Add `paths` to the `pull_request` trigger so CI doesn't run when nothing in `app/` or `.github/workflows/` changed:

```yaml
on:
  pull_request:
    branches: [main]
    paths:
      - 'app/**'
      - '.github/workflows/ci.yml'
```

### 2.4: Document

In `submissions/lab3.md`:
- Wall-clock times: baseline vs cached vs matrix
- The exact YAML changes for matrix + path filter
- One sentence: *why does pinning the runner to `ubuntu-24.04` (not `-latest`) protect you over a 6-month course?*

---

## Bonus Task — GitLab CI Mirror (2 pts)

For students using **GitLab** (or any student who wants to learn portability):

### B.1: Write `.gitlab-ci.yml`

```yaml
stages:
  - test
  - lint

variables:
  GO_VERSION: "1.24"

.go_image: &go_image
  image: golang:1.24-alpine

test:
  <<: *go_image
  stage: test
  cache:
    key:
      files: [app/go.sum]
    paths:
      - .go/pkg/mod/
  variables:
    GOPATH: "$CI_PROJECT_DIR/.go"
  before_script:
    - cd app
  script:
    - go vet ./...
    - go test -race -count=1 ./...

lint:
  image: golangci/golangci-lint:v2.5.0
  stage: lint
  script:
    - cd app
    - golangci-lint run
```

### B.2: Push to GitLab

Either mirror your fork to GitLab or work directly there:

```bash
git remote add gitlab git@gitlab.pg.innopolis.university:YOU/DevOps-Intro.git
git push gitlab feature/lab3
```

Open a Merge Request and watch CI run.

### B.3: Document

In `submissions/lab3.md`:
- Link to a successful GitLab CI run
- A side-by-side mapping of GH Actions concepts → GitLab CI concepts (workflow vs pipeline, job, stage, matrix, etc.)
- One sentence: *what part of the mapping surprised you?*

---

## How to Submit

1. `submissions/lab3.md` covers all attempted tasks
2. PR opened from `feature/lab3` → course repo's `main`
3. PR is green (your CI passing also runs on the course-repo side if mirroring)
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `.github/workflows/ci.yml` exists with vet + test + lint jobs
- ✅ All third-party actions pinned by 40-char SHA (with tag comment)
- ✅ Branch protection enabled on `main` requiring CI to pass
- ✅ Evidence of a deliberate failure being blocked, then fixed

### Task 2 (4 pts)
- ✅ Matrix build runs Go 1.23 + 1.24 in parallel
- ✅ `cache: true` in setup-go
- ✅ Path filter excludes docs-only PRs
- ✅ Wall-clock measurements captured

### Bonus Task (2 pts)
- ✅ `.gitlab-ci.yml` runs the equivalent vet + test + lint
- ✅ A green GitLab CI run linked
- ✅ Concept mapping between platforms documented

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — GH Actions PR gate | **6** | YAML present, actions pinned by SHA, branch protection, failure+fix evidence |
| **Task 2** — Cache + matrix + path filter | **4** | All three optimizations applied, timing comparison |
| **Bonus** — GitLab CI mirror | **2** | Green pipeline, side-by-side mapping |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **`actions/checkout@v4` instead of pinning by SHA** — works, but defeats the point of Task 1. tj-actions taught us this in March 2025
- 🪤 **`working-directory: app` missing** — Go commands fail because the module is in `app/`, not repo root
- 🪤 **`fail-fast: true` (the default)** in matrix → one fail cancels the others → you can't see which combo broke
- 🪤 **Branch protection on someone else's fork's `main`** — you can only protect *your* fork's `main`. The upstream course repo has its own protection
- 🪤 **`paths:` filter blocks CI entirely for the PR description-only edit** — that's *fine*; it's the intent
- 🪤 **GitLab CI YAML mis-indented** — GitLab is *very* strict about anchors (`<<: *name`); use the GitLab CI linter (`/ci/lint`) to check

---

## Guidelines

- Pin tool versions (`golangci-lint v2.5.0`, not "latest") — pinning protects you over months
- Treat CI YAML as code: review changes in PR, don't push directly to main
- The "wall-clock time" numbers matter — if a PR-gate takes 4 minutes, engineers stop using it locally
- Caching is what makes "vet + test + lint" feel free; without it, every PR is a coffee break

---

## Resources

- 📖 [GitHub Actions — Securing your use of third-party actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)
- 📖 [tj-actions/changed-files supply-chain incident postmortem](https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised)
- 📖 [GitLab CI — Quick start guide](https://docs.gitlab.com/ee/ci/quick_start/)
- 🛠️ [`pinact`](https://github.com/suzuki-shunsuke/pinact) — automatic SHA-pinning for your workflow files
- 🛠️ [`act`](https://github.com/nektos/act) — run GitHub Actions locally before pushing
