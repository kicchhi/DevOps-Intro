# Lab 8 — SRE & Monitoring: Golden Signals Dashboard + One Good Alert

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-SRE%20%2B%20Monitoring-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Prometheus%20%2B%20Grafana-informational)

> **Goal:** Scrape QuickNotes' `/metrics`, build a Golden Signals dashboard in Grafana, write one good alert. Bonus: add an external Checkly probe.
> **Deliverable:** A PR from `feature/lab8` to the course repo with `monitoring/` + `submissions/lab8.md`. Submit the PR link via Moodle.

---

## Overview

By the end of this lab:
- Prometheus runs alongside QuickNotes (Compose) and scrapes `/metrics` every 15 s
- Grafana auto-provisions a dashboard with the four golden signals: rate, error %, p95 latency, saturation
- An alert fires when error rate exceeds 5% for 5 minutes
- (Bonus) Checkly polls your QuickNotes from outside

---

## Project State

**Starting point:** Lab 6 complete — `compose.yaml` runs QuickNotes.

**After this lab:** `docker compose up` brings up QuickNotes + Prometheus + Grafana. The Grafana UI on `:3000` shows your dashboard.

---

## Prerequisites

- Lab 6 complete (QuickNotes container available)
- Docker + Compose v2
- (Bonus) Checkly free-tier account, or another synthetic-monitoring tool

---

## Task 1 — Prometheus + Grafana Provisioned Dashboard (6 pts)

### 1.1: Lay out monitoring config

```text
monitoring/
├── prometheus/
│   └── prometheus.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasource.yml
        └── dashboards/
            ├── dashboard.yml
            └── golden-signals.json
```

### 1.2: Prometheus config

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: quicknotes
    static_configs:
      - targets: ['quicknotes:8080']
```

### 1.3: Grafana data source provisioning

```yaml
# monitoring/grafana/provisioning/datasources/datasource.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

```yaml
# monitoring/grafana/provisioning/dashboards/dashboard.yml
apiVersion: 1
providers:
  - name: 'default'
    folder: ''
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

### 1.4: Extend `compose.yaml`

Add to the existing Compose file:

```yaml
services:
  # ... quicknotes service from Lab 6 ...

  prometheus:
    image: prom/prometheus:v3.1.0
    ports: ["9090:9090"]
    volumes:
      - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    depends_on:
      quicknotes:
        condition: service_healthy

  grafana:
    image: grafana/grafana:11.4.0
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: lab8devops
      GF_AUTH_ANONYMOUS_ENABLED: "false"
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      - prometheus
```

### 1.5: Golden Signals dashboard

Save the following as `monitoring/grafana/dashboards/golden-signals.json` (skeleton — fill in the queries):

```json
{
  "title": "QuickNotes — Golden Signals",
  "uid": "qn-golden",
  "schemaVersion": 39,
  "panels": [
    { "title": "Rate (req/s)",        "type": "timeseries",
      "targets": [{ "expr": "rate(quicknotes_http_requests_total[1m])" }] },
    { "title": "Error % (5xx + 4xx)", "type": "stat",
      "targets": [{ "expr": "sum(rate(quicknotes_http_responses_by_code_total{code=~\"5..|4..\"}[5m])) / sum(rate(quicknotes_http_requests_total[5m]))" }] },
    { "title": "Notes total (gauge)", "type": "stat",
      "targets": [{ "expr": "quicknotes_notes_total" }] },
    { "title": "Notes created/deleted", "type": "timeseries",
      "targets": [
        { "expr": "rate(quicknotes_notes_created_total[5m])" },
        { "expr": "rate(quicknotes_notes_deleted_total[5m])" }
      ]}
  ]
}
```

> 💡 You can also build the dashboard in the Grafana UI and export it (Settings → JSON Model) — that's a fine workflow

### 1.6: Boot the stack and generate traffic

```bash
docker compose up -d
sleep 5
# generate some traffic
for i in {1..200}; do
  curl -s http://localhost:8080/notes > /dev/null
  curl -s -X POST -H 'Content-Type: application/json' \
    -d "{\"title\":\"note-$i\",\"body\":\"x\"}" \
    http://localhost:8080/notes > /dev/null
done
```

### 1.7: Document

In `submissions/lab8.md`:
- Screenshot of the Grafana dashboard with non-trivial graphs
- `curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'` showing `"up"`
- Brief explanation of each panel

---

## Task 2 — One Good Alert (4 pts)

### 2.1: Configure alerting in Grafana

In Grafana UI: **Alerting → Alert rules → New alert rule**:

- **Name:** `quicknotes-high-error-rate`
- **Query:** `sum(rate(quicknotes_http_responses_by_code_total{code=~"5..|4.."}[5m])) / sum(rate(quicknotes_http_requests_total[5m]))`
- **Condition:** `IS ABOVE 0.05` (5%)
- **For:** 5m (sustained breach)
- **Labels:** `severity: page`
- **Annotations:** `runbook: docs/runbook/high-error-rate.md`

Save + activate.

### 2.2: Write the runbook

```markdown
# docs/runbook/high-error-rate.md

## What this alert means
QuickNotes is returning ≥ 5% non-2xx responses for the last 5 minutes.

## Triage steps
1. Look at the Golden Signals dashboard — which code class dominates?
2. Check `journalctl -u quicknotes -n 100` for stack traces
3. If 5xx — recent deploy? Roll back to previous tag
4. If 4xx — client misuse? Check Lab 9's WAF / rate-limiting

## Mitigations
- Roll back: `git revert <SHA> && deploy`
- Scale up: `docker compose up -d --scale quicknotes=3`
- Disable failing endpoint: feature flag (out of scope for intro)

## Post-incident
Write a blameless postmortem (Lecture 1 template).
```

### 2.3: Trigger the alert deliberately

Use a tool to fire bad requests:

```bash
# 100 requests, half deliberately broken (invalid JSON)
for i in {1..50}; do
  curl -s -X POST -H 'Content-Type: application/json' \
    -d 'INVALID' http://localhost:8080/notes > /dev/null
  curl -s http://localhost:8080/notes > /dev/null
done
```

Watch the Grafana alert dashboard. Within 5-6 minutes you should see the alert transition `Pending` → `Firing`.

### 2.4: Document

In `submissions/lab8.md`:
- Screenshot of the alert rule definition
- Screenshot of the alert in `Firing` state
- The full `docs/runbook/high-error-rate.md`
- 4-5 sentences: *why is this alert "good" by Lecture 8's hygiene criteria? what would make it noisy?*

---

## Bonus Task — Synthetic Monitoring with Checkly (2 pts)

### B.1: Expose QuickNotes publicly (briefly)

Use ngrok or Cloudflared tunnel to get a public URL for your local QuickNotes:

```bash
# ngrok (sign up free)
ngrok http 8080
# or via Cloudflared
cloudflared tunnel --url http://localhost:8080
```

### B.2: Configure a Checkly probe

In Checkly:
- Create an HTTP check against `https://YOUR_TUNNEL.ngrok.io/health`
- Frequency: 1 min
- Locations: pick **2** distant regions (e.g., Frankfurt + Singapore)
- Alert on: status != 200 or response time > 2 s

### B.3: Compare internal vs external view

After 30 minutes, compare:
- Prometheus inside the Compose network: error rate, latency
- Checkly's view: same metrics from outside

### B.4: Document

In `submissions/lab8.md`:
- Screenshot of Checkly results
- Side-by-side: internal Prometheus latency vs Checkly's latency from 2 regions
- One paragraph: *what kind of failure would Checkly catch that Prometheus can't?*

---

## How to Submit

1. `monitoring/` directory + extended `compose.yaml` in your fork
2. `docs/runbook/high-error-rate.md`
3. `submissions/lab8.md` covers all attempted tasks
4. PR from `feature/lab8` → course repo's `main`
5. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Prometheus scrapes QuickNotes (`up == 1`)
- ✅ Grafana provisions dashboard from JSON on startup
- ✅ Dashboard shows non-trivial graphs (after traffic generation)

### Task 2 (4 pts)
- ✅ Alert rule defined; runbook documented
- ✅ Alert triggered + transitioned to Firing
- ✅ "Good alert" reasoning per Lecture 8 criteria

### Bonus Task (2 pts)
- ✅ Checkly probe configured from 2+ regions
- ✅ Internal vs external comparison documented
- ✅ Synthetic-vs-internal failure-mode reasoning

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Prom + Grafana + dashboard | **6** | Scraping works, dashboard provisioned + populated, 4 panels meaningful |
| **Task 2** — Alert + runbook + trigger | **4** | Rule + runbook in repo, fired in `Firing`, hygiene reasoning |
| **Bonus** — Checkly synthetic | **2** | External probe, comparison, failure-mode reasoning |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **Prometheus `up == 0`** — service name in `targets:` must match Compose service name + port
- 🪤 **Grafana shows "No data"** — give Prometheus 30 seconds to scrape after first start; check the targets page
- 🪤 **Dashboard not auto-provisioned** — JSON must be valid; check Grafana logs (`docker compose logs grafana`)
- 🪤 **Alert never fires** — the *expression* might not evaluate to a single scalar; use the Grafana "Test query" button
- 🪤 **Alert fires immediately, then resolves, then fires again (flapping)** — increase the `For:` duration
- 🪤 **`docker compose down -v`** wipes the Grafana database — but provisioning is YAML-only, you don't lose dashboards

---

## Guidelines

- The four panels should be useful at a glance — if you'd actually run an incident with this dashboard, it's good
- Alerts must fire on **symptoms** users see, not on **causes** (don't alert on CPU)
- The runbook must be specific enough that a 3 a.m. on-call who hasn't seen the system can act
- For the bonus, ngrok's free tier gives you 1 tunnel — that's enough

---

## Resources

- 📕 *Site Reliability Engineering* (Google) — Chapters 3, 4, 6, 10 — [free](https://sre.google/sre-book/table-of-contents/)
- 📗 *The SRE Workbook* — Chapter 5 (alerting) — [free](https://sre.google/workbook/table-of-contents/)
- 📖 [Prometheus query examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- 📖 [Grafana alerting docs](https://grafana.com/docs/grafana/latest/alerting/)
- 🛠️ [Checkly free tier](https://www.checklyhq.com/), [ngrok](https://ngrok.com/), [Cloudflared](https://github.com/cloudflare/cloudflared)
