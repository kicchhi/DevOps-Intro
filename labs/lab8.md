# Lab 8 — SRE & Monitoring: Golden Signals Dashboard + One Good Alert

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-SRE%20%2B%20Monitoring-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Prometheus%20%2B%20Grafana-informational)

> **Goal:** Configure Prometheus to scrape QuickNotes, write a Grafana dashboard for the four golden signals, define one good alert, write its runbook. Bonus: add a Checkly synthetic probe from outside.
> **Deliverable:** A PR from `feature/lab8` to the course repo with `monitoring/` + `docs/runbook/` + `submissions/lab8.md`. Submit the PR link via Moodle.

---

## Overview

This lab is **all** writing: you write the Prometheus config, the Grafana data-source + dashboard provisioning files, the dashboard panels, the alert rule, and the runbook. No copy-paste YAML in this spec.

By the end:
- `docker compose up` brings up QuickNotes + Prometheus + Grafana
- Grafana auto-loads your dashboard with the four golden signals
- One alert fires when error rate exceeds 5% for 5 minutes; its runbook is a real document
- *(Bonus)* A Checkly probe from 2+ regions watches the deployed QuickNotes

---

## Project State

**Starting point:** Lab 6 Compose runs QuickNotes; its `/metrics` already exposes Prometheus-format metrics.

**After this lab:** The Compose stack also runs Prometheus + Grafana; the dashboard is provisioned from JSON on startup.

---

## Prerequisites

- Lab 6 done (QuickNotes container image works)
- Docker + Compose v2
- *(Bonus)* Free Checkly account, or another synthetic monitor

---

## Task 1 — Prometheus + Grafana with a Provisioned Dashboard (6 pts)

### 1.1: Layout

You will produce:

```text
monitoring/
├── prometheus/
│   └── prometheus.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasource.yml
        └── dashboards/
            ├── dashboard.yml          (provider config)
            └── golden-signals.json    (the actual dashboard)
```

Plus an extension to your **existing** Lab 6 `compose.yaml` adding `prometheus` and `grafana` services.

### 1.2: Prometheus config requirements

`prometheus/prometheus.yml` MUST:

1. Set a `global.scrape_interval` of 15 s
2. Define **one** scrape job for QuickNotes
3. Target the QuickNotes service by its **Compose service name** + port (DNS resolves inside the Compose network)

### 1.3: Grafana provisioning requirements

Two YAML files under `grafana/provisioning/`:

1. **`datasources/*.yml`** — auto-provisions a Prometheus data source pointing at the Prometheus *Compose service*, set as default
2. **`dashboards/*.yml`** — tells Grafana where to load dashboards from on startup (a file path inside the container)

Then a **JSON dashboard** mounted to that path. The dashboard MUST have **four panels**, one per golden signal:

1. **Latency** — request duration (use the available histogram if QuickNotes exposes one; otherwise the synthetic `quicknotes_http_requests_total` rate as a proxy)
2. **Traffic** — `rate()` of total requests
3. **Errors** — ratio of 4xx + 5xx to total requests
4. **Saturation** — `quicknotes_notes_total` (a gauge) or another available signal

> 💡 You can build the dashboard *interactively* in the Grafana UI, then export it (Settings → JSON Model). Mount the exported JSON. This is the normal workflow.

### 1.4: Compose extension requirements

Add to your Lab 6 `compose.yaml`:

1. `prometheus` service:
   - Pinned image (`prom/prometheus:v3.x.y` — pick a real version, not `:latest`)
   - Mount `monitoring/prometheus/prometheus.yml` read-only into `/etc/prometheus/`
   - Publish port 9090 (so you can browse the Prometheus UI)
   - `depends_on: { quicknotes: { condition: service_healthy } }` (Lab 6 healthcheck pays off here)
2. `grafana` service:
   - Pinned image (`grafana/grafana:13.x.y`)
   - Mount `monitoring/grafana/provisioning` and `monitoring/grafana/dashboards` (your choice of paths) into `/etc/grafana/` and `/var/lib/grafana/dashboards` respectively
   - Set `GF_SECURITY_ADMIN_*` env vars (don't ship default credentials)
   - Publish port 3000
   - `depends_on: prometheus`

### 1.5: Design questions

- a) **Pull vs push:** Prometheus pulls. What does that mean for *which side* (Prometheus or QuickNotes) needs to be reachable? What's the failure mode if Prometheus can't reach QuickNotes?
- b) **`scrape_interval: 15s`** is a default. What query problems do you create by setting it to `5s`? To `5m`?
- c) **PromQL `rate()` vs `irate()` vs `delta()`** — which one is right for the Traffic panel and why?
- d) **Why provision Grafana from files** instead of clicking through the UI on every fresh stack?

### 1.6: Where to start

- 📖 [Prometheus — Getting started](https://prometheus.io/docs/prometheus/latest/getting_started/)
- 📖 [Prometheus — Scrape config reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- 📖 [PromQL examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- 📖 [Grafana — Provision data sources](https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources)
- 📖 [Grafana — Provision dashboards](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)
- 📖 [Compose — `depends_on` with `condition`](https://docs.docker.com/reference/compose-file/services/#depends_on)

### 1.7: Generate traffic + verify

After `docker compose up -d`, generate ~200 mixed requests against QuickNotes. Check:
- `http://localhost:9090/targets` shows `quicknotes` as `UP`
- `http://localhost:3000` shows your dashboard (auto-loaded) with non-trivial graphs

### 1.8: Document

In `submissions/lab8.md`:
- All four config files (paste or link)
- Screenshot of the Grafana dashboard with traffic
- `curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'` showing `"up"`
- Design questions a-d answered

---

## Task 2 — One Good Alert + Runbook (4 pts)

### 2.1: Alert requirements

Configure **one** Grafana alert rule (or Prometheus rule — either is acceptable) that:

1. Fires when **error ratio exceeds 5%** sustained for **5 minutes**
2. Carries a `severity: page` label
3. Carries an `annotation` linking to a runbook path inside this repo
4. Does **NOT** fire on a single 4xx burst (Lecture 8 hygiene: sustained, not single-event)

### 2.2: Runbook requirements

Write `docs/runbook/high-error-rate.md` with the following sections:

1. **What this alert means** — one sentence
2. **Triage steps** — ≥ 3 numbered steps, in order
3. **Mitigations** — ≥ 2 options to stop the bleeding fast
4. **Post-incident** — what to do after (link to Lecture 1 postmortem template)

A 3 AM on-call who has *never seen QuickNotes before* should be able to act from this runbook.

### 2.3: Trigger the alert deliberately

Hit QuickNotes with traffic that produces ≥ 5% errors sustained for ≥ 5 minutes (e.g., a script that fires `POST /notes` with malformed JSON every second alongside healthy traffic). Watch the alert transition `Normal → Pending → Firing`.

### 2.4: Design questions

- e) **Why "sustained for 5 minutes"** instead of "fire immediately on first bad request"?
- f) **Symptom alerts vs cause alerts:** the alert above is a symptom alert. What's an example of a *cause* alert someone might write for QuickNotes? Why is it worse?
- g) **Alert fatigue:** Lecture 8 cited it as the bigger danger than too few alerts. What's a quantitative threshold ("page X% of the time the user wasn't actually affected") that would mean your alert is too noisy?

### 2.5: Document

In `submissions/lab8.md`:
- Alert rule definition (screenshot or YAML)
- Screenshot of the alert in `Firing` state
- The full runbook
- Design questions e, f, g answered

---

## Bonus Task — Synthetic Monitoring from the Outside (2 pts)

### B.1: Goal

A robot probe from at least **2 distinct regions** polls a *publicly reachable* QuickNotes every minute and alerts if the response is bad.

### B.2: Requirements

1. Get a public URL for your QuickNotes — pick one:
   - `ngrok http 8080` (free tier, 1 tunnel)
   - `cloudflared tunnel --url http://localhost:8080` (Cloudflare; free)
   - Lab 10's Cloud Run deployment if you've done Lab 10
2. Configure a Checkly **API check** (or equivalent: Pingdom, AWS Route 53 health checks, Better Stack):
   - Frequency: 1 minute
   - From **at least 2 regions** (e.g., Frankfurt + Singapore)
   - Alert if HTTP status != 200 or response time > 2 s
3. Let it run for **≥ 30 minutes** to gather data

### B.3: Compare internal vs external

In `submissions/lab8.md`, compare what your **Prometheus** sees vs what **Checkly** sees over the same window:

| | Prometheus (inside the Compose net) | Checkly (from 2 regions) |
|--|---|---|
| Avg latency p50 | ? | ? |
| Avg latency p95 | ? | ? |
| Errors observed | ? | ? |

Then 4-5 sentences:
- What kind of failure would Checkly catch that Prometheus cannot?
- What kind would Prometheus catch that Checkly cannot?

---

## How to Submit

1. `monitoring/` directory + extended `compose.yaml` + `docs/runbook/high-error-rate.md` in your fork
2. `submissions/lab8.md` covers all attempted tasks
3. PR from `feature/lab8` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Prometheus scrapes QuickNotes (`up == 1`)
- ✅ Grafana auto-provisions dashboard with 4 golden-signal panels
- ✅ Non-trivial graphs after traffic
- ✅ All 4 design questions answered

### Task 2 (4 pts)
- ✅ Alert rule defined, sustained-breach gate present
- ✅ Runbook complete with all 4 sections
- ✅ Alert observed `Firing`
- ✅ Design questions e, f, g answered

### Bonus Task (2 pts)
- ✅ External probe configured from 2+ regions
- ✅ Internal vs external comparison table with real numbers
- ✅ Written failure-mode analysis

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Prom + Grafana + dashboard | **6** | Scraping works, 4-panel dashboard provisioned, design questions |
| **Task 2** — Alert + runbook + trigger | **4** | Sustained-breach alert, runbook complete, Firing observed |
| **Bonus** — Checkly synthetic | **2** | 2 regions, comparison, failure-mode analysis |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Prometheus `up == 0`** — the `targets:` host must match the Compose service name + the port QuickNotes listens on inside the container
- 🪤 **Dashboard panel "No data"** — wait 30 s after first start for the scrape; check `/targets` in the Prometheus UI
- 🪤 **Dashboard provisioning not loading** — JSON must be valid; check `docker compose logs grafana`. Also: the *provider* YAML's `path:` must match where you mount the JSON
- 🪤 **Alert never fires** — your expression must evaluate to a single scalar. Use Grafana's "Test query" button before saving
- 🪤 **Alert flaps** — every 30s on/off → "For:" duration too short, or your error-injecting script is too bursty
- 🪤 **Default Grafana password committed** — never. Set `GF_SECURITY_ADMIN_PASSWORD` to something unique; or set `GF_SECURITY_DISABLE_INITIAL_ADMIN_CREATION=true` and use OIDC

---

## Guidelines

- The 4 panels should be the *first* thing you'd look at during an incident — design them that way
- Alerts on **symptoms users see**, never on **cause metrics** like CPU
- Runbooks live next to the code they're about — `docs/runbook/` works; auto-link from the alert annotation
- For the bonus, ngrok free tier (1 tunnel) is enough — don't pay for this lab

---

## Resources

- 📕 *Site Reliability Engineering* (Google) — Chapters 3, 4, 6, 10 — [free online](https://sre.google/sre-book/table-of-contents/)
- 📗 *The SRE Workbook* — Chapter 5 (Alerting on SLOs) — [free online](https://sre.google/workbook/table-of-contents/)
- 📖 [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- 📖 [PromQL examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- 📖 [Grafana alerting docs](https://grafana.com/docs/grafana/latest/alerting/)
- 🛠️ [Checkly free tier](https://www.checklyhq.com/), [ngrok](https://ngrok.com/), [Cloudflared](https://github.com/cloudflare/cloudflared)
