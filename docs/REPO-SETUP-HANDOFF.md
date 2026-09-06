# Repository Setup & ADR Handoff — Running Buddy App

**Course:** 2110521 Software Architectures — Term Project **Organization:** `Chula-Com-Sci-2026` (already created ✅) **Repository:** *not created yet — this doc is how we create it* **Status:** Ready to execute · assign one person to run §2–§6, then open a PR for review

---

## 0\. What this covers

A single, followable path to go from "empty organization" to "repo with our folder structure, a README, and a working ADR log." The focus is **ADRs**: we record architecture decisions as markdown in-repo and manage them with **adr-tools (npryce)**.

One person should run steps §2–§6 locally and push. Everyone else then just clones.

---

## 1\. Prerequisites

Before you start, make sure you have:

- Membership in the `Chula-Com-Sci-2026` org with permission to create repos  
- `git` installed and configured (`git config --global user.name` / `user.email`)  
- An SSH key added to your GitHub account (or use HTTPS — both shown below)  
- **adr-tools (npryce)** installed — see §5. This is bash-based, so Windows users need WSL or Git Bash.  
- Agreement on the repo name. This doc assumes **`running-buddy`**.

---

## 2\. Create the repository under the organization

Do this on GitHub, but **do not let GitHub initialize it** — we'll push our own structure so there's no "unrelated histories" mess.

1. Go to the org: `https://github.com/Chula-Com-Sci-2026`  
2. **Repositories → New repository** (or the top-right **\+ → New repository**)  
3. **Owner:** select `Chula-Com-Sci-2026` — *not* your personal account  
4. **Repository name:** `running-buddy`  
5. **Visibility:** **Private**  
6. **Initialize this repository with:** leave **all unchecked** (no README, no .gitignore, no license)  
7. **Create repository**

Copy the repo URL shown on the next screen. It will look like one of:

- SSH: `git@github.com:Chula-Com-Sci-2026/running-buddy.git`  
- HTTPS: `https://github.com/Chula-Com-Sci-2026/running-buddy.git`

---

## 3\. Build the structure locally

From an empty working folder:

mkdir running-buddy && cd running-buddy

git init

git branch \-M main

\# top-level docs

mkdir \-p docs

\# (the ADR log itself is created by adr-tools in §6 — don't hand-create docs/adr)

\# move the existing requirements doc in (adjust the source path)

\# cp /path/to/requirements-running-app.md docs/requirements.md

We deliberately do **not** pre-create `src/` or `services/` here. The service decomposition is an architecture decision, not a setup decision — it belongs in **ADR-0002** (see §8). Create those folders once that ADR is accepted.

---

## 4\. Target folder structure

After §6 and the first commits, the repo should look like this:

running-buddy/

├── README.md               ← project intro \+ how we use ADRs (drop-in below, §9)

├── .gitignore

├── .adr-dir                ← created by \`adr init\`; records where ADRs live. COMMIT THIS.

├── docs/

│   ├── requirements.md     ← the requirements doc

│   └── adr/                ← the ADR log (adr-tools manages this)

│       ├── 0001-record-architecture-decisions.md

│       └── templates/      ← optional custom template (MADR/Nygard); see §7

└── services/               ← added later, once ADR-0002 decides the decomposition

Rule of thumb: **decisions live in `docs/adr/`, the code they describe lives beside it in the same repo.** They version together and flow through the same pull-request review.

---

## 5\. Install adr-tools (npryce)

Pick your platform. The tool is a set of bash scripts, so Windows needs a bash environment.

**macOS (Homebrew):**

brew install adr-tools

**Linux / manual install (also works in WSL and Git Bash):**

git clone https://github.com/npryce/adr-tools.git \~/adr-tools

\# add its src dir to your PATH (put this line in \~/.bashrc or \~/.zshrc to make it permanent)

export PATH="$PATH:$HOME/adr-tools/src"

**Windows:** use **WSL** (recommended) or **Git Bash**, then follow the Linux steps above.

Verify:

adr help

> Note: the original `npryce/adr-tools` repo is stable but not very actively maintained. If a teammate hits an install snag, there are maintained community forks with the same `adr` command. Installing the tool is a convenience — an ADR is ultimately just a numbered markdown file, so anyone who can't install it can still create one by hand following the same numbering.

---

## 6\. Initialize the ADR log

From the repo root:

adr init docs/adr

This:

- creates `docs/adr/`  
- writes **`0001-record-architecture-decisions.md`** (records that we use ADRs, links to Nygard's original article)  
- creates a hidden **`.adr-dir`** file at the repo root pointing at `docs/adr` — **commit this** so `adr new` works for everyone without re-specifying the path.

Then commit and push the whole initial setup:

git add .

git commit \-m "chore: initialize repo with README, requirements, and ADR log"

git remote add origin git@github.com:Chula-Com-Sci-2026/running-buddy.git

git push \-u origin main

(Swap in the HTTPS URL if you're not using SSH.)

---

## 7\. How we manage ADRs

**Everyday commands:**

| Command | What it does |
| :---- | :---- |
| `adr new "Use PostgreSQL with PostGIS"` | creates the next numbered ADR and opens your editor |
| `adr new -s 4 "Switch to event-driven notifications"` | creates a new ADR that **supersedes** ADR-0004 and auto-flips 0004's status |
| `adr link 5 "Amends" 3 "Amended by"` | links two ADRs |
| `adr generate toc` | generates a table of contents |
| `adr generate graph` | generates a Graphviz graph of ADR links |
| `adr list` | lists all ADRs |

**Conventions we follow:**

- **Numbering is append-only.** Never renumber or delete an ADR. Numbers are permanent IDs.  
- **Status lifecycle:** `Proposed → Accepted`, and later `Superseded by NNNN` if reversed. We keep superseded ADRs — the record of *why we changed our minds* is the valuable part.  
- **One ADR per pull request.** An architecture decision gets proposed as a PR, discussed by the team, and merged only when accepted. This is the decision's review trail.  
- **Trace to a driver.** Each real ADR should reference the architectural driver / NFR it satisfies (see the "Architectural drivers" table in `docs/requirements.md`). The grader wants to see the reasoning chain from requirement → decision.  
- **Custom template (optional):** if you want MADR instead of the default Nygard format, drop a `docs/adr/templates/template.md` and adr-tools will use it for new ADRs.

---

## 8\. Seed ADRs — write these as the decisions actually happen

`adr init` gives you 0001 for free. The following are the decisions our requirements already point to, mapped to the architectural drivers. **Don't write them all up front** — create each one with `adr new` at the moment the team actually decides it, so the ADR captures a real decision, not a guess.

| ADR | Decision to record | Driver (from requirements §4) |
| :---- | :---- | :---- |
| 0001 | Record architecture decisions | — (created by `adr init`) |
| 0002 | Service decomposition & data ownership (which services, who owns what data) | Driver 1 — NFR-7.1 / 7.3 |
| 0003 | Inter-service communication: sync vs async, \+ anti-corruption layer around the weather provider | Driver 2 — NFR-3.2 / 3.3 |
| 0004 | Database & geospatial indexing choice (e.g. PostGIS for nearby-runner/nearby-run queries) | Drivers 1 & 4 — NFR-7.3, 1.1 / 1.2 |
| 0005 | AuthN/AuthZ approach: verify every request (zero-trust), authorization placement, location coarsening | Driver 3 — NFR-4.3 / 5.1 |

---

## 9\. Drop-in `README.md`

Create `README.md` at the repo root with this content. It states plainly that we use adr-tools (npryce) to manage our ADRs, so any teammate opening the repo knows how to proceed.

\# Running Buddy App

A web app that helps runners find each other, schedule spontaneous group runs at a pinned meeting

point, and plan around the weather. Term project for \*\*2110521 Software Architectures\*\*.

\#\# Documentation

\- \*\*Requirements:\*\* \[\`docs/requirements.md\`\](docs/requirements.md)

\- \*\*Architecture Decision Records:\*\* \[\`docs/adr/\`\](docs/adr/)

\#\# Architecture Decision Records (ADRs)

We record every significant architecture decision as an ADR — a numbered markdown file in

\[\`docs/adr/\`\](docs/adr/) — and we manage them with \*\*\[adr-tools (npryce)\](https://github.com/npryce/adr-tools)\*\*.

ADRs live next to the code they describe so that a decision and its implementation version together

and flow through the same pull-request review.

\#\#\# Working with ADRs

Install the tool (see the \[adr-tools README\](https://github.com/npryce/adr-tools) — \`brew install

adr-tools\` on macOS; clone \+ PATH on Linux/WSL/Git Bash), then from the repo root:

\`\`\`bash

\# create the next numbered ADR

adr new "Short title of the decision"

\# create an ADR that supersedes an earlier one (e.g. ADR 4\)

adr new \-s 4 "Replacement decision"

\# regenerate the table of contents

adr generate toc

\`\`\`

\*\*Conventions\*\*

\- Numbering is append-only; never renumber or delete an ADR.

\- Status flows \`Proposed → Accepted\`, then \`Superseded by NNNN\` if reversed. Superseded ADRs stay.

\- One ADR per pull request, reviewed by the team before merge.

\- Each ADR references the architectural driver / NFR it addresses.

\#\# Getting started

\_Local run instructions to be added once the service structure is decided (see ADR-0002).\_

---

## 10\. `.gitignore`

Create a `.gitignore` appropriate to whatever stack you land on. A safe starting set:

\# dependencies

node\_modules/

\_\_pycache\_\_/

.venv/

\# env & secrets

.env

\*.local

\# build output

dist/

build/

\# OS / editor

.DS\_Store

.idea/

.vscode/

---

## 11\. Setup checklist

- [ ] Repo `running-buddy` created under `Chula-Com-Sci-2026`, **Private**, not auto-initialized  
- [ ] Local repo initialized, `main` branch  
- [ ] `docs/requirements.md` added  
- [ ] `README.md` added (from §9)  
- [ ] `.gitignore` added (from §10)  
- [ ] `adr init docs/adr` run; `0001` and `.adr-dir` present  
- [ ] `.adr-dir` committed  
- [ ] Initial commit pushed to `origin/main`  
- [ ] Teammates invited to the org / repo  
- [ ] Instructor/TA added as read-only member or outside collaborator (if they want to follow along)  
- [ ] (Optional) `docs/adr/templates/template.md` added if we prefer MADR over the default format

---

## 12\. Notes for the team

- **Branch protection** (required reviewers, protected `main`) needs the paid Team plan. On free, we enforce "no merge without one approval" by convention — agreed, not enforced by GitHub.  
- **Member visibility** defaults to private; you can lock it in under **Org settings → Member privacy**. All repos stay private regardless.  
- Keep this handoff in the repo (e.g. `docs/REPO-SETUP-HANDOFF.md`) so the setup is reproducible.

