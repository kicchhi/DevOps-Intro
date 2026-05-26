# Lab 10 — Cloud Computing: Ship QuickNotes to a Real Cloud

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Cloud%20%2B%20Serverless-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Cloud%20Run%20%2F%20Fly.io-informational)

> **Goal:** Push the QuickNotes image to a real registry (automated from CI). Deploy it to a serverless container platform. Verify scale-to-zero. Bonus: compare cold-start latency across two platforms.
> **Deliverable:** A PR from `feature/lab10` to the course repo with `cloud/` + a CI release workflow + `submissions/lab10.md`. Submit the PR link via Moodle.

---

## Overview

The capstone. By the end:
- A tag on `main` triggers CI to build + push the QuickNotes image to a real registry
- The image runs on a public URL on Cloud Run (or Fly.io)
- Scale-to-zero demonstrated
- *(Bonus)* Cold-start latency measured across two platforms

You will not be handed `gcloud` commands or a `fly.toml`. The skill is **writing the deployment artifacts** from requirements + provider docs.

---

## Project State

**Starting point:** Lab 6 image works locally; Lab 9 has hardened it; Lab 3 CI runs.

**After this lab:** A tagged release produces a publicly-reachable QuickNotes URL via automated CI.

---

## Pick Your Platform

| Platform | Pick this if… |
|----------|---------------|
| **Google Cloud Run** | You have a GCP account + working payment method |
| **Fly.io** | You can't use GCP (sanctions, card issues). Accepts more payment methods + works from more countries |
| **AWS Lambda (container image)** | You have an AWS account and want to learn the FaaS variant |

State your choice in `submissions/lab10.md`.

---

## Prerequisites

- Cloud account on your chosen platform; CLI installed
- Lab 6 Dockerfile produces a clean image
- Lab 3 CI workflow exists

---

## Task 1 — CI-Automated Registry Push (6 pts)

### 1.1: Requirements

Add a **new** CI workflow (e.g. `.github/workflows/release.yml` for GH path; equivalent for GitLab) that:

1. **Triggers on push of a Git tag** matching `v*` (semver)
2. **Builds** the QuickNotes image from `app/`
3. **Pushes** to a public OCI registry — your choice of:
   - **`ghcr.io`** (recommended for the GH path; free, OIDC-friendly)
   - **AWS ECR public** (if you're on AWS)
   - **Google Artifact Registry** (if you're on GCP)
4. Image is tagged as both `<your version>` (e.g. `v0.1.0`) **and** `latest`
5. **Permissions are scoped** — only what's needed (e.g. `packages: write` for ghcr)
6. All third-party actions pinned by 40-char SHA (carrying forward from Lab 3)

The image must be **publicly pullable** after the workflow succeeds — `docker pull <URL>` from a clean machine works without auth.

### 1.2: Design questions

- a) **OIDC vs service-account JSON.** Why does CI prefer OIDC for cloud auth? What concrete attack does it prevent that JSON keys don't?
- b) **`:latest` tag vs `:v0.1.0` immutable tag** — Lab 6 already covered why `:latest` is mutable. So why do you still ship a `:latest` tag in production releases? When is it *the right call*?
- c) **`packages: write` scope on `GITHUB_TOKEN`.** What's the principle behind the scoped token, and what's at risk if you give the whole workflow `write: all`?

### 1.3: Where to start

- 📖 [GitHub Container Registry — Working with the registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- 📖 [`docker/build-push-action`](https://github.com/docker/build-push-action)
- 📖 [GitHub Actions — Authenticating with the registry](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
- 📖 [GitHub Actions — OIDC for cloud providers](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

### 1.4: Tag a release

```bash
git tag -a -s v0.1.0 -m "Lab 10 release"
git push origin v0.1.0
```

The workflow should fire, build, push. Verify by pulling the image from another machine (or your own laptop after `docker rmi`).

### 1.5: Document

In `submissions/lab10.md`:
- Your release workflow (paste or link)
- The registry URL where the image lives + evidence of a successful pull from a clean state
- A green CI release-run URL
- Design questions a, b, c answered

---

## Task 2 — Deploy to a Serverless Container Platform (4 pts)

### 2.1: Requirements

Deploy the image you pushed in Task 1 to your chosen platform with the following constraints:

1. **Region:** pick **one** (`europe-west4` / `ams` / wherever) and document the choice
2. **Resource limits:** CPU = 1; memory = 256-512 MiB (Lecture 5/10: right-size, don't over-provision)
3. **`max-instances` ≤ 5** — runaway autoscale on a course project is real money
4. **`min-instances = 0`** — scale-to-zero is the Lab 2 measurement target
5. The container's port `8080` is exposed publicly
6. **No long-lived cloud credentials** in the deploy step — use OIDC if your platform supports it (Cloud Run does via GH Actions; Fly.io has its own OIDC flow)
7. Authentication on the public URL: unauthenticated (`--allow-unauthenticated` or platform equivalent) so curl from your laptop works

Deploy artifacts go in `cloud/`:
- For GCP: a deploy script with `gcloud run deploy` + a `cloudbuild.yaml` if you use Cloud Build
- For Fly.io: `fly.toml` + a deploy script
- For AWS Lambda: SAM/CDK/Terraform config

The deploy must be runnable from `cloud/deploy.sh` (or equivalent) — a script another student could run from a fresh clone.

### 2.2: Demonstrate scale-to-zero

Pick **two** measurements:
1. **Warm latency:** make 5 consecutive requests; record p50 latency (`curl -w '%{time_total}'`)
2. **Cold latency:** wait ≥ 5 minutes idle, then a single request; record its latency

For Cloud Run, the second request after a warm period should be ~100 ms; the cold one ~1-2 s for a tiny Go image.

### 2.3: Tear it down

Document the teardown command(s). Leave the cluster as you found it (cost discipline).

### 2.4: Design questions

- d) **Region choice:** what trade-offs go into picking a region for QuickNotes? (Latency, data residency, cost, your own location)
- e) **`min-instances=0`** is what enables scale-to-zero. When would you set it to 1 instead, and what would that *cost*?
- f) **Cold start dominators:** for a tiny Go image, cold start is ~1-2 s. What changes if your image is 200 MB? 1 GB? What's the lever?

### 2.5: Document

In `submissions/lab10.md`:
- Your `cloud/` artifacts (paste or link)
- Public URL + curl output proof (or screenshot if torn down)
- Warm + cold latency measurements
- Design questions d, e, f answered

---

## Bonus Task — Cold-Start Across Two Platforms (2 pts)

### B.1: Goal

Deploy the **same** QuickNotes image to **two** of: Cloud Run, Fly.io, AWS Lambda (container image), Render. Measure cold + warm latencies on both.

### B.2: Requirements

1. Same image (same SHA) deployed to both
2. Same region geography where possible (both in EU, or both in US — apples to apples)
3. Use a measurement tool: `hyperfine`, `wrk`, `vegeta` (not a single `curl`)
4. Cold = ≥ 5 min idle prior; warm = ≥ 5 requests already served

### B.3: Table

| Metric             | Platform A (?) | Platform B (?) |
|--------------------|---------------:|---------------:|
| Image size         |              ? |              ? |
| Cold start p50     |              ? |              ? |
| Cold start p95     |              ? |              ? |
| Warm p50           |              ? |              ? |
| Warm p95           |              ? |              ? |

### B.4: Analysis

4-6 sentences:
- What dominates the cold-start curve on each? (Network pull, runtime init, your code's start logic)
- If you cut your image in half, which number would drop the most?
- If your team's product was latency-sensitive (< 100 ms p99), which platform would you pick — and what's the cost of that choice?

---

## How to Submit

1. Release CI workflow + `cloud/` directory in your fork
2. Tagged release exists on origin
3. `submissions/lab10.md` covers all attempted tasks
4. PR from `feature/lab10` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Tagged release triggers the workflow
- ✅ Image is in the chosen registry, publicly pullable
- ✅ All third-party actions SHA-pinned (carried from Lab 3)
- ✅ Design questions a-c answered

### Task 2 (4 pts)
- ✅ Public URL serves `/health` and `/notes`
- ✅ Resource limits enforced (≤ 1 CPU, ≤ 512 MiB, max-instances ≤ 5)
- ✅ Scale-to-zero observed (cold-start much slower than warm)
- ✅ Design questions d-f answered

### Bonus Task (2 pts)
- ✅ Same image deployed to 2 platforms
- ✅ Latency table populated from real measurements
- ✅ Written cold-start dominant-factor analysis

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Tag → CI → registry push | **6** | Workflow correct, image pullable, design questions |
| **Task 2** — Serverless deploy | **4** | Public URL, scale-to-zero evidence, design questions |
| **Bonus** — Cross-platform cold-start | **2** | Two platforms, latency table, dominant-factor analysis |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Image not public** — Cloud Run can't pull a private GHCR package without auth. Either make the package public or wire OIDC → Workload Identity
- 🪤 **`min-instances=1` "to avoid cold starts"** → eats the Bonus measurement and runs up the bill 24/7
- 🪤 **Workflow committed before the SHA pins** → tj-actions class of supply-chain vulnerability. Pin first, commit second
- 🪤 **Region accident** — deploying to `us-central1` while your laptop is in Europe makes warm latency look bad
- 🪤 **Forgot to tear down** — cloud bills can run for months on a forgotten test instance. Add `cloud/teardown.sh`
- 🪤 **Multi-arch image needed by some platforms** — Fly.io = linux/amd64 by default; Lambda needs arm64 in some configs. Read your provider's docs

---

## Guidelines

- Always cap `max-instances` to a small number for course exercises — runaway scaling is real money
- Tag the image, sign it (Cosign — see Lecture 9), use a CD pipeline. Treat this as production rehearsal
- Bonus measurements need a clean, repeatable test rig — note your machine, region, and time of day

---

## Resources

- 📖 [Cloud Run — Quickstart](https://cloud.google.com/run/docs/quickstarts/build-and-deploy)
- 📖 [Fly.io — Deploy from a Docker image](https://fly.io/docs/launch/from-docker/)
- 📖 [AWS Lambda — Container image support](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
- 📖 [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- 📖 [GitHub Actions — OIDC to Google Cloud](https://github.com/google-github-actions/auth)
- 📝 [AWS us-east-1 December 2021 outage summary](https://aws.amazon.com/message/12721/)
- 🛠️ [`hyperfine`](https://github.com/sharkdp/hyperfine), [`flyctl`](https://fly.io/docs/flyctl/), [`gcloud`](https://cloud.google.com/sdk/gcloud)
