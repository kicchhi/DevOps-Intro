# Lab 2 — Version Control & Advanced Git

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Git%20%26%20Version%20Control-blue)
![points](https://img.shields.io/badge/points-10-orange)

> **Goal:** Deepen Git fundamentals: object model, reset/reflog, history visualization, tagging, and modern commands (`git switch`/`git restore`).  
> **Deliverable:** A PR from `feature/lab2` to the course repo with `labs/submission2.md` including outputs and brief explanations for each task. Submit the PR link via Moodle.

---

## Overview

In this lab you will practice:
- Inspecting Git's object model with `git cat-file`.
- Recovering work with `git reset` and `git reflog` safely.
- Visualizing history and branches with `git log --graph`.
- Tagging commits for releases.
- Using modern commands: `git switch` and `git restore` vs `git checkout`.

---

## Tasks

### Task 1 — Git Object Model Exploration (2 pts)

**Objective:** Understand how Git stores data as blobs, trees, and commits.

#### 1.1: Create Sample Commits

1. **Make Sample Commits:**

   ```sh
   # Create a few test commits for analysis
   echo "Test content" > test.txt
   git add test.txt
   git commit -m "Add test file"
   ```

#### 1.2: Inspect Git Objects

<details>
<summary>🔍 How to find object hashes</summary>

```sh
# Get commit hash
git log --oneline -1

# Get tree hash from commit
git cat-file -p HEAD

# Get blob hash from tree
git cat-file -p <tree_hash>
```

</details>

1. **Examine Git Objects:**

   ```sh
   # Replace with real object IDs from your repo
   git cat-file -p <blob_hash>
   git cat-file -p <tree_hash>
   git cat-file -p <commit_hash>
   ```

In `labs/submission2.md`, document:
- All command outputs for object inspection.
- A 1–2 sentence explanation of what each object type represents.
- Analysis of how Git stores repository data.
- Example of blob, tree, and commit object content.

---

### Task 2 — Reset and Reflog Recovery (2 pts)

**Objective:** Practice using `git reset` variants and `git reflog` to navigate history.

#### 2.1: Create Practice Branch

1. **Set Up Practice Environment:**

   ```sh
   git switch -c git-reset-practice
   echo "First commit" > file.txt && git add file.txt && git commit -m "First commit"
   echo "Second commit" >> file.txt && git add file.txt && git commit -m "Second commit"
   echo "Third commit"  >> file.txt && git add file.txt && git commit -m "Third commit"
   ```

#### 2.2: Explore Reset Modes

1. **Test Different Reset Options:**

   ```sh
   git reset --soft HEAD~1   # move HEAD; keep index & working tree
   git reset --hard HEAD~1   # move HEAD; discard index & working tree
   git reflog                # view HEAD movement
   git reset --hard <reflog_hash>  # recover a previous state
   ```

In `labs/submission2.md`, document:
- The exact commands you ran and why.
- Snippets of `git log --oneline` and `git reflog`.
- What changed in the working tree, index, and history for each reset.
- Analysis of recovery process using reflog.

---

### Task 3 — Visualize Commit History (2 pts)

**Objective:** Use Git's log graph to see branching and merges.

1. **Create a short-lived branch, commit, then view the graph:**

   ```sh
   git switch -c side-branch
   echo "Branch commit" >> history.txt
   git add history.txt && git commit -m "Side branch commit"
   git switch -
   git log --oneline --graph --all
   ```

In `labs/submission2.md`, document:
- A snippet/screenshot of the graph.
- Commit messages list.
- A 1–2 sentence reflection on how the graph aids understanding.

---

### Task 4 — Tagging Commits (1 pt)

**Objective:** Create and push lightweight tags to mark releases.

1. **Tag the latest commit and push:**

   ```sh
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. Optionally make one more commit and tag `v1.1.0`.

In `labs/submission2.md`, document:
- Tag names and commands used.
- Associated commit hashes.
- A short note on why tags matter (versioning, CI/CD triggers, release notes).

---

### Task 5 — git switch vs git checkout vs git restore (2 pts)

**Objective:** Learn modern Git commands and when to use each.

<details>
<summary>🔄 Option A: git switch (Modern - Recommended)</summary>

```sh
git switch -c cmd-compare   # create and switch
git switch -                # toggle back to previous branch
```

**Purpose:** Branch switching only (clear and focused)

</details>

<details>
<summary>🔄 Option B: git checkout (Legacy - Overloaded)</summary>

```sh
git checkout -b cmd-compare-2   # also creates + switches branches
# Note: `git checkout -- <file>` used to restore files (confusing!).
```

**Problem:** Does too many things - branches AND files

</details>

<details>
<summary>📂 git restore (Modern - File Operations)</summary>

```sh
echo "scratch" >> demo.txt
git restore demo.txt                 # discard working tree changes
git restore --staged demo.txt        # unstage (keep working tree)
git restore --source=HEAD~1 demo.txt # restore from another commit
```

**Purpose:** File restoration only (clear and focused)

</details>

In `labs/submission2.md`, document:
- Commands you ran and their outputs.
- `git status`/`git branch` outputs showing state changes.
- 2–3 sentences on when to use each command.

---

### Task 6 — GitHub Community Engagement (1 pt)

**Objective:** Explore GitHub's social features that support collaboration and discovery.

**Actions Required:**
1. **Star** the course repository
2. **Star** the [simple-container-com/api](https://github.com/simple-container-com/api) project — a promising open-source tool for container management
3. **Follow** your professor and TAs on GitHub:
   - Professor: [@Cre-eD](https://github.com/Cre-eD)
   - TA: [@marat-biriushev](https://github.com/marat-biriushev)
   - TA: [@pierrepicaud](https://github.com/pierrepicaud)
4. **Follow** at least 3 classmates from the course

**Document in labs/submission2.md:**

Add a "GitHub Community" section (after Challenges & Solutions) with 1-2 sentences explaining:
- Why starring repositories matters in open source
- How following developers helps in team projects and professional growth

<details>
<summary>💡 GitHub Social Features</summary>

**Why Stars Matter:**

**Discovery & Bookmarking:**
- Stars help you bookmark interesting projects for later reference
- Star count indicates project popularity and community trust
- Starred repos appear in your GitHub profile, showing your interests

**Open Source Signal:**
- Stars encourage maintainers (shows appreciation)
- High star count attracts more contributors
- Helps projects gain visibility in GitHub search and recommendations

**Professional Context:**
- Shows you follow best practices and quality projects
- Indicates awareness of industry tools and trends

**Why Following Matters:**

**Networking:**
- See what other developers are working on
- Discover new projects through their activity
- Build professional connections beyond the classroom

**Learning:**
- Learn from others' code and commits
- See how experienced developers solve problems
- Get inspiration for your own projects

**Collaboration:**
- Stay updated on classmates' work
- Easier to find team members for future projects
- Build a supportive learning community

**Career Growth:**
- Follow thought leaders in your technology stack
- See trending projects in real-time
- Build visibility in the developer community

**GitHub Best Practices:**
- Star repos you find useful (not spam)
- Follow developers whose work interests you
- Engage meaningfully with the community
- Your GitHub activity shows employers your interests and involvement

</details>

---

## How to Submit

1. Create a branch for this lab and push it:

   ```bash
   git switch -c feature/lab2
   # add labs/submission2.md with your findings
   git add labs/submission2.md
   git commit -m "docs: add lab2 submission"
   git push -u origin feature/lab2
   ```

2. Open a PR from your fork's `feature/lab2` branch → **course repository's main branch**.

3. In the PR description, include:

   ```text
   - [x] Task 1 done
   - [x] Task 2 done
   - [x] Task 3 done
   - [x] Task 4 done
   - [x] Task 5 done
   - [x] Task 6 done
   ```

4. **Copy the PR URL** and submit it via **Moodle before the deadline**.

---

## Acceptance Criteria

- ✅ Branch `feature/lab2` exists with commits for each task.
- ✅ File `labs/submission2.md` contains required outputs/explanations for Tasks 1–6.
- ✅ A tag (e.g., `v1.0.0`) is created locally and pushed to origin.
- ✅ PR from `feature/lab2` → **course repo main branch** is open.
- ✅ PR link submitted via Moodle before the deadline.

---

## Rubric (10 pts)

| Criterion                                   | Points |
| ------------------------------------------- | -----: |
| Task 1 — Object model exploration           |   **2**|
| Task 2 — Reset and reflog recovery          |   **2**|
| Task 3 — History visualization              |   **2**|
| Task 4 — Tagging commits                    |   **1**|
| Task 5 — switch vs checkout vs restore      |   **2**|
| Task 6 — GitHub community engagement        |   **1**|
| **Total**                                   |  **10**|

---

## Guidelines

- Use clear Markdown headers to organize sections in `submission2.md`.
- Include both command outputs and written analysis for each task.
- Use clear commit messages and keep screenshots/snippets concise.
- Organize files under `labs/` and name them predictably.

<details>
<summary>📚 References</summary>

- [Git Documentation](https://git-scm.com/doc)
- [Pro Git Book](https://git-scm.com/book/en/v2)

</details>

<details>
<summary>💡 Git Command Tips</summary>

1. Prefer `git switch`/`git restore` over legacy `git checkout` for clarity.
2. Always check `git status` after reset operations to understand the state.
3. Use `git reflog` for recovery when commits seem lost.

</details>