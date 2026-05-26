# Lab 9 — DevSecOps: Scan QuickNotes with Trivy + ZAP

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-DevSecOps-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Trivy%20%2B%20ZAP-informational)

> **Goal:** Scan the QuickNotes container image with Trivy. Run an OWASP ZAP baseline against the running app. Triage findings; fix at least one. Bonus: integrate `govulncheck` into the Lab 3 CI.
> **Deliverable:** A PR from `feature/lab9` to the course repo with security artifacts + `submissions/lab9.md`. Submit the PR link via Moodle.

---

## Overview

By the end of this lab you have:
- A Trivy report against `quicknotes:lab6` with an SBOM
- A ZAP baseline report against running QuickNotes
- A demonstrated fix for at least one ZAP-flagged finding (likely missing security headers)
- (Bonus) `govulncheck` running as part of your Lab 3 CI workflow

---

## Project State

**Starting point:** Lab 6 image exists; Lab 8 monitoring optional but helpful for the dashboard.

**After this lab:** A security gate exists in your CI; the QuickNotes binary has security headers; you can answer "am I affected by CVE-X" via SBOM.

---

## Prerequisites

- Docker (Lab 6)
- Lab 3 CI pipeline (for the Bonus)
- Optionally Lab 8 Compose stack to observe scan-time metrics

---

## Task 1 — Trivy: Image, Filesystem, SBOM (6 pts)

### 1.1: Scan the Lab 6 image

```bash
docker pull aquasec/trivy:0.59.1   # pin

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.59.1 image \
  --severity HIGH,CRITICAL \
  --no-progress \
  quicknotes:lab6 | tee trivy-image.txt
```

If you get HIGH/CRITICAL findings: read the description, look at fix versions, and write a **disposition** (fix now, accept with reason + date, watch).

### 1.2: Scan the repo filesystem

```bash
docker run --rm -v "$PWD:/repo" aquasec/trivy:0.59.1 fs \
  --severity HIGH,CRITICAL --no-progress /repo | tee trivy-fs.txt

docker run --rm -v "$PWD:/repo" aquasec/trivy:0.59.1 config /repo | tee trivy-config.txt
```

`trivy fs` reads Go modules + lockfiles. `trivy config` scans your Dockerfile + compose.yaml for misconfig (e.g., running as root, secrets in ENV).

### 1.3: Generate an SBOM

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD:/out" aquasec/trivy:0.59.1 sbom \
  --format cyclonedx \
  -o /out/quicknotes.sbom.cdx.json \
  image quicknotes:lab6
```

You now have a machine-readable inventory of every dependency.

### 1.4: Document

In `submissions/lab9.md`:
- Top of `trivy-image.txt`, `trivy-fs.txt`, `trivy-config.txt`
- The first 30 lines of `quicknotes.sbom.cdx.json`
- For *every* HIGH/CRITICAL finding: disposition (fix / accept / watch) + a one-line reason
- One paragraph: *with this SBOM, if Log4Shell-2 drops tomorrow, how do you check whether QuickNotes is affected?*

---

## Task 2 — OWASP ZAP Baseline + Fix a Finding (4 pts)

### 2.1: Run QuickNotes + ZAP

```bash
docker run -d --rm --name qn -p 8080:8080 quicknotes:lab6
sleep 2

docker run --rm \
  --network host \
  -v "$PWD:/zap/wrk:rw" \
  ghcr.io/zaproxy/zaproxy:2.16.0 \
  zap-baseline.py \
    -t http://localhost:8080 \
    -r zap-baseline.html \
    -J zap-baseline.json \
    -I    # don't fail the script on findings

docker stop qn
```

Open `zap-baseline.html` in your browser. Most findings will be missing security headers.

### 2.2: Triage every finding

For each ZAP finding, document in `submissions/lab9.md`:
- ID + name
- Risk level
- Affected URL
- Decision: **fix in code**, **accept with reason**, or **suppress with rationale**

### 2.3: Add security headers in code

Pick at least one finding to **fix in code**. The common winner is missing security headers — implement an HTTP middleware in `app/`:

```go
// app/middleware.go (new file)
package main

import "net/http"

func withSecurityHeaders(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("Referrer-Policy", "no-referrer")
        w.Header().Set("Content-Security-Policy", "default-src 'none'")
        h.ServeHTTP(w, r)
    })
}
```

Wire it into `main.go`:

```go
srv := &http.Server{
    Addr:              addr,
    Handler:           withSecurityHeaders(server.Routes()),   // ← wrap
    ReadHeaderTimeout: 5 * time.Second,
}
```

Add a unit test in `handlers_test.go`:

```go
func TestSecurityHeadersOnHealth(t *testing.T) {
    srv := newTestServer(t)
    rec := do(t, srv, http.MethodGet, "/health", nil)
    if got := rec.Header().Get("X-Content-Type-Options"); got != "nosniff" {
        t.Errorf("missing nosniff header: %q", got)
    }
}
```

> ⚠️ The test above won't pass unless you also call `withSecurityHeaders` in the test setup. Adjust your tests to use a shared handler factory or mount middleware in `Routes()` directly.

### 2.4: Re-scan and verify

Rebuild the image, re-run ZAP. The chosen finding should be **gone** from the report.

### 2.5: Document

In `submissions/lab9.md`:
- Triage table for every ZAP finding
- The code diff for the fix (or link to commit)
- Before/after ZAP report excerpts showing the finding gone
- 3-4 sentences: *which findings are real, which are false positives, and what's the cost of a "false alarm"?*

---

## Bonus Task — `govulncheck` in CI (2 pts)

### B.1: Run locally

```bash
cd app/
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

If clean, deliberately introduce a vulnerable dep to prove the check works:

```bash
# example: add an old, vulnerable yaml.v2
go get gopkg.in/yaml.v2@2.2.1
govulncheck ./...    # should report at least one finding
```

### B.2: Add to Lab 3 CI

Extend `.github/workflows/ci.yml`:

```yaml
  vulncheck:
    name: govulncheck
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<SHA>
      - uses: actions/setup-go@<SHA>
        with: { go-version: '1.24', cache: true }
      - working-directory: app
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          $(go env GOPATH)/bin/govulncheck ./...
```

### B.3: Document

In `submissions/lab9.md`:
- govulncheck local output (with and without the bad dep)
- CI run showing govulncheck blocking
- One paragraph: *how is govulncheck more useful than `trivy fs` for Go projects? (hint: reachability)*

---

## How to Submit

1. Scan reports (`trivy-*.txt`, `zap-baseline.html`, `quicknotes.sbom.cdx.json`) in your fork
2. Security-headers code change committed
3. `submissions/lab9.md` covers all attempted tasks
4. PR from `feature/lab9` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Trivy run on image, fs, and config
- ✅ SBOM generated (CycloneDX)
- ✅ Every HIGH/CRITICAL has a documented disposition

### Task 2 (4 pts)
- ✅ ZAP baseline run; report saved
- ✅ All findings triaged in a table
- ✅ At least one fix landed; before/after evidence

### Bonus Task (2 pts)
- ✅ govulncheck runs in CI
- ✅ Demonstrated catching a deliberately-bad dep
- ✅ Written reachability explanation

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Trivy scans + SBOM | **6** | All three Trivy modes ran, SBOM produced, dispositions documented |
| **Task 2** — ZAP triage + code fix | **4** | Findings triaged, ≥1 fix landed, before/after evidence |
| **Bonus** — govulncheck in CI | **2** | Local + CI working, reachability reasoning |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Trivy needs internet** — first run downloads the vuln DB (~200 MB)
- 🪤 **ZAP active scan instead of baseline** — *do not* run `zap-full-scan` against anything you don't own
- 🪤 **Security headers break clients** — `Content-Security-Policy: default-src 'none'` is *strict*. For QuickNotes (API-only) it's fine; for a real frontend you'd loosen it
- 🪤 **`govulncheck` says "no vulnerabilities"** — that's the goal; introduce a bad dep to prove the check actually runs
- 🪤 **`.trivyignore` instead of fixing** — use it for documented accepted findings only, with a date for re-evaluation
- 🪤 **SBOM committed to Git** — fine for a course; in production, you'd attach it to the release artifact, not the repo

---

## Guidelines

- Treat every finding as a decision: fix, accept, or watch. Document *all three* options for the same finding if relevant
- Don't dismiss "informational" ZAP findings — many become "high" once you look at them in context (e.g. missing CSP)
- For govulncheck, the *reachability* analysis is the value: a vulnerable module that your code doesn't actually call is much less urgent than one your hot path touches
- Document accepted findings with a date — re-evaluate them next semester

---

## Resources

- 📖 [Trivy documentation](https://trivy.dev/)
- 📖 [OWASP ZAP docs](https://www.zaproxy.org/docs/)
- 📖 [OWASP Top 10 — 2021](https://owasp.org/Top10/)
- 📖 [govulncheck — Go vulnerability checker](https://go.dev/blog/vuln)
- 📝 [Log4Shell incident summary (Sonatype)](https://blog.sonatype.com/log4shell-vulnerability-the-first-30-days)
- 📝 [Equifax 2017 — GAO report](https://www.gao.gov/products/gao-18-559)
- 🛠️ Optional tools: Syft, Grype, gitleaks, trufflehog
