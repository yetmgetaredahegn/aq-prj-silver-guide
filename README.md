# AQ Project-Silver Guide
### A field guide to making money on AfterQuery Project Silver — written from real approvals, real rejections, and real money lost.

> **Who this is for.** Someone who has access to Project Silver and wants to actually get paid, not just read theory. It assumes you will be using an AI coding agent (Claude Code / Codex) to help you build tasks. Every claim here is grounded in one of three things: (1) the official Silver FAQ (the live source of truth), (2) the official training, or (3) hard-won experience on this account — **one repo accepted, two repos rejected, one task approved + paid ($75), and ~six tasks killed by the gates.** Where experience contradicts theory, experience wins and I say so.

> **How this guide is different from the others in this folder.** `PROJECT_SILVER_GUIDE.md`, `AMOLE_TASK_PLAYBOOK.md`, and the `training/` snapshots are accurate but they teach the *happy path*. They do not show you a bad instruction next to a good one, they do not give you a decision tree for "my task got rejected, now what," they do not show you how to actually encode a `test_patch` into JSON, and they have exactly one worked example. This guide is built around the **failure modes** — because on Silver, money is lost at the gates, not at the keyboard. Read this one for *why*; keep the others open for *reference*.

---

## Table of contents

1. [The one-paragraph mental model](#1-the-one-paragraph-mental-model)
2. [How you actually get paid (the economics)](#2-how-you-actually-get-paid-the-economics)
3. [The two-phase game: repo first, then tasks](#3-the-two-phase-game-repo-first-then-tasks)
4. [PHASE 1 — Getting a repo accepted](#4-phase-1--getting-a-repo-accepted)
5. [The nine gates a task must pass](#5-the-nine-gates-a-task-must-pass)
6. [Anatomy of a task package — every file, what it does, what to edit](#6-anatomy-of-a-task-package--every-file-what-it-does-what-to-edit)
7. [The tools: Harbor, the verifier, NOP & Oracle](#7-the-tools-harbor-the-verifier-nop--oracle)
8. [The heart of the craft: what makes a task HARD-but-FAIR](#8-the-heart-of-the-craft-what-makes-a-task-hard-but-fair)
9. [Writing `instruction.md` (the AI-text gate is real)](#9-writing-instructionmd-the-ai-text-gate-is-real)
10. [Writing the tests and encoding `test_patch`](#10-writing-the-tests-and-encoding-test_patch)
11. [The local validation loop (do this before every submit)](#11-the-local-validation-loop-do-this-before-every-submit)
12. [Failure recovery — a decision tree for every rejection](#12-failure-recovery--a-decision-tree-for-every-rejection)
13. [War stories: what got paid, what got killed, and why](#13-war-stories-what-got-paid-what-got-killed-and-why)
14. [Diversifying to 5+ tasks without tripping similarity](#14-diversifying-to-5-tasks-without-tripping-similarity)
15. [The pre-submit checklist](#15-the-pre-submit-checklist)
16. [Quick reference card](#16-quick-reference-card)

---

## 1. The one-paragraph mental model

Project Silver pays you to manufacture **SWE-bench-style coding problems** out of your own private repository. One "task" = a frozen starting state of your repo (`base_commit`), a set of **new tests** you wrote that *fail* on that starting state, and a **reference fix** (`solve.sh`) that makes them pass. AfterQuery then unleashes an AI agent (Sonnet 4.6) on your starting state + your instruction, *without* your fix, and checks whether the agent can solve it. **You get paid $75 when your task lands in a narrow sweet spot: hard enough that the agent usually fails, but fair enough that the reference fix genuinely passes and the instruction genuinely describes the wanted behavior.** Too easy → rejected. Unsolvable → rejected. Tests that don't match the instruction → rejected. Everything in this guide is about hitting that sweet spot repeatably.

---

## 2. How you actually get paid (the economics)

Source: Silver FAQ → 💰 Payments. Internalize these because they change your strategy.

| Thing | Amount | The catch that bites people |
|---|---|---|
| One approved task | **$75** | Only counts if attached to a repo that reaches 5 approvals (see below) |
| A repo hitting **5+ approved tasks** | **+$300 bonus** | This is the real prize. |
| A repo with **0–4 approved tasks** | **$0 — including bonuses** | **This is the trap.** Four approved tasks on a repo that never reaches five pays you *nothing* if the payout is gated on the repo. Read the FAQ line: *"Payouts including bonuses are contingent on repos that have tasks attached to them… You need 5 to unlock the repo's payouts."* |
| Cap per repo | 20 approved tasks | After 20 that repo is closed to you. |
| Volume bonuses | Tier 1 at 25 cumulative approved, Tier 2 at 50, etc., additive, repeating | Cumulative across all repos. |
| Referral | $100 each, after the referral completes 10 approved tasks | — |

**Strategic consequences (this account learned these the expensive way):**

- **A repo is only worth starting if you believe you can get 5 tasks out of it.** One or two approvals on a repo you then abandon is wasted effort. Before you invest in a repo, ask: *does its commit history contain at least 5 genuinely hard, carvable changes?* (See §8 and §13 — most "improvement" commits are NOT carvable.)
- **Submission timestamp — not approval timestamp — determines bonus eligibility.** Get tasks *submitted* early, especially before Wednesday cutoffs (payments are processed manually Wednesdays EOD PST).
- **Opus/difficulty runs are metered and audited.** The FAQ is explicit: *"Limit your Opus runs to max 5 per task… Excessive reruns will lead to task creation skill being disabled… project admins will remove you."* Each full DIFFICULTY_CHECK run is assumed to cost $5–10. **Never burn runs on a task you haven't locally proven.** Local NOP/Oracle is free; the platform's difficulty probe is not.

---

## 3. The two-phase game: repo first, then tasks

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  PHASE 1: REPO REVIEW (one-shot, holistic, FINAL if rejected)    │
   │  Upload a private repo → daily ~12AM EST review → accept/reject  │
   │  Accept ⇒ a frozen base image is built and tasks unlock.         │
   │  Reject ⇒ permanent. Cannot fix-and-resubmit. Cannot reupload.   │
   └─────────────────────────────────────────────────────────────────┘
                                   │  (only if accepted)
                                   ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  PHASE 2: TASK CREATION (up to 20 per repo, iterate freely)      │
   │  For each task: build package → validate locally with Harbor →   │
   │  submit → 9 gates → approved ($75) or sent back for revision.    │
   │  Revisions ARE allowed. Reruns are metered. 5 approvals = $300.  │
   └─────────────────────────────────────────────────────────────────┘
```

The asymmetry is the whole game: **repo rejection is final and unappealable; task rejection is recoverable.** So you spend disproportionate care on the repo, and you iterate cheaply on tasks — but always *locally first*.

---

## 4. PHASE 1 — Getting a repo accepted

> Official refs: `training/Submit a repository.md`, FAQ → 📦 Repos. Read both. This section adds the **why behind rejections**, which the training omits.

### 4.1 The non-negotiable mechanical requirements

| Requirement | Detail | The failure it prevents |
|---|---|---|
| **You own the IP** | Your own private repo. No friends' repos, no repo you contributed to, no briefly-public repo. They actively probe GitHub/GitLab/Bitbucket/Gitea and reject matches, forks, mirrors, renames. | Instant rejection. |
| **Private hosting** | Private GitHub or fully off-GitHub. | — |
| **Real `.git/` with sustained history** | Not a fresh `git init`, not one squashed commit, not a snapshot. | The "no reachable commits" / "thin history" rejection. |
| **`tests/` directory** | Must be named `tests/` (plural, not `test/`), contain real tests, be recoverable by the harness. | "Insufficient test coverage" umbrella rejection. |
| **`.dockerignore` must NOT contain `.git`** | This is the **#1 cause of downstream failures** per the FAQ. | Every task on the repo fails later with "fatal: not a git repository." |
| **Substantive codebase** | Real project. Scaffolds, starter templates, toy projects, **and AI/LLM-generated repos all fail the quality bar.** | The holistic-quality rejection (see §4.3). |
| **Default branch `main`** | Rename `master`→`main`: `git branch -m master main`. | History-not-found rejection. |
| **Correct ZIP layout** | `repo-name.zip → repo-name/ → {.git/, src/, tests/, …}`. No double-wrapping, no flat zip. | Skeleton generator breaks. |

### 4.2 The ZIP, precisely

```
repo-name.zip
└── repo-name/
    ├── .git/            ← MUST be present and reachable (not in .dockerignore!)
    ├── <source dirs>/
    ├── tests/           ← plural
    └── ...
```

Constraints from the FAQ / training: keep the eventual **image < 500 MB** (use `python:3.x-slim`, not full), zip should be reasonably small. After zipping, **extract your own zip into a clean folder and confirm `.git/HEAD` exists** — some OS zip tools strip dotfiles. This 30-second check prevents the most common repo rejection.

**Real example that passed** — the `amole` repo (this account's only accepted repo):
- 258 source files, ~15k LOC Python backend + ~8k LOC TS frontend, 8 Django domain apps.
- `.dockerignore` led with a literal comment: `# IMPORTANT: do not exclude .git/ — task harness needs full history`.
- 253 backend tests, 8 cross-app integration tests, factory_boy fixtures, CI against real Postgres+Redis.
- 3 formal Architecture Decision Records, deployment runbooks, monetization docs.
- Cohesive product vision end-to-end (no `MyApp`/`UserModel` generic names).

That is what "substantive and real" looks like concretely.

### 4.3 The rejection nobody warns you about: the holistic AI-quality bar

This is the most important and least-documented thing about Phase 1, and it cost this account **two repos**.

The repo review is **holistic and the error codes are hidden** (FAQ: *"over 100 error codes… the error message to you is generic"*). The generic message you get — almost always **"Insufficient test coverage"** — is an *umbrella*. It does **not** literally mean your coverage percentage was low.

**Proof from this account:** the `alx-project-nexus` repo was rejected for "Insufficient test coverage" while measuring **90% branch coverage** locally (234 tests, 2117 statements, 163 missed). 90% is not a coverage failure. The real cause was that the repo's recent history was an **AI-generated enhancement burst** — dozens of large commits authored in a short window by an AI agent. The earlier `WeCare` repo died the same way. 

**The lesson, stated plainly:** *AfterQuery's reviewer can smell AI-burst / AI-generated history and code, and it rejects on that regardless of how high your coverage is or how many tests you have.* The only repo that passed here was the one with **organic, human, sustained development history**. 

> ⚠️ **Do not "enhance" a repo with an AI agent to make it look more impressive before submitting.** It backfires. A modest, real, hand-built project beats an AI-buffed one every time at this gate. If your repo's history is one giant AI burst, it will be rejected and you cannot resubmit it — the rejection is final and even a re-upload is blocked as "too similar to an existing one."

### 4.4 What a "5-task-worthy" repo looks like *before* you commit to it

Because 0–4 approvals pays $0, vet the repo for *carvable difficulty* before Phase 1:

- Does the history contain **changes that modified existing behavior** (bug fixes, refactors, hardening)? Those carve into tasks. Pure greenfield feature-adds do **not** carve well (there's no "naive before" state to start the agent from). See §8.
- Are there **at least 5 distinct domains/mechanisms**? You can't ship 5 near-identical tasks — similarity gate caps it at "no more than 5 similar" and in practice much less. You want variety: a constraint here, a permission there, a serializer redaction, a state-machine guard, etc.
- Is the test harness **deterministic-friendly**? If the only "hard" things in the repo are concurrency races, beware (see §13, the concurrency dead-end).

---

## 5. The nine gates a task must pass

The training lists ~6 stages; the live FAQ lists more; in practice your task is judged by this full pipeline. Know each gate, what trips it, and **which file you edit to fix it**.

| # | Gate | What it checks | Trips on | File you tune |
|---|---|---|---|---|
| 1 | **Similarity** | Cosine ≥ 0.75 **or** Levenshtein ≥ 0.70 vs *any* existing task (not just yours) → rejected | Paraphrased/cloned tasks | `instruction.md` + pick a different mechanism |
| 2 | **AI-text check** | `instruction.md` reads like a human wrote it | "AI-assistant register": bullet hierarchies, "robust/idiomatic/in conclusion" | `instruction.md` |
| 3 | **Quality review** | Substantive, on-spec, holistic | Trivial or off-spec tasks | whole package |
| 4 | **Image build** | The Dockerfile builds | `.git` in `.dockerignore`; missing `git` in base; >500 MB | `environment/Dockerfile` |
| 5 | **NOP check** | Without the fix, the new tests **fail** (and `test_patch` adds ≥ N new tests) | `test_patch` adds 0 tests; tests pass on base anyway (weak) | `tests/config.json`, the test file |
| 6 | **Oracle check** | Your `solve.sh` makes the tests **pass** | `RewardFileNotFoundError`, `ModuleNotFoundError`, patch doesn't apply, F2P doesn't flip | `solution/solve.sh`, Dockerfile plumbing |
| 7 | **Easiness probe (too-easy)** | Sonnet 4.6 should **not** trivially solve it (≤ ~4/10 of probe attempts) | One-line/obvious fixes; stated logic | `instruction.md` (remove hints), task design |
| 8 | **Difficulty assessment** | Pass rate in **1–4 / 10** (Opus 4.8 graded) | 0/10 = unverifiable/unsolvable; 5+/10 = too easy | hints in `instruction.md`, test design |
| 9 | **Fairness** | Tests only assert what the instruction actually says | Hidden requirements (exact strings, ordering, field names not specified) | align `instruction.md` ↔ tests |

The middle gates (4–6) are **plumbing** — mechanical, fully testable locally, no excuse to fail them on the platform. The outer gates (7–9) are **craft** — this is where money is actually won or lost, and where §8 lives.

---

## 6. Anatomy of a task package — every file, what it does, what to edit

This is the exact structure of the **accepted, paid** `task-02-listing-soft-delete`. Clone this skeleton for every new task; only change the task-specific parts.

```
task-NN-name/
├── instruction.md          ← what the agent reads. THE craft file. (gates 2,7,8,9)
├── reference_plan.md        ← your blueprint of the fix (human notes; not shown to agent)
├── task.toml                ← metadata + timeouts + resources
├── environment/
│   ├── Dockerfile           ← FROM the frozen repo image; reset to base_commit; plumbing
│   └── docker-compose.yaml  ← parameterized; copy verbatim, never edit
├── tests/
│   ├── config.json          ← THE CONTRACT: base_commit, test_patch, fail_to_pass, pass_to_pass, patch
│   ├── test.sh              ← verifier orchestrator (copy verbatim)
│   ├── run_script.sh        ← pytest wrapper (copy verbatim)
│   └── parser.py            ← pytest-output → JSON (copy verbatim)
└── solution/
    └── solve.sh             ← the gold fix, embedded inline as a heredoc patch
```

### 6.1 `instruction.md` — the only thing the agent sees

A bug-report narrative in plain human voice. Says **what** behavior is wanted, never **how**. Here is the **verbatim text that got paid** (note the typos and casual voice — that's not sloppiness, that's what beats the AI-text gate):

```
<uploaded_files>
/app/amole
</uploaded_files>

 a seller deleted their own listing through the api. Even though, the delete
call returned 204 a success, the listings still appeared in their marketplace
feed after reload. and the rows was still in the database and moderators could
still see it.

The method ListingService.delete_listing which can be found in
backend/listings/services.py file currently removes the row from the database
completely. Once it is deleted, moderators can't review it later for disputes
or abuse reports.

Based on the inputs we desire this kind of outputs. When u delete your own
listing it should return 204. After that delete, if any public request to
access that listing should return 404. And the listing mustnot appear in the
normal marketplace feed or list queries. But the row must still exists in the
database only reacheable for the administrators through a separate path. Overall
listings in other statuses or normal listings are unaffected.

Make your changes in the application source code only, not in the tests. Do not
edit, add, or remove any test.
```

Why this works (dissected fully in §8 and §9): it describes observable behavior (204 / 404 / hidden-from-feed / row-survives / others-unaffected), it **never names the mechanism** ("soft delete," "manager override," "`is_deleted` flag" appear nowhere), and it reads like a teammate typing in Slack.

### 6.2 `reference_plan.md` — your private blueprint

Plain-language description of the intended fix. Not shown to the agent; it's for you and the reviewer. The paid one described the two-manager pattern, the two new fields, the `soft_delete()` method, the migration, and which existing tests must keep passing. Write this *before* you write the tests — it forces you to know your own answer.

### 6.3 `task.toml` — metadata

```toml
[metadata]
author_name = "Your Name"
author_email = "you@example.com"
difficulty = "medium"
category = "feature"            # feature | bugfix | refactor | perf
tags = ["feature", "swe-bench-pro", "django", "<domain>", "<mechanism>"]
human_solve_time = ""           # populated at submit
requires_internet = false
repo_type = ""
task_type = ""
verifier_runtime_sec = 0

[verifier]
timeout_sec = 3000

[agent]
timeout_sec = 3600

[environment]
build_timeout_sec = 1800.0
cpus = 1
memory_mb = 4096
storage_mb = 10240
```

Common packaging failure (FAQ): missing `[verifier]`/`[agent]`/`[environment]` sections → import failure. Keep all three.

### 6.4 `environment/Dockerfile` — frozen base + reset + plumbing

This is the single most failure-prone *plumbing* file. The accepted pattern, annotated:

```dockerfile
# 1) FROM the EXACT frozen image AQ built for your accepted repo.
#    Copy this reference verbatim from your repo's page. Never guess it.
FROM us-docker.pkg.dev/afterqueryai/silver-repo-images/amole-ynn3v:v1@sha256:bfb44e...

USER root
ENV DEBIAN_FRONTEND=noninteractive

# 2) AQ's cloud build prunes empty .git/refs/{heads,tags}. Git then refuses to
#    recognize the repo ("fatal: not a git repository"). Recreate them.
RUN mkdir -p /workspace/.git/refs/heads /workspace/.git/refs/tags

# 3) Copy the workspace clone into the path your tests expect, if needed.
RUN set -e \
 && if [ -d /workspace/.git ] && [ ! -d /app/amole/.git ]; then \
      mkdir -p /app && cp -a /workspace /app/amole; \
    fi

WORKDIR "/app/amole"

# 4) Freeze to the base_commit. THIS is what defines the task's starting state.
RUN git config --global --add safe.directory /app/amole \
 && git reset --hard <BASE_COMMIT> \
 && git clean -fd \
 && git checkout <BASE_COMMIT>

# 5) Install runtime + test deps (with the --break-system-packages fallback).
RUN pip install --no-cache-dir -r backend/requirements.txt || \
    pip install --no-cache-dir --break-system-packages -r backend/requirements.txt || true
RUN pip install --no-cache-dir pytest==8.3.4 pytest-django==4.9.0 jsonschema==4.21.1

# 6) Env that makes tests run on SQLite, no Redis, eager Celery, etc.
ENV DJANGO_SETTINGS_MODULE=backend.settings USE_SQLITE_FOR_TESTS=1 \
    PYTHONPATH=/app/amole/backend ...

# 7) REWARD PLUMBING (prevents RewardFileNotFoundError). Seed reward files at
#    container start, always-pass healthcheck, keep container alive.
RUN mkdir -p /logs/verifier && chmod 1777 /logs /logs/verifier
HEALTHCHECK CMD true
ENTRYPOINT ["/bin/sh","-c","mkdir -p /logs/verifier; printf '0\\n' > /logs/verifier/reward.txt; exec \"$@\"","--"]
CMD ["sh","-c","sleep infinity"]
```

Per task you change **one thing**: `<BASE_COMMIT>` (in two places). Everything else is copy-verbatim. The FAQ's RuntimeError/RewardFileNotFound clusters all map to getting steps 2, 7, or the `FROM` wrong.

> **Healthcheck gotcha (FAQ):** if your base image inherits a `HEALTHCHECK` that curls a server, Harbor's `sleep infinity` override makes it unhealthy and `docker compose up --wait` aborts with a RuntimeError. Override with an always-true probe: `HEALTHCHECK --interval=5s --timeout=2s --start-period=2s --retries=2 CMD true`.

### 6.5 `environment/docker-compose.yaml` — copy verbatim, never touch

```yaml
services:
  main:
    build: { context: ${CONTEXT_DIR} }
    image: ${MAIN_IMAGE_NAME}
    command: [ "sh", "-c", "sleep infinity" ]
    network_mode: ${NETWORK_MODE:-bridge}
    environment: [ "TEST_DIR=${TEST_DIR}" ]
    volumes:
      - ${HOST_VERIFIER_LOGS_PATH}:${ENV_VERIFIER_LOGS_PATH}
      - ${HOST_AGENT_LOGS_PATH}:${ENV_AGENT_LOGS_PATH}
    deploy: { resources: { limits: { cpus: ${CPUS}, memory: ${MEMORY} } } }
```

### 6.6 `tests/config.json` — THE CONTRACT

Every field and what it's for:

| Field | What it is |
|---|---|
| `repo` | your repo name, e.g. `"amole"` |
| `instance_id` | unique id, e.g. `"instance_you__amole-62c606b__ban-enforcement"` |
| `base_commit` | the commit the Dockerfile resets to — the starting state |
| `test_patch` | a unified **git diff** that adds your new test file(s). This is what the harness applies. |
| `fail_to_pass` | node IDs that must go **RED→GREEN** (the new behavior) |
| `pass_to_pass` | node IDs that must stay **GREEN** (regression guards) |
| `selected_test_files_to_run` | which test file(s) pytest runs |
| `patch` | the **gold solution** diff (same content as `solve.sh`) |
| `problem_statement` | one-line summary |
| `before_repo_set_cmd` | `git reset --hard <base> && git clean -fd && git checkout <base>` |

Node IDs **must exactly match** what pytest prints (`path::Class::test_name`). Mismatched names are a top cause of silent validation failure (FAQ). Always copy them from a real local run, never hand-type them.

### 6.7 `tests/{test.sh, run_script.sh, parser.py}` — the harness, copy verbatim

- **`test.sh`** — the orchestrator. Writes reward 0 immediately; extracts `test_patch` from config and `git apply`s it (deleting any pre-existing file at the new path first, so *your* gold tests are authoritative — this closes the "agent drops a trivially-passing test" hole); runs `run_script.sh`; runs `parser.py`; checks that **every** name in `fail_to_pass ∪ pass_to_pass` is PASSED; writes reward 1 only then.
- **`run_script.sh`** — thin pytest wrapper: `pytest -v --tb=short --no-header -rA <files>`.
- **`parser.py`** — converts pytest stdout/stderr to `{"tests":[{"name","status"}]}`.

You do **not** edit these. They are identical across tasks. (One subtlety from the FAQ: do **not** also re-apply `test_patch` inside `test.sh` if the platform already applies it during build — double-apply fails. The accepted `test.sh` applies it once, which is correct for the local/standalone flow.)

### 6.8 `solution/solve.sh` — the gold fix, inline

```bash
#!/bin/bash
set -euo pipefail
cd /app/amole
# Embed the diff INLINE (heredoc). Do NOT read it from /tests/config.json —
# during the Oracle trial /tests is not mounted in the agent context, so a
# runtime read returns empty and the oracle silently runs against base code.
git apply --verbose <<'GOLD_PATCH_EOF'
<the full unified diff of your fix>
GOLD_PATCH_EOF
```

For tasks that *create new files* (like task-10's `authentication.py`), it's cleaner and equally valid to `cat > file <<'PY' … PY` and then `sed`/`python` to wire it in, rather than a git diff. Either way: **self-contained, no dependency on `/tests` at runtime.**

---

## 7. The tools: Harbor, the verifier, NOP & Oracle

> Official: `training/Test locally with Harbor.md`. Install: `uv tool install harbor` (or `pip install harbor`); ensure `~/.local/bin` is on `$PATH`.

**Harbor** is AfterQuery's local runner. It builds your `environment/` image, starts the container per `docker-compose.yaml`, and runs your verifier (`test.sh`) under two agents:

- **NOP (null) check** — runs the verifier against the **base repo with no fix applied**. Your `fail_to_pass` tests **must FAIL** here → reward **0**. If they pass on base, your tests are weak or aimed at behavior that already exists (gate 5 fails). 
- **Oracle check** — applies your `solve.sh`, then runs the verifier. Everything **must PASS** → reward **1**. If not, your fix and tests don't line up (gate 6 fails).

`reward.txt` lives at exactly `/logs/verifier/reward.txt` and contains `0` or `1`. The mantra, from both the FAQ and brutal experience: **NOP reward 0, Oracle reward 1 — locally — before you ever submit.** The platform's outer gates cost real money; these two checks are free.

You don't strictly need Harbor itself to get this signal — you can replicate it with plain Docker (build the base image, run the container, exec `test.sh` with and without `solve.sh`). That's exactly how the paid tasks on this account were validated. What matters is that you reproduce NOP=0 / Oracle=1 locally by *some* deterministic means.

---

## 8. The heart of the craft: what makes a task HARD-but-FAIR

This section is the reason this guide exists. Gates 7–9 (too-easy / difficulty / fairness) are where ~5 of this account's tasks died and where the one winner succeeded. The training says "non-obvious root cause, cross-cutting, edge cases." True but vague. Here is the **operational law**, learned by losing:

### 8.1 The Law

> **A task is hard if and only if the *obvious* fix is INSUFFICIENT in a way the solver won't anticipate, AND your test catches the insufficiency.**

Blast radius (touching many files) does **not** make a task hard. Soft-delete (the winner) touched 4 files; plenty of *rejected* tasks touched more. What made soft-delete hard was that **the obvious fix is wrong**:

- The naive reading of "make delete hide the listing" → "add a `is_deleted` filter in the listing **view**." 
- That passes a retrieve→404 test but **FAILS** the test that asserts `Listing.objects.filter(id=...).exists()` is `False` *while the raw DB row count is unchanged*. Only overriding the **default manager** (`objects`) satisfies both at once.
- The instruction never says "manager." The solver has to *discover* that the contended surface is the default queryset, not the view.

That is the shape you are hunting: **a test that pins the real mechanism by asserting something the obvious fix can't produce.**

### 8.2 The four reliable patterns (with the trap that makes each hard)

1. **DB-level invariant the service "already" enforces in Python.** The service code *looks* correct; the real fix is a DB constraint, and the test writes a violating row **directly via the ORM, below the service**, so any service patch is dead code. *Trap:* the naive `UniqueConstraint`/`CheckConstraint` is often *one-directional* and rejects legitimate rows → your regression guards catch it.

2. **Default-manager / queryset override** (the soft-delete winner). *Trap:* a view-level filter passes the "hidden from feed" test but fails the "default manager excludes it" assertion.

3. **Enforcement at the right layer.** (task-10 on this account — **and a cautionary tale, see §13.5**.) The bug: bans aren't enforced. *Two stacked traps:* (a) blocking at **login** is insufficient because JWT is stateless — a token issued *before* the ban keeps working; (b) the obvious global `DEFAULT_PERMISSION_CLASSES` is **silently overridden** by every viewset's own `permission_classes`. *But this task still failed the too-easy gate* — because the instruction ("enforce on every authenticated request") signposted the auth layer too clearly. A trap only counts if the right mechanism stays non-obvious **even after reading the instruction**. Read §13.5 before you trust your own "I proved it's hard" instinct.

4. **State-machine asymmetry.** Acting on a terminal state must be a silent **no-op** in one case and a hard **error** in another. *Trap:* the naive "raise on any terminal state" fails the no-op assertion; "return on any" fails the error assertion.

### 8.3 What is DEAD (don't waste runs)

- **Stated business logic.** If the instruction states the rule and the fix is to implement the stated rule in one obvious place, Sonnet 4.6 aces it. This account lost task-1 (free-listing cap), task-3, task-7 (message-delete + unread), task-8 (PII redaction) exactly this way — all "too easy, 5/5 probe solved."
- **Concurrency races that don't trip a deterministic DB effect.** This is subtle and cost real time. A threaded "two users race" test is only reliably reproducible if the race trips a **deterministic DB constraint** (a `CHECK`/`UNIQUE` violation → `IntegrityError` every time). A *pure logical* TOCTOU (e.g., "two accepted offers" where two accepted rows is not itself a DB violation) gets **serialized by the GIL + fast local Postgres** and never reproduces. Measured on this account: a trade-accept race produced **0 oversells over 35 threaded runs**. That task (task-9) was abandoned. Meanwhile a stock-oversell race that tripped a `stock_quantity >= 0` CHECK reproduced **every time** (NOP=fail, Oracle=pass) — because the constraint made it deterministic.
- **Pure refactors with no behavioral delta.** If a commit refactored already-correct code (e.g., moved inline `if completed: noop` logic into a policy dict) there is **no testable difference** between base and gold → no valid NOP → not a task. Most "improvement" commits in a mature repo are this. Verify there's a real behavioral delta before building.

### 8.4 How to find carvable tasks in your repo's history

```bash
# Changes that MODIFIED existing behavior carve best (bug fixes, hardening, refactors-with-delta):
git log <base> --oneline --no-merges | grep -iE "fix|harden|enforce|prevent|guard|validate"

# For a candidate commit, confirm a real behavioral delta exists at its PARENT:
git show <commit>^:path/to/file.py   # the "naive before"
git show <commit>:path/to/file.py    # the "gold after"
# If the parent already does the right thing → dead (refactor). If it's genuinely
# missing the behavior → carvable. base_commit = <commit>^, gold = <commit>.

# Or find ADDITIVE gaps: a model field / flag that exists but is enforced nowhere.
git grep -n "is_banned"   # → exists on the model… but checked in how many places?
```

---

## 9. Writing `instruction.md` (the AI-text gate is real)

Gate 2 rejects prose that reads like an AI assistant wrote it. This is not a minor stylistic note — over-polished instructions get bounced.

### 9.1 Side-by-side: what fails vs what passed

**❌ FAILS the AI-text gate (and leaks the mechanism, failing fairness/too-easy too):**
```
## Problem Statement
The `delete_listing` method currently performs a hard delete. We need to
implement a robust soft-delete pattern to ensure data integrity. 

### Requirements
1. Add an `is_deleted` boolean field and a `deleted_at` timestamp.
2. Implement a custom model manager that filters out deleted records.
3. Ensure the API returns a 404 for deleted listings.
4. In conclusion, this provides a more idiomatic approach to deletion.
```
Three things are lethal here: the **AI register** (numbered hierarchy, "robust," "idiomatic," "In conclusion"), and it **hands the solver the mechanism** (`is_deleted`, "custom model manager") — which simultaneously makes it too easy and tells the agent exactly what to do.

**✅ PASSED (the actual paid instruction, §6.1):** a casual bug report, typos intact, describes only observable behavior, names no mechanism.

### 9.2 The rules that keep you on the right side

- Write it the way you'd **describe a bug to a teammate in Slack** — 2–4 short paragraphs.
- **Symptom first** ("a seller deleted their listing but it still showed in the feed").
- Describe **what** the system should do (status codes, what's visible, what survives), never **how** (no class names, no field names, no pattern names).
- **No bullet hierarchies of three-noun phrases. No "robust / idiomatic / leverage / in conclusion / seamless."**
- A few natural imperfections (a typo, a run-on) *help*. Don't manufacture them, but don't sand them all off either.
- Pre-check with GPTZero (gptzero.me). If it reads >80% human there, it usually passes. If flagged, rewrite by hand — do not just run it through a "humanizer."
- Include the standard guardrail line: *"Make your changes in the application source code only… Do not edit, add, or remove any test."*

### 9.3 The fairness corollary

Gate 9 (fairness): **every assertion your tests make must be findable in the instruction.** If a test checks an exact error string, the instruction must give that string (or assert the field/type instead). If a test checks ordering, the instruction must state the tiebreaker. The rule of thumb from the FAQ: *"If the instruction doesn't say it, either add it or stop testing it."* This is in direct tension with §9's "don't name the mechanism" — the resolution is: **specify observable behavior precisely; leave the implementation mechanism unspecified.** Behavior is fair to test; mechanism is what makes it hard.

---

## 10. Writing the tests and encoding `test_patch`

### 10.1 Test design that survives the gates

- **≥ 6 tests** is the safe floor (the platform has bounced tasks for "test_patch only adds 5 tests; at least 6 required"). Split: a few `fail_to_pass` (the new behavior) + a few `pass_to_pass` (regression guards).
- **`fail_to_pass`** must genuinely fail on base. If it passes on base, you're testing existing behavior (gate 5).
- **`pass_to_pass`** must pass on base *and* after the fix. These are your fairness/regression anchors and they also catch the "naive one-directional fix" (see §8.2).
- **Assert exact outcomes, not "something happened."** The FAQ's weak-test list: `hasattr`/`callable`, import-only, "API returned something," "non-empty." Strong: exact status codes, exact JSON, exact counts, persisted DB state, raw-SQL row counts.
- **Gut check (from the FAQ):** *would an empty stub or an obviously-wrong implementation still pass?* If yes, your tests are too loose and the task will be rejected as weak — or pass too-easily.
- **Bypass the obvious path.** The single most valuable trick (from the soft-delete winner): write at least one test that operates **below** the layer the solver will naively patch — direct ORM writes, raw SQL counts, a pre-issued token — so the lazy fix is literally dead code in your test.

### 10.2 Encoding `test_patch` into `config.json` — the part other guides skip

`test_patch` is a **unified git diff, as a JSON string** (newlines escaped). Don't hand-write it. Generate it:

```bash
# 1) Put your new test file in place inside a checkout at base_commit, then:
git -C /path/to/repo add backend/accounts/test_ban_enforcement.py
git -C /path/to/repo diff --cached backend/accounts/test_ban_enforcement.py > test_patch.diff
#   (or for a brand-new untracked file, craft the "new file mode" diff —
#    a hash-object + header, or just `git add` then `git diff --cached`.)

# 2) Embed it into config.json with proper JSON escaping (Python does it for you):
python3 - <<'PY'
import json
cfg = json.load(open("tests/config.json"))
cfg["test_patch"] = open("test_patch.diff").read()
cfg["patch"]      = open("solution/gold.patch").read()
json.dump(cfg, open("tests/config.json","w"), indent=2)
PY
```

Then **verify the platform sees your tests** — the FAQ's quickest debug for "test_patch only adds 0 tests":

```bash
git -C /path/to/repo checkout <base_commit>
git -C /path/to/repo apply test_patch.diff
git -C /path/to/repo diff <base_commit> -- tests/   # this is EXACTLY what the platform reads
```

If that diff is empty, your `test_patch` is empty (common when you edit in the website editor — use the **import** function instead, the in-site editor flattens patches). Other zero-tests causes: files/functions not prefixed `test_`; missing `tests/__init__.py`; a syntax error in the test file (the harness reads a non-loading file as zero tests).

---

## 11. The local validation loop (do this before every submit)

This is the loop that protects your metered runs. Every task on this account that got paid went through exactly this; every task that *skipped* it wasted a platform run.

```bash
# 0) Build a LOCAL stand-in image at the base_commit (the frozen AQ image can't be
#    pulled outside AQ, so build an equivalent: same deps, SQLite test settings).
docker build -f environment/Dockerfile.local -t myrepo-base:<base> .

# 1) NOP — base code, no fix. The fail_to_pass tests MUST FAIL.
docker run -d --name nop myrepo-base:<base> sleep infinity
docker cp tests/. nop:/tests/
docker exec nop bash /tests/test.sh ; docker exec nop cat /logs/verifier/reward.txt
#   EXPECT: RESULT FAILED, reward 0.

# 2) ORACLE — apply the gold fix, rerun. Everything MUST PASS.
docker exec nop bash /solution/solve.sh
docker exec nop bash /tests/test.sh ; docker exec nop cat /logs/verifier/reward.txt
#   EXPECT: RESULT PASSED, reward 1.

# 3) NAIVE-FIX PROOF (your local stand-in for the difficulty gate, which you
#    CANNOT run locally). Implement the *obvious* fix and confirm it still leaves
#    fail_to_pass RED. If the obvious fix passes, the task is too easy — fix the
#    test to pin the real mechanism BEFORE submitting.

# 4) REGRESSION — run the whole touched app suite under the Oracle; confirm the
#    gold adds zero new failures vs base (compare counts; pre-existing failures on
#    a stand-in image are fine as long as base == oracle for them).
```

The **naive-fix proof** (step 3) is the highest-leverage *disqualifier* in this whole guide — but mind its limit. It can only **disprove** hardness: if your own obvious fix passes, the task is definitely too easy, and you've saved a metered run. It **cannot prove** hardness, because you only test the wrong fixes *you* thought of, while Sonnet explores the whole solution space (this is exactly how task-10 slipped through — see §13.5). So pair step 3 with the **non-obviousness gut check**: *after reading my instruction, is the correct mechanism still ambiguous — multiple plausible places to put the fix, only one of which my tests accept?* Spend the metered run only when **both** hold: the obvious fix fails locally **and** the mechanism stays non-obvious after the instruction.

---

## 12. Failure recovery — a decision tree for every rejection

The existing guides index failures by *symptom*. Here they're indexed by **"it failed, what do I do next?"** — including whether to fix inline or pivot the whole task.

```
TASK CAME BACK. Which gate?
│
├─ Gate 1 SIMILARITY ("Repo saturated" / too similar)
│   → Your last tasks are too alike. DO NOT paraphrase — the embedding won't move.
│     Pick a DIFFERENT domain AND a different mechanism (see §14). PIVOT, don't patch.
│
├─ Gate 2 AI-TEXT
│   → Rewrite instruction.md by hand in Slack-voice (§9). Drop all bullet hierarchies
│     and "robust/idiomatic/in conclusion." Re-check on GPTZero. INLINE fix.
│
├─ Gate 4 IMAGE BUILD ("fatal: not a git repository" / build error)
│   → .git in .dockerignore? remove it. git missing in base? add it. >500MB? slim base.
│     Recreate .git/refs/{heads,tags} (§6.4). INLINE fix. (Pure plumbing — never should
│     have reached the platform; you skipped §11.)
│
├─ Gate 5 NOP ("test_patch adds 0 tests" / NOP passed when it should fail)
│   → 0 tests: names not test_*, missing __init__.py, syntax error, empty diff (§10.2).
│     NOP passed: your fail_to_pass tests pass on base → they test existing behavior, or
│     they're weak. Make them assert the NEW outcome below the obvious layer. INLINE fix.
│
├─ Gate 6 ORACLE (RewardFileNotFound / ModuleNotFound / F2P didn't flip)
│   → RewardFileNotFound = plumbing (§6.4 ENTRYPOINT, reward early in test.sh, .git present).
│     ModuleNotFound = PYTHONPATH / conftest / pip install -e. F2P didn't flip = your
│     solve.sh doesn't actually make the tests pass; fix the patch. INLINE fix.
│
├─ Gate 7/8 TOO EASY (5+/10 solved)  ← the expensive one
│   → The obvious fix works. You must change the TASK, not the wording. Either:
│     (a) move a test BELOW the obvious patch layer so the lazy fix is dead code, or
│     (b) add a subtlety that makes the naive fix wrong (one-directional constraint,
│         stateless-token, overridden-default), or
│     (c) ABANDON and pick a harder commit. Run the §11 naive-fix proof before resubmitting.
│     Do NOT just delete hints hoping it gets harder — if the fix is still obvious once
│     discovered, hints don't matter. This usually means PIVOT.
│
├─ Gate 8 TOO HARD / UNVERIFIABLE (0/10)
│   → Either unsolvable as specified or the harness is broken. Confirm solve.sh + tests
│     line up locally (Oracle green). Then ADD ONE hint at a time to instruction.md until
│     it enters 1–4/10. INLINE fix, but change ONE thing per run (runs are metered).
│
└─ Gate 9 FAIRNESS (tests assert something the instruction never said)
    → Two legal fixes only: (a) add the missing behavior to instruction.md, or
      (b) loosen/remove the test. Never leave a hidden requirement. INLINE fix.
```

**The meta-rule:** plumbing gates (4,5,6) should *never* reach the platform — §11 catches them for free. Craft gates (7,8,9) are where you legitimately iterate, but **change one thing per metered run** and re-prove locally first.

---

## 13. War stories: what got paid, what got killed, and why

Concrete, because patterns stick better than rules.

| Task | Domain / mechanism | Outcome | Lesson |
|---|---|---|---|
| **Listing soft-delete** | default-manager override + migration | **APPROVED, PAID $75** | The obvious view-filter fix fails the "default manager excludes it / raw row survives" test. Mechanism unnamed in instruction. This is the template. |
| Free-listing cap | service business logic | Rejected: too easy 5/5 | A stated COUNT policy in one obvious place. Sonnet aces it. |
| Message-delete + unread | endpoint + counter | Rejected: too easy | CRUD + a counter decrement. Obvious. |
| PII redaction | serializer conditional fields | Rejected: too easy | Textbook DRF. Obvious. |
| **task-9 trade-accept race** | concurrency (logical TOCTOU) | **Abandoned before submit** | 0 oversells / 35 threaded runs. Pure logical races serialize under GIL+fast PG. Not reproducible ⇒ not testable. |
| **task-10 ban enforcement** | auth-layer enforcement | **Built, locally validated, submitted** | Double trap: login-block insufficient (stateless JWT) + global permission overridden by every view ⇒ must fix at auth layer. Naive global-permission fix proven to leave tests red. |
| **task-10 ban enforcement** | auth-layer enforcement | **Submitted, FAILED too-easy gate** | The local naive-fix proof passed but only tested fixes *I* thought of. Sonnet found the auth-layer fix anyway — because the instruction signposted it. See §13.5. |
| `WeCare` **repo** | — | Rejected (final) | AI-burst history. |
| `alx-project-nexus` **repo** | — | Rejected (final), **90% coverage** | "Insufficient test coverage" was a *proxy* for AI-burst quality. Coverage % is not the real bar. |

### 13.5 The most important failure: task-10 and the limits of a local "hardness" proof

This account built task-10 (ban enforcement), ran the §11 **naive-fix proof**, watched two obvious fixes leave the tests red, concluded "this is genuinely hard," submitted — and it **failed the too-easy gate (Sonnet solved it ≥5/10).** This is the single most instructive failure in this guide, so internalize it:

- **A local naive-fix proof only tests the wrong fixes *you* thought of.** I checked "block at login" and "global permission class." Sonnet 4.6 doesn't limit itself to my guesses — it explores the whole solution space and found the correct auth-layer fix directly.
- **Why it found it: the instruction signposted the mechanism.** The instruction said the ban must "take hold immediately on *every authenticated endpoint*." To a competent agent that sentence *is* a pointer to the authentication/permission layer. The "trap" (login-block insufficient, global-permission overridden) only filtered out the *lazy* solver; it did nothing to a *good* one, because the right layer was discoverable from the spec.
- **Contrast with the winner.** Nothing in the soft-delete instruction points at "default manager." The behavior (204 / 404 / row-survives / hidden-from-feed) is describable *without* hinting at the mechanism, and there are several plausible-but-wrong places to put the fix. The mechanism stays non-obvious *after* reading the instruction. That is the bar.

**The corrected rule (this supersedes any earlier confidence about local proofs):**

> The naive-fix proof in §11 is necessary but **not sufficient**. It can only *disprove* hardness (if the obvious fix passes, it's definitely too easy), never *prove* it. The real test of difficulty is: **after reading my instruction, is the correct mechanism still non-obvious — are there several reasonable places a good engineer might put the fix, only one of which my tests accept?** If the instruction's behavior spec implies the fix location, it's too easy no matter how many lazy fixes you ruled out.

Practical takeaways when you build:
- After writing the instruction, ask: *"if I were a strong engineer reading only this, would I know which layer/mechanism to reach for?"* If yes, either the behavior spec is leaking the mechanism, or the mechanism genuinely isn't hard — pivot.
- Prefer tasks where **multiple plausible implementations exist and only one is correct**, and the instruction can't disambiguate them without naming the mechanism (which fairness forbids you from doing). That gap — plausible-but-wrong vs correct — is the hardness.
- Treat a passed local naive-fix proof as "not *obviously* too easy," not "proven hard." Spend the metered run only when you also pass the "is the mechanism still non-obvious after the instruction?" gut check.

**The three durable lessons:**
1. **Only repos with organic human history pass Phase 1.** AI-enhancing a repo to impress the reviewer gets it permanently rejected.
2. **"Too easy" is the default fate of any task whose obvious fix works.** Engineer insufficiency into the obvious fix, and pin it with a test that bypasses the obvious layer.
3. **Concurrency is a trap unless the race trips a deterministic DB constraint.** Otherwise it won't reproduce and you'll waste days.

---

## 14. Diversifying to 5+ tasks without tripping similarity

You need 5 approvals per repo for the money, but no more than ~5 similar tasks are allowed and the cosine/Levenshtein gate is unforgiving. Deliberately spread across **both** axes:

- **CRUD-operation axis** (the FAQ's explicit ask): *bug fixes (UPDATE)*, *features (CREATE/ADD)*, *refactors-with-delta (UPDATE/DELETE)*, *perf (UPDATE/ADD/DELETE)*.
- **Domain axis:** in a marketplace like amole — accounts, listings, trades, orders, reviews, messaging, notifications. One task per domain keeps embeddings far apart.
- **Mechanism axis:** DB constraint, manager/queryset override, auth-layer enforcement, state-machine guard, serializer redaction, signal/denormalization maintenance.

A 5-task spread that would *not* trip similarity: (1) soft-delete via manager [listings/refactor], (2) ban enforcement at auth layer [accounts/security], (3) a biconditional CHECK invariant [orders/bugfix], (4) a state-machine no-op-vs-error asymmetry [trades/feature], (5) a denormalized-counter maintenance fix [messaging/bugfix]. Five domains, five mechanisms, four CRUD classes.

> Honest caveat from this account: a *mature, well-built* repo can be **thin on carvable tasks** because good code has already closed the obvious gaps. Vet for 5 before you commit (§4.4). If a repo only yields 1–2 real tasks, that's a strategic dead-end given the 5-to-get-paid rule.

---

## 15. The pre-submit checklist

Print this. Run it every single time, before the metered platform sees your task.

```
REPO (once per repo, Phase 1)
[ ] Private, you own the IP, never public/forked/mirrored
[ ] Real sustained HUMAN commit history (NOT an AI burst)
[ ] .dockerignore does NOT contain .git
[ ] tests/ exists (plural), default branch is main
[ ] Extracted my own zip; .git/HEAD present at repo-name/.git/
[ ] I believe this repo has ≥5 carvable hard tasks in its history

TASK (every task, Phase 2)
[ ] instruction.md: Slack-voice, symptom-first, NO mechanism named, NO AI register
[ ]   → passes GPTZero >80% human
[ ] Every test assertion is findable as BEHAVIOR in instruction.md (fairness)
[ ] ≥6 tests; fail_to_pass + pass_to_pass node IDs copied from a REAL pytest run
[ ] At least one test operates BELOW the obvious patch layer (bypass trick)
[ ] test_patch verified non-empty: `git diff <base> -- tests/` shows my tests
[ ] config.json: base_commit, test_patch, patch, fail_to_pass, pass_to_pass all set
[ ] Dockerfile FROM = exact frozen image ref; resets to base_commit; reward plumbing present
[ ] solve.sh is self-contained (inline patch, no /tests dependency)

LOCAL PROOF (the money-saver — never skip)
[ ] NOP  → fail_to_pass FAIL, reward 0
[ ] ORACLE → all pass, reward 1
[ ] NAIVE-FIX PROOF → the obvious fix still leaves fail_to_pass RED (disproves "too easy")
[ ] NON-OBVIOUSNESS CHECK → after reading ONLY my instruction, the correct mechanism/layer is
[ ]   still ambiguous (several plausible fixes, only one passes). If the spec implies the fix
[ ]   location, it's too easy — pivot. (This is the gate task-10 failed; see §13.5.)
[ ] REGRESSION → gold adds zero new failures vs base on the touched suite
[ ] Diversity: different domain+mechanism than my last tasks (similarity)
```

---

## 16. Quick reference card

```
PAYMENT          $75/approved task · +$300 at 5 approvals/repo · 0–4 approvals = $0 · cap 20
THE TRAP         A repo you can't take to 5 pays nothing. Vet for 5 before you start.
REPO REJECTION   Final. Unappealable. No resubmit. AI-burst history = death.
RUNS             ≤5 Opus/difficulty runs per task. Each ~$5–10, audited. Prove locally first.

THE LAW          Hard = the OBVIOUS fix is insufficient AND a test catches it.
                 Blast radius ≠ hard. Stated logic = too easy. 
WINNING SHAPES   DB invariant (test below service) · manager override · auth-layer enforcement
                 · state-machine no-op-vs-error asymmetry
DEAD ENDS        stated business logic · logical-TOCTOU concurrency · refactors w/ no delta

LOCAL LOOP       NOP=0 → ORACLE=1 → NAIVE-FIX still RED → mechanism still non-obvious → no regress
                 (naive-fix proof DISPROVES easy; it can't PROVE hard — §13.5)
REWARD FILE      /logs/verifier/reward.txt  (0 or 1); seed it early; .git must be present

INSTRUCTION      Slack-voice bug report. Symptom first. WHAT not HOW. Never name the mechanism.
                 No "robust/idiomatic/in conclusion." Typos OK. GPTZero >80% human.
TESTS            ≥6. Exact outcomes, not "something". One test BELOW the obvious layer.
                 Node IDs copied from a real run. Verify `git diff <base> -- tests/` non-empty.
DIVERSIFY        Spread across CRUD class × domain × mechanism. Paraphrase ≠ diversity.

GATES            1 similarity · 2 AI-text · 3 quality · 4 build · 5 NOP · 6 oracle
                 7 too-easy · 8 difficulty(1–4/10) · 9 fairness
PLUMBING GATES   4,5,6 — should NEVER reach the platform; §11 catches them free.
CRAFT GATES      7,8,9 — iterate here, ONE change per metered run, re-prove locally.
```

---

### Source notes & where to go deeper

- **Live source of truth:** the Project Silver **FAQ** (pinned in the project) supersedes everything, including the AQ website. Re-read it when anything here seems stale.
- **Official training (in this folder):** `training/Submit a repository.md`, `training/Write a Dockerfile.md`, `training/Author a task.md`, `training/Test locally with Harbor.md`, `training/What makes a good task.md`, `training/The rubric.md`, `training/The validation pipeline.md`, `training/Common pitfalls.md`, `training/Practical workflow.md`. Plus the official index `aq-instructions.md`.
- **Companion guides (in this folder):** `PROJECT_SILVER_GUIDE.md` (good concise reference), `AMOLE_TASK_PLAYBOOK.md` (amole-specific roadmap + timing estimates).
- **Canonical examples to study:** the official `example-repo+task.zip` from the project; the SWE-bench Pro public dataset at `https://hub.harborframework.com/datasets/cais/swebenchpro/latest`; the worked example task analyzed for this guide (`task-1`, a pyawips SIGMET feature-add — note its dual-purpose `parser.py` and inline-heredoc `solve.sh`).
- **The proven template on disk:** `amole-tasks/task-02-listing-soft-delete/` — the accepted, paid package. Clone it.
- **Harbor docs:** `https://www.harborframework.com/docs`. **AI pre-check:** `https://gptzero.me/`.
- **Help, in order (FAQ):** the Silver Help Assistant (`experts.afterquery.com/projects/silver/help`) → daily help threads → escalate only as the FAQ's POC table directs. Time in Slack is unpaid time.

*Written from the trenches of one accepted repo, two rejected repos, one paid task, and a pile of instructive failures. The map is honest about where the dragons are.*
