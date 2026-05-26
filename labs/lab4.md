# Lab 4 — OS & Networking: Trace, Debug, and Read the Substrate

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-OS%20%2B%20Networking-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Linux%20%2B%20Go-informational)

> **Goal:** Use `ss`, `dig`, `tcpdump`, `curl -v`, and `journalctl` to deeply understand what happens when a single `POST /notes` flies across the network into QuickNotes.
> **Deliverable:** A PR from `feature/lab4` to the course repo with `submissions/lab4.md`. Submit the PR link via Moodle.

---

## Overview

Every prior incident in this course (Knight Capital, Facebook BGP, AWS us-east-1) had a substrate-layer failure mode. This lab makes the substrate visible:
- Capture and analyze real TCP packets to QuickNotes
- Walk an outside-in debugging chain on a deliberately-broken instance
- Build the muscle memory for "what's actually happening" at L3/L4/L7

---

## Project State

**Starting point:** QuickNotes runs locally on `:8080`; tools installed.

**After this lab:** You have a packet capture, a complete annotated debug trace, and (Bonus) a decoded TLS handshake.

---

## Prerequisites

- Linux or macOS terminal (Windows: use WSL2 for `tcpdump` + `ss`)
- Install: `tcpdump`, `iproute2` (provides `ss`, `ip`), `dnsutils` (provides `dig`, `host`), `mtr`, `htop`, `jq`
- Wireshark optional but useful (Bonus)
- `sudo` on your host machine (for `tcpdump`)

---

## Task 1 — Trace a Request End-to-End (6 pts)

### 1.1: Start QuickNotes + capture

In terminal A:

```bash
cd app/
go run .
```

In terminal B:

```bash
sudo tcpdump -i lo -nn -s 0 -A 'tcp port 8080' -w lab4-trace.pcap &
TCPDUMP_PID=$!
```

In terminal C, fire one request:

```bash
curl -v -X POST http://localhost:8080/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"trace me","body":"in flight"}'
```

Stop the capture:

```bash
sudo kill $TCPDUMP_PID
wait $TCPDUMP_PID 2>/dev/null
```

### 1.2: Decode the capture

```bash
# show packets in human format
sudo tcpdump -r lab4-trace.pcap -nn -A | tee lab4-trace.txt
```

Identify in the capture:
- The TCP three-way handshake (SYN → SYN/ACK → ACK)
- The HTTP request line (`POST /notes HTTP/1.1`) + the JSON body
- The HTTP response line (`HTTP/1.1 201 Created`) + the response JSON
- The connection close (`FIN` or `RST`)

### 1.3: Run the five debugging commands

For each, capture the output:

```bash
ss -tlnp   | grep :8080         # 1. what's listening?
ip route show                    # 2. routes from your host
mtr -rwc 5 localhost             # 3. reachability (loop on lo)
dig +short example.com @1.1.1.1  # 4. DNS works
journalctl --user -u quicknotes -n 20 || true  # 5. logs (if installed as service)
```

### 1.4: Document in `submissions/lab4.md`

- The annotated `lab4-trace.txt` (highlight handshake, HTTP req/resp, close)
- All five command outputs from 1.3
- One paragraph: *what would you check first if QuickNotes returned 502?*

---

## Task 2 — Outside-In Debugging on a Broken Deploy (4 pts)

A deliberately-broken QuickNotes branch is provided. Run it under `systemd-run --user` (or just `bash` with manual env vars), then debug it.

### 2.1: Run a broken instance

```bash
cd app/
# break it: bind to a port that's already taken
ADDR=:8080 go run . &
PID1=$!
sleep 1
ADDR=:8080 go run . 2>&1 | tee /tmp/qn-broken.log &
PID2=$!
sleep 2

# Use whichever you see — one process or both:
ps -ef | grep "go run" | grep -v grep
```

The second one should have failed to bind. Capture the exact error.

### 2.2: Walk the outside-in chain

For each of these steps, document command + output + decision:

```bash
# 1) systemctl-style: is it running?
ps -ef | grep quicknotes

# 2) is it listening?
ss -tlnp | grep 8080

# 3) reachable from host?
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health

# 4) firewall blocking?
sudo iptables -L -n -v 2>/dev/null || sudo nft list ruleset 2>/dev/null || true

# 5) DNS?
dig +short localhost
```

### 2.3: Repair + re-verify

Kill the conflicting first instance:

```bash
kill $PID1
sleep 1
ADDR=:8080 go run . &
sleep 1
curl -s http://localhost:8080/health
```

### 2.4: Document

In `submissions/lab4.md`:
- The full outside-in chain with the exact commands you ran
- The root cause (`bind: address already in use`)
- A mini-postmortem (≤ 200 words) framed blamelessly: what's *systemic* about this kind of failure, and what tooling could prevent it?

---

## Bonus Task — Decode the TLS Handshake (2 pts)

QuickNotes runs HTTP only — for this Bonus you'll point a TLS-terminating proxy at it.

### B.1: Add an HTTPS layer

Quickest path — use the public Caddy automatic-HTTPS reverse proxy with a self-signed local cert:

```bash
# install Caddy (linux):
sudo apt install caddy

# minimal Caddyfile in /etc/caddy/Caddyfile:
echo 'localhost:8443 {
  reverse_proxy localhost:8080
}' | sudo tee /etc/caddy/Caddyfile

sudo systemctl restart caddy
```

### B.2: Capture the TLS handshake

```bash
sudo tcpdump -i lo -nn -s 0 -w lab4-tls.pcap 'tcp port 8443' &
TCPDUMP_PID=$!

curl -vk https://localhost:8443/health

sudo kill $TCPDUMP_PID; wait $TCPDUMP_PID 2>/dev/null
```

### B.3: Decode with Wireshark

Open `lab4-tls.pcap` in Wireshark. Find the **ClientHello** and **ServerHello** packets. Capture screenshots of:
- ClientHello showing TLS version, cipher suites offered, SNI
- ServerHello showing chosen cipher + TLS version
- The certificate chain shown by `openssl s_client -connect localhost:8443 -showcerts </dev/null`

In `submissions/lab4.md`, annotate one screenshot: *which negotiation step kills TLS 1.0 / 1.1 in 2026?*

---

## How to Submit

1. `submissions/lab4.md` covers all attempted tasks
2. Include `lab4-trace.txt` (or excerpts) in your submission
3. PR from `feature/lab4` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Packet capture exists; you can identify the TCP handshake, HTTP request, HTTP response, connection close
- ✅ All five debugging commands produced sensible output
- ✅ Written 502-debug reflection (one paragraph)

### Task 2 (4 pts)
- ✅ Broken deploy reproduced; root cause identified
- ✅ Outside-in chain documented with command + output + decision at each step
- ✅ Blameless mini-postmortem (≤ 200 words)

### Bonus Task (2 pts)
- ✅ TLS handshake captured and decoded
- ✅ ClientHello + ServerHello + cert chain documented
- ✅ TLS 1.0/1.1 deprecation reasoning included

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Trace request, debug commands | **6** | Packet capture annotated, 5 commands with output, 502 reflection |
| **Task 2** — Outside-in debugging | **4** | Broken instance reproduced, chain walked, mini-postmortem |
| **Bonus** — TLS handshake decode | **2** | Capture + Wireshark screenshots + cert chain |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **`tcpdump: permission denied`** — use `sudo`; on Linux, the `pcap` group avoids this for power users
- 🪤 **`-i lo` shows nothing** — your traffic might be going via a different interface (Docker bridge, eth0). Try `tcpdump -i any` instead
- 🪤 **`ss -tlnp` shows no process name** — needs `sudo` to see PIDs you don't own
- 🪤 **Caddy refuses to start** — port 80 or 443 already taken by another process; pick custom ports
- 🪤 **Wireshark can't open `.pcap`** — capture as root, then `sudo chown $USER lab4-trace.pcap` before opening
- 🪤 **"It's never DNS" — until it is** — keep `dig` muscle memory; you'll need it for years

---

## Guidelines

- Capture once, analyze offline (`.pcap` files are reusable) — don't re-trigger the bug to look again
- For every debugging step, write down *why* you ran the command — not just what you ran
- Treat the bonus TLS exercise as preparation for Lab 9 (DevSecOps) — same handshake, different framing

---

## Resources

- 📖 [Brendan Gregg — Linux Performance](https://www.brendangregg.com/linuxperf.html) — the diagram is essential
- 📖 [tcpdump tutorial — Daniel Miessler](https://danielmiessler.com/p/tcpdump/)
- 📖 [Cloudflare's blog on the 2021 Facebook outage](https://blog.cloudflare.com/october-2021-facebook-outage/)
- 📖 [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- 🛠️ Tools cheat sheet: `ss`, `ip`, `dig`, `host`, `tcpdump`, `mtr`, `lsof -i`, `journalctl`
