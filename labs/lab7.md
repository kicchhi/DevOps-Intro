# Lab 7 — Configuration Management: Deploy QuickNotes via Ansible

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Config%20Management-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Ansible%2010.x-informational)

> **Goal:** Deploy QuickNotes to your Lab 5 VirtualBox VM via an Ansible playbook. Demonstrate idempotency. Bonus: wire `ansible-pull` for a GitOps-style auto-converge.
> **Deliverable:** A PR from `feature/lab7` to the course repo with `ansible/` + `submissions/lab7.md`. Submit the PR link via Moodle.

---

## Overview

This is the lab that proves the cattle-vs-pets pattern in practice:
- A `playbook.yaml` *is* the recipe for the deploy
- Re-running it is safe (idempotency)
- The Bonus task makes the VM auto-converge from Git every 5 minutes — a true GitOps loop on a VM

---

## Project State

**Starting point:** Lab 5 VM exists; `vagrant up` works.

**After this lab:** `ansible-playbook ansible/playbook.yaml -i ansible/inventory.ini` deploys QuickNotes idempotently. Re-runs show `changed=0`.

---

## Prerequisites

- Lab 5 VM running (`vagrant up`)
- Ansible **10.x** on your host (`ansible --version` — requires Python 3.11+)
- Pre-built QuickNotes binary in `ansible/files/quicknotes` (built statically: `cd app && CGO_ENABLED=0 go build -o ../ansible/files/quicknotes .`)

---

## Task 1 — Idempotent Deploy to the Lab 5 VM (6 pts)

### 1.1: Lay out the Ansible directory

```text
ansible/
├── inventory.ini
├── playbook.yaml
├── files/
│   └── quicknotes               # static Go binary
└── templates/
    └── quicknotes.service.j2
```

### 1.2: Inventory targeting the Vagrant VM

```ini
# ansible/inventory.ini
[quicknotes_vm]
qn-vm ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key
```

Find your VM's actual SSH port:

```bash
vagrant ssh-config
# copy IdentityFile and Port into your inventory if different
```

### 1.3: The playbook

```yaml
# ansible/playbook.yaml
- name: Deploy QuickNotes
  hosts: quicknotes_vm
  become: true

  vars:
    listen_addr: ":8080"
    data_dir: /var/lib/quicknotes

  tasks:
    - name: Create system user
      user:
        name: quicknotes
        system: true
        shell: /usr/sbin/nologin
        home: "{{ data_dir }}"
        create_home: true

    - name: Ensure data dir
      file:
        path: "{{ data_dir }}"
        state: directory
        owner: quicknotes
        group: quicknotes
        mode: "0750"

    - name: Install QuickNotes binary
      copy:
        src: files/quicknotes
        dest: /usr/local/bin/quicknotes
        owner: root
        group: root
        mode: "0755"
      notify: restart quicknotes

    - name: Install systemd unit
      template:
        src: templates/quicknotes.service.j2
        dest: /etc/systemd/system/quicknotes.service
        owner: root
        group: root
        mode: "0644"
      notify:
        - restart quicknotes

    - name: Enable + start service
      systemd:
        name: quicknotes
        enabled: true
        state: started
        daemon_reload: true

  handlers:
    - name: restart quicknotes
      systemd:
        name: quicknotes
        state: restarted
```

### 1.4: The systemd unit template

```jinja
# ansible/templates/quicknotes.service.j2
[Unit]
Description=QuickNotes API
After=network-online.target

[Service]
ExecStart=/usr/local/bin/quicknotes
Restart=on-failure
RestartSec=2
User=quicknotes
Group=quicknotes
WorkingDirectory={{ data_dir }}
Environment=ADDR={{ listen_addr }}
Environment=DATA_PATH={{ data_dir }}/notes.json
Environment=SEED_PATH=/usr/local/share/quicknotes-seed.json

[Install]
WantedBy=multi-user.target
```

### 1.5: Run it

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml --check    # dry-run
ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml            # real run
```

Verify from your host:

```bash
curl -s http://localhost:18080/health   # Vagrant port-forward from Lab 5
```

### 1.6: Document

In `submissions/lab7.md`:
- Output of `ansible --version`
- Full `ansible-playbook` run output (or PLAY RECAP summary)
- `curl` output from host hitting the VM
- The full `playbook.yaml` + `quicknotes.service.j2` (or links to your fork)

---

## Task 2 — Prove Idempotency + Selective Re-run (4 pts)

### 2.1: Re-run = zero changes

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml
# expected PLAY RECAP: changed=0, failed=0
```

Capture the recap — `changed=0` is the proof.

### 2.2: Change one variable, observe selective change

Edit `ansible/playbook.yaml`:

```yaml
vars:
  listen_addr: ":9090"   # was :8080
```

Update the Vagrant port-forward (Lab 5) accordingly or use a different host port. Re-run:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml
```

Expected: **only** the systemd unit template task reports `changed`, and the `restart quicknotes` handler fires. Other tasks: `ok`, no change.

### 2.3: Use --check + --diff for previewing

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml --check --diff
```

Capture an example diff for any task that would change.

### 2.4: Document

In `submissions/lab7.md`:
- The `changed=0` second-run recap
- The `changed=1` variable-tweak recap with the handler firing
- `--check --diff` example
- One paragraph: *what specifically prevents Ansible from making a change when nothing's drifted? (modules vs `shell:` escape hatch)*

---

## Bonus Task — `ansible-pull` GitOps Loop (2 pts)

### B.1: Push your `ansible/` to your fork

```bash
git switch feature/lab7
git add ansible/
git commit -S -s -m "ansible(lab7): deploy QuickNotes"
git push origin feature/lab7
```

### B.2: Bootstrap `ansible-pull` inside the VM

```bash
vagrant ssh

sudo apt-get install -y ansible python3-pip git
sudo mkdir -p /etc/ansible-pull
sudo tee /etc/ansible-pull/inventory.ini > /dev/null << 'EOF'
[quicknotes_vm]
127.0.0.1 ansible_connection=local
EOF
```

### B.3: Systemd timer

```ini
# /etc/systemd/system/ansible-pull.service
[Unit]
Description=Pull QuickNotes config from Git

[Service]
Type=oneshot
ExecStart=/usr/bin/ansible-pull \
  -U https://github.com/YOU/DevOps-Intro.git \
  -C feature/lab7 \
  -i /etc/ansible-pull/inventory.ini \
  ansible/playbook.yaml
```

```ini
# /etc/systemd/system/ansible-pull.timer
[Unit]
Description=Run ansible-pull every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ansible-pull.timer
sudo systemctl list-timers | grep ansible-pull
```

### B.4: Demonstrate convergence

From your host: edit `playbook.yaml`, push to your fork's `feature/lab7`. Wait ≤ 5 minutes. SSH into the VM, confirm the change reconciled automatically.

### B.5: Document

In `submissions/lab7.md`:
- `systemctl list-timers` showing `ansible-pull.timer`
- A timeline: git commit timestamp → next timer fire → reconciled state inside VM
- One paragraph: *how does this compare to ArgoCD's reconcile loop on Kubernetes?*

---

## How to Submit

1. `ansible/` directory in your fork (playbook, inventory template, files, templates)
2. `submissions/lab7.md` covers all attempted tasks
3. PR from `feature/lab7` → course repo's `main`
4. Submit the PR URL via Moodle

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ Playbook deploys QuickNotes to the Vagrant VM
- ✅ Service runs, host can reach `:18080/health`
- ✅ Submission shows full PLAY RECAP

### Task 2 (4 pts)
- ✅ Second-run shows `changed=0`
- ✅ Variable tweak triggers exactly the affected handler
- ✅ `--check --diff` example captured

### Bonus Task (2 pts)
- ✅ `ansible-pull` systemd timer installed
- ✅ Push-to-Git → reconciled inside VM observed
- ✅ Compared to ArgoCD's reconcile loop

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Idempotent Ansible deploy | **6** | Playbook works, service runs, recap shows tasks completed |
| **Task 2** — Idempotency + handler logic | **4** | `changed=0` on re-run, selective change on tweak, --diff example |
| **Bonus** — `ansible-pull` GitOps loop | **2** | Timer installed, convergence demoed, compared to ArgoCD |
| **Total** | **10 + 2 bonus** | |

---

## Common Pitfalls

- 🪤 **SSH connection refused** — Vagrant insecure key path varies by OS; use `vagrant ssh-config` to find it
- 🪤 **`shell:` escape hatch instead of modules** — kills idempotency. Use `apt`, `file`, `copy`, `template`, `systemd`
- 🪤 **`become: true` missing** — most tasks need sudo. Either set on the play or per-task
- 🪤 **`changed=1` on every run for the *same* file** — `copy:` mode/owner mismatch; check `--diff`
- 🪤 **Handler not firing on second tweak** — `notify:` is by name; check spelling
- 🪤 **Vagrant box's Python missing** — `apt install python3` via `raw:` first, then real tasks

---

## Guidelines

- Use the QuickNotes binary you built in `app/` — don't rebuild inside the VM (Lab 5 already proved you *can*)
- Treat `--check --diff` as your dry-run discipline before every change
- The ansible-pull bonus is the "GitOps preview" — the same conceptual pattern as ArgoCD, just at the VM layer
- Document everything you tried *and* the wrong turns — the postmortem is part of the deliverable

---

## Resources

- 📖 [Ansible docs — User Guide](https://docs.ansible.com/ansible/latest/user_guide/index.html)
- 📕 *Ansible: Up and Running* (3rd ed) — Lorin Hochstein & René Moser
- 📗 *Ansible for DevOps* — Jeff Geerling
- 📖 [ansible-pull docs](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html)
- 🛠️ `ansible-lint`, `molecule` (advanced test framework — out of scope for this lab)
