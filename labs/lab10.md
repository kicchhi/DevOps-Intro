# Lab 10 — Cloud Computing: Ship QuickNotes to a Real Cloud

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Cloud%20%2B%20Serverless-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Cloud%20Run%20%2F%20Fly.io-informational)

> **Goal:** Push the QuickNotes image to a real registry, deploy it to Cloud Run (or Fly.io). Verify scale-to-zero. Bonus: compare cold-start latency across two platforms.
> **Deliverable:** A PR from `feature/lab10` to the course repo with `cloud/` + `submissions/lab10.md`. Submit the PR link via Moodle.

---

## Overview

This is the capstone of the main course — the QuickNotes image you've been building, hardening, and scanning since Lab 6 finally goes to the cloud.

By the end:
- Your image is pushed to a real registry (GHCR, Artifact Registry, or ECR)
- QuickNotes runs on Cloud Run (or Fly.io) at a public URL
- Scale-to-zero demonstrated
- (Bonus) Cold-start latency measured across two platforms

---

## Project State

**Starting point:** Lab 6 image works locally; Lab 9 has hardened it.

**After this lab:** A public URL serves your QuickNotes. The deploy is automated from CI (Bonus).

---

## Prerequisites

- One of:
  - GCP account (free tier — credit card may be required even for free use)
  - Fly.io account (alternative — works from more countries)
- Lab 3 CI workflow exists
- Lab 6 Dockerfile produces a clean image

---

## Task 1 — Push to a Real Registry (6 pts)

### Option A: GitHub Container Registry (free, OIDC-friendly)

```bash
echo $GITHUB_PAT | docker login ghcr.io -u YOUR_USERNAME --password-stdin

docker build -t ghcr.io/YOUR_USERNAME/quicknotes:v0.1.0 ./app
docker push ghcr.io/YOUR_USERNAME/quicknotes:v0.1.0
```

In the package's GitHub UI, make the package **public** so Cloud Run can pull without auth.

### Option B: Artifact Registry (GCP)

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT

# create the registry once
gcloud artifacts repositories create qn \
  --repository-format=docker \
  --location=europe-west4

gcloud auth configure-docker europe-west4-docker.pkg.dev

docker build -t europe-west4-docker.pkg.dev/$PROJECT/qn/quicknotes:v0.1.0 ./app
docker push     europe-west4-docker.pkg.dev/$PROJECT/qn/quicknotes:v0.1.0
```

### 1.4: Wire push into CI (recommended)

Extend `.github/workflows/ci.yml` with a `release` workflow triggered on tags:

```yaml
# .github/workflows/release.yml
name: release
on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write    # for ghcr.io
  id-token: write    # for OIDC

jobs:
  push:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<SHA>
      - uses: docker/login-action@<SHA>
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@<SHA>
        with:
          context: ./app
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/quicknotes:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}/quicknotes:latest
```

Tag a release:

```bash
git tag -a -s v0.1.0 -m "Lab 10 release"
git push origin v0.1.0
```

Watch the workflow push.

### 1.5: Document

In `submissions/lab10.md`:
- Registry URL + pulled-by-anyone evidence (`docker pull <URL>` from a clean machine or browser)
- The release workflow YAML
- A successful CI release run URL

---

## Task 2 — Deploy to Cloud Run (or Fly.io) (4 pts)

### Option A: Cloud Run

```bash
gcloud run deploy quicknotes \
    --image europe-west4-docker.pkg.dev/$PROJECT/qn/quicknotes:v0.1.0 \
    --region europe-west4 \
    --port 8080 \
    --memory 256Mi --cpu 1 \
    --max-instances 5 \
    --min-instances 0 \
    --allow-unauthenticated

# get URL
URL=$(gcloud run services describe quicknotes --region europe-west4 --format='value(status.url)')
curl -s "$URL/health"
```

### Option B: Fly.io

```bash
flyctl auth login
cd app/
flyctl launch \
  --name quicknotes-USERNAME \
  --image ghcr.io/YOUR_USERNAME/quicknotes:v0.1.0 \
  --internal-port 8080 \
  --region ams \
  --no-deploy

# edit fly.toml as needed, then:
flyctl deploy
flyctl open
```

### 2.3: Demonstrate scale-to-zero (Cloud Run)

```bash
# wait 5+ minutes with no traffic, then:
URL_TIME=$(curl -s -o /dev/null -w '%{time_total}' "$URL/health")
echo "first-after-idle: $URL_TIME"
URL_TIME=$(curl -s -o /dev/null -w '%{time_total}' "$URL/health")
echo "warm:            $URL_TIME"
```

Expected: the first request after idle is ~1-2 s; the warm request is ~50-100 ms.

### 2.4: Tear it down

```bash
# Cloud Run
gcloud run services delete quicknotes --region europe-west4 --quiet

# Fly.io
flyctl apps destroy quicknotes-USERNAME
```

### 2.5: Document

In `submissions/lab10.md`:
- Public URL (or screenshot if torn down)
- `gcloud run services describe` output
- Cold-vs-warm latency numbers
- One paragraph: *what does scale-to-zero buy you, and what does it cost?*

---

## Bonus Task — Cold-Start Comparison Across Platforms (2 pts)

Pick **two** platforms among Cloud Run, Fly.io, AWS Lambda (with custom runtime), and Render. Deploy the same QuickNotes image to both.

### B.1: Measure

Use `hyperfine` or a script:

```bash
# wait 10 min for both to scale to zero, then:
hyperfine --warmup 0 --runs 10 \
  'curl -s -o /dev/null https://qn-cloudrun.example.com/health' \
  'curl -s -o /dev/null https://qn-flyio.example.com/health'
```

Repeat for warm requests (after they're awake).

### B.2: Plot the distribution

Optional but nice: use `gnuplot` or matplotlib to plot the cold + warm distributions side-by-side. Capture as PNG.

### B.3: Document

In `submissions/lab10.md`:
- Numeric table: platform × (cold p50, cold p95, warm p50, warm p95)
- Plot if you made one
- 4-5 sentences: *what dominates the cold-start curve in each? what would change if your image were 200 MB instead of 15 MB?*

---

## How to Submit

1. Image pushed to a public registry (URL in the submission)
2. Deploy artifact (`fly.toml`, gcloud command in a script, etc.) committed to `cloud/`
3. `submissions/lab10.md` covers all attempted tasks
4. PR from `feature/lab10` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Image in a real registry; pullable
- ✅ CI workflow pushes on tag
- ✅ Tagged release exists with green CI run

### Task 2 (4 pts)
- ✅ QuickNotes responds at a public URL
- ✅ Cold + warm latency measured
- ✅ Scale-to-zero trade-off discussed

### Bonus Task (2 pts)
- ✅ Two platforms compared
- ✅ Numeric latency table
- ✅ Discussion of what dominates cold start

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Registry push (CI-automated) | **6** | Image in registry, release workflow, tagged release |
| **Task 2** — Cloud deploy | **4** | Public URL, scale-to-zero evidence, trade-off paragraph |
| **Bonus** — Cross-platform cold-start | **2** | Two platforms, numeric table, dominant-factor analysis |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Cloud Run can't pull from a private GHCR package** — make the package public or wire up GitHub OIDC → GCP
- 🪤 **`gcloud` quota errors** — new accounts get hit; raise the quota or use a smaller region
- 🪤 **`min-instances=1`** disables scale-to-zero — opposite of the Bonus measurement
- 🪤 **Multi-arch builds for Fly.io** — Fly.io runs `linux/amd64` by default; build accordingly
- 🪤 **Idle billing surprise** — Cloud Run with `min-instances=1` keeps a hot instance and **charges for it 24/7**
- 🪤 **Submitting before tearing down** — leave a teardown script so the next student can verify; or screenshot first, then destroy

---

## Guidelines

- Always set `--max-instances` to a small number for a course lab — runaway scaling is real money
- Treat the public URL as production: tag the image, sign it (Cosign — see Lecture 9 Bonus), use a CD pipeline
- Cold-start numbers depend on platform, region, image size, and language. Note the test machine and connection too
- The capstone is the *operationalization* of every prior lecture — your CI is the artifact, the image is the artifact, the deploy is the artifact

---

## Resources

- 📖 [Cloud Run — quickstart for containers](https://cloud.google.com/run/docs/quickstarts/build-and-deploy)
- 📖 [Fly.io docs — deploy from a Docker image](https://fly.io/docs/launch/from-docker/)
- 📖 [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- 📖 [AWS Lambda container image support](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
- 📝 [AWS us-east-1 Dec 2021 outage summary](https://aws.amazon.com/message/12721/)
- 🛠️ [`hyperfine`](https://github.com/sharkdp/hyperfine) — micro-benchmarking
- 🛠️ [`flyctl`](https://fly.io/docs/flyctl/), [`gcloud`](https://cloud.google.com/sdk/gcloud)
