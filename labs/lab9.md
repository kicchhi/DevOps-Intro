# Lab 9 — DevSecOps: Scan QuickNotes with Trivy + ZAP

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-DevSecOps-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Trivy%20%2B%20ZAP-informational)

> **Goal:** Scan the QuickNotes image with Trivy. Run OWASP ZAP baseline against the running app. Triage every finding; fix at least one in code. Bonus: add `govulncheck` to your Lab 3 CI as a PR gate.
> **Deliverable:** A PR from `feature/lab9` to the course repo with scan artifacts + code fix + `submissions/lab9.md`. Submit the PR link via Moodle.

---

## Overview

By the end:
- A **Trivy** report (image, filesystem, config) with every HIGH/CRITICAL triaged
- A **CycloneDX SBOM** of the QuickNotes image
- A **ZAP baseline** report; every finding triaged in a written decision (fix / accept / suppress)
- At least one finding **fixed in code** with the before/after evidence
- *(Bonus)* `govulncheck` runs in CI as a PR gate

The skill is **triage + decision** — not running tools. Anyone can run a scanner; the value is what you do with the output.

---

## Project State

**Starting point:** Lab 6 image exists (`quicknotes:lab6`); Lab 8 stack optional.

**After this lab:** Your CI has a security gate; QuickNotes ships with security headers; you can answer "am I affected by CVE-X?" via the SBOM.

---

## Prerequisites

- Lab 6 done (QuickNotes container)
- Lab 3 CI workflow (for the Bonus)
- Docker

---

## Task 1 — Trivy: Image + Filesystem + Config + SBOM (6 pts)

### 1.1: Required scans

Run all four — capture each output:

1. **Image scan** — `trivy image quicknotes:lab6` with `--severity HIGH,CRITICAL`
2. **Filesystem scan** of your repo — `trivy fs <repo>` with `--severity HIGH,CRITICAL`
3. **Config scan** — `trivy config <repo>` (scans Dockerfile, compose.yaml, etc. for misconfig)
4. **SBOM generation** — `trivy sbom --format cyclonedx` on the image, save the JSON

Pin Trivy to a specific version (e.g. `aquasec/trivy:0.59.x`) — not `:latest`.

### 1.2: Triage every HIGH/CRITICAL

For **every** HIGH or CRITICAL finding across the three scans, document a **disposition** with a single label + reason in `submissions/lab9.md`:

| Label | Means | Required documentation |
|-------|-------|------------------------|
| **FIX** | Patch / upgrade now | Link to PR / commit where you fixed it |
| **ACCEPT** | Risk is acceptable for now | Why it's acceptable + date by when you'd re-evaluate (≤ 6 months) |
| **WATCH** | Not actionable yet (no upstream fix) | What you're watching + when you'll re-check |
| **FALSE POSITIVE** | Scanner is wrong | Why it's wrong (with reasoning) |

The discipline is **a decision per finding**, not a yes/no on "did you scan".

### 1.3: Design questions

- a) **CVE severity is one input, not the answer.** What else (reachability, exploit availability, deployment context) matters when triaging?
- b) **Distroless images often show zero HIGH/CRITICAL.** Why is the minimal base the strongest single security control?
- c) **`.trivyignore`** lets you suppress findings. When is that the right move, and when is it security theater?
- d) **The SBOM** is a list of components. What concrete *future* problem does having it today solve? (Hint: Log4Shell, Lecture 9.)

### 1.4: Where to start

- 📖 [Trivy quickstart](https://trivy.dev/latest/getting-started/installation/)
- 📖 [Trivy — supported targets](https://trivy.dev/latest/docs/target/)
- 📖 [CycloneDX SBOM spec](https://cyclonedx.org/specification/overview/)

### 1.5: Document

In `submissions/lab9.md`:
- Top of each of the four scan outputs
- The triage **table** with every HIGH/CRITICAL + disposition
- The first 30 lines of your CycloneDX SBOM
- Design questions a-d answered

---

## Task 2 — OWASP ZAP Baseline + Fix at Least One Finding (4 pts)

### 2.1: Run ZAP baseline

Start QuickNotes (from Lab 6), then run `zap-baseline.py` against `http://localhost:8080`. **Do not** run the active scan; baseline is passive only. Save the HTML + JSON report.

Pin ZAP to a specific image tag (e.g. `ghcr.io/zaproxy/zaproxy:2.16.x`).

### 2.2: Triage every finding

For each ZAP finding, document in `submissions/lab9.md`:
- ID + name
- Risk level
- Affected URL / parameter
- Disposition (FIX / ACCEPT / SUPPRESS / FALSE POSITIVE) + reason

### 2.3: Pick one finding and fix it in code

The most common ZAP baseline findings are **missing HTTP security headers**. Pick one (or more) and fix it in the QuickNotes Go code:

Requirements for the fix:
1. Implement it as **middleware** that wraps the router — don't sprinkle `Header().Set` calls across handlers
2. Apply to **all** routes (not just `/health`)
3. Add at least one **unit test** that asserts the header is present on a response
4. The test must fail if the middleware is removed (so the fix is genuinely guarded)

### 2.4: Re-scan and prove the finding is gone

Rebuild the image. Re-run `zap-baseline.py`. The finding you fixed should **not** appear in the new report.

### 2.5: Design questions

- e) **Why a middleware** and not per-handler header sets?
- f) **`Content-Security-Policy: default-src 'none'`** is the strictest CSP. What does it break? Why is it OK for QuickNotes (an API) but not for a website?
- g) **False positives vs accepted findings:** ZAP often flags **informational** issues that aren't real problems. What's the cost of marking them all "accepted" without reading them?

### 2.6: Document

In `submissions/lab9.md`:
- Full triage table for every ZAP finding
- Code diff or PR link for your fix
- Before/after ZAP report excerpts proving the finding is gone
- Design questions e, f, g answered

---

## Bonus Task — `govulncheck` as a CI PR Gate (2 pts)

### B.1: Goal

Add `govulncheck` to your **Lab 3** CI workflow as a job that blocks the PR if it finds a vulnerability reachable from the QuickNotes call graph.

### B.2: Requirements

1. The job runs `govulncheck ./...` against `app/`
2. The Go version matches the rest of CI (1.24)
3. `govulncheck` is **pinned** (not `@latest`) — pick a version and reference it
4. The job has its own status check; failing it blocks the PR
5. Demonstrate the check **catches** something — temporarily add a known-vulnerable dependency, push, observe red, revert

### B.3: Design questions

- h) **Reachability** is `govulncheck`'s key idea. How is "this module has a CVE but we don't call the affected function" different from "this module has a CVE" — and what does that mean for triage workload?
- i) **`go install golang.org/x/vuln/cmd/govulncheck@<version>`** — why pin the version of the *scanner*, not just `@latest`?
- j) **govulncheck only knows about Go.** What's it *not* going to catch that Trivy (image scan) would?

### B.4: Document

In `submissions/lab9.md`:
- Your CI workflow's new job (paste or link)
- Screenshot or log of the "red" CI run when you introduced the vulnerable dep
- Screenshot or log of the "green" run after revert
- Design questions h, i, j answered

---

## How to Submit

1. Scan reports + SBOM committed to your fork (or in submission)
2. Security-headers code change committed; passing test
3. `submissions/lab9.md` covers all attempted tasks
4. PR from `feature/lab9` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ All four scans run; output captured
- ✅ Every HIGH/CRITICAL has a documented disposition
- ✅ CycloneDX SBOM generated
- ✅ Design questions a-d answered

### Task 2 (4 pts)
- ✅ ZAP baseline run; report saved
- ✅ Every finding triaged in a table
- ✅ ≥ 1 fix landed in code (middleware + test); before/after evidence
- ✅ Design questions e, f, g answered

### Bonus Task (2 pts)
- ✅ `govulncheck` job in CI
- ✅ Demonstrated catching a bad dep
- ✅ Design questions h, i, j answered

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Trivy scans + SBOM + triage | **6** | All scans + SBOM + per-finding disposition + design questions |
| **Task 2** — ZAP triage + code fix | **4** | Findings triaged, fix landed, before/after, design questions |
| **Bonus** — govulncheck CI gate | **2** | Job in CI, caught bad dep, design questions |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Trivy first run is slow** — downloads the vuln DB (~200 MB). Cache it in CI
- 🪤 **`zap-baseline.py` instead of `zap-full-scan.py`** — baseline is passive and safe; full is active and *can break things*. Never run active against anything you don't own
- 🪤 **CSP too strict for a real app** — for QuickNotes (API) it's fine; for a website, allowlist what you actually need
- 🪤 **`.trivyignore` instead of fixing** — only legitimate for documented, dated acceptances
- 🪤 **`govulncheck` shows "no vulnerabilities"** — that's the *goal*. To prove the check works, deliberately introduce a bad dep
- 🪤 **Triage table missing** — running scanners without acting on them is the most common DevSecOps theater. Don't fall into it
- 🪤 **Security headers break clients** — strict CSP can break Swagger UI if you add it later. Test before strict-locking

---

## Guidelines

- The lab measures **decision quality**, not scanner output volume
- For every finding you accept, set a *date* to re-evaluate. Without a date it's a permanent rug-pull
- The unit test that asserts your security header is present is what separates a "fix" from a "comment"
- govulncheck reachability vs Trivy module-presence is the most important distinction in this lab

---

## Resources

- 📖 [Trivy docs](https://trivy.dev/)
- 📖 [OWASP ZAP docs](https://www.zaproxy.org/docs/)
- 📖 [OWASP Top 10 — 2021](https://owasp.org/Top10/)
- 📖 [`govulncheck` — official tutorial](https://go.dev/blog/vuln)
- 📖 [Mozilla — Web Security guidelines](https://infosec.mozilla.org/guidelines/web_security)
- 📝 [Log4Shell incident summary (Sonatype)](https://blog.sonatype.com/log4shell-vulnerability-the-first-30-days)
- 📝 [Equifax 2017 — GAO report](https://www.gao.gov/products/gao-18-559)
- 🛠️ Optional: Syft (SBOM), Grype (vuln scan), gitleaks, trufflehog
