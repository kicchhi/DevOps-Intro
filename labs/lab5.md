# Lab 5 — Virtualization: QuickNotes in a Vagrant VM

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Virtualization-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Vagrant%20%2B%20VirtualBox-informational)

> **Goal:** Build and run QuickNotes inside a Vagrant-defined VirtualBox VM. Demonstrate the snapshot/restore lifecycle. Compare VM resource cost vs container (Bonus).
> **Deliverable:** A PR from `feature/lab5` to the course repo with `submissions/lab5.md`. Submit the PR link via Moodle.

---

## Overview

By the end of this lab you have:
- A Ubuntu 24.04 VM defined in `Vagrantfile` — reproducible in one command
- QuickNotes installed and running inside the VM, reachable from your host on `:18080`
- A "clean" snapshot you can roll back to in 30 seconds
- (Bonus) Side-by-side baseline: VM vs Docker container for the same workload

This lab feeds Lab 7 directly — your Ansible playbook there will target this VM.

---

## Project State

**Starting point:** QuickNotes runs locally (Lab 1). Lab 3 CI passes.

**After this lab:** Your fork ships a `Vagrantfile` at the repo root; you have a running VM at `127.0.0.1:18080/health`; you've snapshotted, broken, and restored it.

---

## Prerequisites

- A laptop with **hardware virtualization** enabled (BIOS/UEFI). On Windows that means **Hyper-V off, VirtualBox on** (or use WSL2 + VBox 7.1.x with WSL VHCI)
- Install: VirtualBox **7.1.x**, Vagrant **2.4.x**
- ≥ 8 GB RAM on the host (VM uses 1-2 GB)

---

## Task 1 — Vagrant Up + Run QuickNotes Inside (6 pts)

### 1.1: Write the Vagrantfile

At the **repo root** (one level above `app/`), create `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box      = "ubuntu/jammy64"
  config.vm.hostname = "quicknotes-vm"

  # forward host :18080 → guest :8080
  config.vm.network "forwarded_port", guest: 8080, host: 18080, host_ip: "127.0.0.1"

  config.vm.synced_folder "./app", "/srv/quicknotes/src", type: "rsync"

  config.vm.provider "virtualbox" do |vb|
    vb.name   = "quicknotes-lab5"
    vb.memory = 1024
    vb.cpus   = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    set -euo pipefail
    apt-get update -y
    apt-get install -y --no-install-recommends curl ca-certificates
    # install Go 1.24 from the upstream tarball (pinned for the course)
    curl -fsSL https://go.dev/dl/go1.24.5.linux-amd64.tar.gz | tar -C /usr/local -xzf -
    echo 'export PATH=/usr/local/go/bin:$PATH' > /etc/profile.d/go.sh
  SHELL
end
```

### 1.2: Boot the VM

```bash
vagrant up                          # downloads box, boots, provisions (5-10 min first time)
vagrant ssh -c 'go version'         # → go1.24.5 linux/amd64
```

### 1.3: Build + run QuickNotes inside the VM

```bash
vagrant ssh
cd /srv/quicknotes/src
go build -o /tmp/quicknotes .
/tmp/quicknotes &
sleep 1
curl -s http://localhost:8080/health
```

Then from your **host**, hit it via the forwarded port:

```bash
curl -s http://localhost:18080/health  | python3 -m json.tool
curl -s http://localhost:18080/notes   | python3 -m json.tool
```

### 1.4: Document

In `submissions/lab5.md`:
- The first 10 lines of `vagrant up` output (the box download / provision)
- `vagrant ssh -c 'uname -a && go version'` output
- `curl` outputs from inside the VM and from the host (via port-forward)
- One paragraph: *what does the host see vs what the VM sees? How does NAT port-forwarding bridge them?*

---

## Task 2 — Snapshots: Break, Restore, Measure (4 pts)

### 2.1: Take a clean snapshot

```bash
vagrant snapshot save clean-quicknotes
vagrant snapshot list
```

### 2.2: Break the VM deliberately

Inside the VM:

```bash
vagrant ssh
sudo rm -rf /usr/local/go    # uh oh
exit
```

Confirm it's broken:

```bash
vagrant ssh -c 'go version' || echo "broken as expected"
```

### 2.3: Restore + measure

Time the restore:

```bash
time vagrant snapshot restore clean-quicknotes
vagrant ssh -c 'go version'        # should be back to 1.24.5
```

### 2.4: Delete the snapshot

```bash
vagrant snapshot delete clean-quicknotes
```

### 2.5: Document

In `submissions/lab5.md`:
- `vagrant snapshot list` output
- The "broken" evidence (go version failing)
- The `time` output for the restore
- A sentence: *why isn't this a substitute for a backup?*

---

## Bonus Task — VM vs Container: Resource Baseline (2 pts)

Same QuickNotes, two environments. Measure.

### B.1: VM baseline (already running)

While the VM is up and idle:

```bash
vagrant ssh -c 'free -h && ps -A --no-headers | wc -l'   # RAM, process count
# from your host:
VBoxManage list runningvms | head
time vagrant up                 # cold boot time (after `vagrant halt` first)
```

### B.2: Container baseline

```bash
docker run -d --name qn -p 28080:8080 \
  -v $PWD/app:/src -w /src golang:1.24 \
  sh -c 'go build -o /tmp/quicknotes && /tmp/quicknotes'

sleep 5
docker stats --no-stream qn       # CPU + MEM
docker inspect qn | jq '.[0].State.StartedAt'
```

(or use the Lab 6 Dockerfile if you've already done Lab 6.)

### B.3: Compare

Build a table in `submissions/lab5.md`:

| Dimension | Vagrant VM | Docker container |
|-----------|-----------|------------------|
| Cold start | XXs | XXs |
| Idle RAM | XX MB | XX MB |
| Image / box size on disk | XX MB | XX MB |
| Process count | XX | XX |

Write 4-5 sentences: *what does each model trade off, and when is each the right tool?*

---

## How to Submit

1. `Vagrantfile` exists at the repo root in your fork
2. `submissions/lab5.md` covers all attempted tasks
3. PR from `feature/lab5` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `Vagrantfile` checked in; `vagrant up` works clean from a fresh clone
- ✅ Inside-VM `curl` + host-side `curl :18080` both work
- ✅ Written explanation of NAT port-forwarding

### Task 2 (4 pts)
- ✅ Snapshot taken, deliberately broken, restored
- ✅ Restore time captured
- ✅ Snapshot-vs-backup distinction articulated

### Bonus Task (2 pts)
- ✅ VM + container resource numbers measured
- ✅ Comparison table in submission
- ✅ Written trade-off reasoning

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Vagrant + QuickNotes | **6** | Vagrantfile correct, build inside VM works, port-forward verified, host writeup |
| **Task 2** — Snapshot lifecycle | **4** | Save/break/restore documented, restore time measured, backup distinction |
| **Bonus** — VM vs container baseline | **2** | Numbers in a table, written trade-off |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **VirtualBox kernel modules missing** — on Ubuntu host: `sudo apt install virtualbox-dkms`. On Windows: ensure Hyper-V is *off* in features
- 🪤 **`vagrant up` hangs at "SSH auth"** — first boot can be slow on slow disks; give it 5 minutes before assuming it's stuck
- 🪤 **`synced_folder` requires rsync** — install `rsync` on your host
- 🪤 **Box download stalls** — try a different mirror: `config.vm.box_url = "https://..."`
- 🪤 **Snapshots eat disk** — they're copy-on-write but accumulate; delete when done
- 🪤 **`vagrant destroy` while you have uncommitted work in the VM** — sync changes out (rsync, scp) *first*

---

## Guidelines

- Commit only the `Vagrantfile`, not the `.vagrant/` directory (add to `.gitignore`)
- Pin the box version if reproducibility matters across the course
- For the bonus, run baselines on the *same* hardware — comparing your laptop to a friend's is meaningless
- The VM you build here is the deploy target for Lab 7's Ansible playbook — keep it healthy

---

## Resources

- 📖 [Vagrant — Getting Started](https://developer.hashicorp.com/vagrant/tutorials/getting-started)
- 📖 [VirtualBox 7.1 User Manual](https://www.virtualbox.org/manual/UserManual.html)
- 📖 [Brendan Gregg — *Containers vs VMs* (USENIX talk)](https://www.brendangregg.com/Slides/UMCloud2016_container_performance.pdf)
- 📝 [Cloudflare on Heartbleed](https://blog.cloudflare.com/answering-the-critical-question-can-you-get-private-ssl-keys-using-heartbleed/) — why "cattle, not pets" matters
- 🛠️ Tools: `vagrant`, `VBoxManage` CLI, `qemu-img`
