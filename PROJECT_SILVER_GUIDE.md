# Project Silver Field Guide (Straight to the Point)

This guide is a clean, practical summary of Project Silver rules, workflow, and failure modes. It is based on the official instructions and training modules, the Project Silver FAQ you shared, and a real task example (listing soft delete). Use it as the single reference for repo submission and task authoring.

## Quick Navigation
- [What Project Silver Is](#what-project-silver-is)
- [What You Get Paid For](#what-you-get-paid-for)
- [Non-Negotiables (Repo and IP)](#non-negotiables-repo-and-ip)
- [Repo Submission Checklist](#repo-submission-checklist)
- [Tooling and Local Setup](#tooling-and-local-setup)
- [Task Lifecycle in 10 Steps](#task-lifecycle-in-10-steps)
- [Task File Map (What Each File Does)](#task-file-map-what-each-file-does)
- [Picking a Base Commit](#picking-a-base-commit)
- [Tests That Actually Pass Review](#tests-that-actually-pass-review)
- [Validation Pipeline (All Stages)](#validation-pipeline-all-stages)
- [Rubric in Plain English](#rubric-in-plain-english)
- [What Works vs What Does Not](#what-works-vs-what-does-not)
- [Common Failures and Fixes](#common-failures-and-fixes)
- [Difficulty Calibration](#difficulty-calibration)
- [Case Study: Listing Soft Delete](#case-study-listing-soft-delete)
- [Final Pre-Submit Checklist](#final-pre-submit-checklist)
- [Help, Escalation, and Timing](#help-escalation-and-timing)

## What Project Silver Is
Project Silver is a task creation program for SWE-bench style coding tasks. Each task is a real bug or feature request inside a private codebase, with tests and a reference solution so the platform can score agents automatically.

The flow is always:
1) Submit a private repo (with full git history).
2) Author tasks against specific commits of that repo.
3) Let the pipeline validate the task across multiple stages.

## What You Get Paid For
- Approved task: $75 each.
- Approved repo bonus: $300 only after 5 approved tasks on that repo.
- Cap: 20 approved tasks per repo.

Important: If you have only 0 to 4 approved tasks on a repo, you do not get the $300 repo bonus.

## Non-Negotiables (Repo and IP)
- Your repo must be private or off public hosting entirely.
- You must own the IP fully. No employer code, no forks of public repos, no shared IP.
- The repo must be real and substantive. No toy projects, no tutorials, no scaffolds.
- Your zip must include the .git directory with full history.
- The tests folder must be named tests/ (not test/).
- .dockerignore must not exclude .git.

## Repo Submission Checklist
- Zip structure must be repo-name/ at top level, not a flat zip and not a double-wrapped zip.
- Include .git/ with HEAD and objects intact.
- Keep .dockerignore clean, do not exclude .git.
- Confirm tests/ exists and can run locally.
- Confirm the default branch is main (master can cause history issues in the pipeline).

## Tooling and Local Setup
These tools are mandatory for local validation:
- Docker (for local task runs).
- Harbor CLI (local task verifier).
- Git (to select base commits and check history).

Install Harbor:
```
uv tool install harbor
# or
pip install harbor
```

Local validation commands:
```
# Null run: fail_to_pass tests must FAIL
harbor run -p ./my-task -a nop

# Oracle run: all required tests must PASS
harbor run -p ./my-task -a oracle
```

If either run fails locally, do not submit yet.

## Task Lifecycle in 10 Steps
1) Pick a real bug or feature.
2) Identify the commit before the fix (base_commit).
3) Download the template for that commit.
4) Write instruction.md first.
5) Write the tests and config.json.
6) Write solve.sh with a clean diff.
7) Run Harbor nop and oracle locally.
8) Re-check the rubric in plain English.
9) Import the task zip and submit.
10) Fix based on pipeline feedback, not guesswork.

## Task File Map (What Each File Does)
This section explains every required file and shows short examples.

instruction.md (what the agent sees)
Short, concrete description of the bug and expected outcome. No pseudocode.
```
When a user deletes their own listing, the API returns 204 but the listing
still shows up in the feed. After deletion, public reads must return 404 and
the listing must disappear from normal queries while staying in the database
for admin review.
```

reference_plan.md (why the task is fair)
Root cause, intended fix area, and test coverage summary for reviewers.
```
Root cause: ListingService.delete_listing does a hard delete.
Fix: soft delete with is_deleted + deleted_at and hide from default manager.
Tests: service layer soft delete + API 404 + keep existing behaviors intact.
```

task.toml (metadata)
```
[metadata]
author_name = "Your Name"
author_email = "you@email.com"
difficulty = "medium"
category = "bug-fix"
```

environment/Dockerfile (task image)
Uses the approved repo image, adds only what the task needs.
```
FROM us-docker.pkg.dev/afterqueryai/silver-repo-images/your-repo:v1@sha256:...
USER root
RUN apt-get update && apt-get install -y --no-install-recommends git
```

solution/solve.sh (reference fix)
Applies a unified diff using git apply.
```
#!/usr/bin/env bash
set -euo pipefail
cd /app/your-repo
git apply --verbose <<'PATCH'
... unified diff ...
PATCH
```

tests/config.json (the contract)
Defines base_commit, test lists, and test_patch.
```
{
  "base_commit": "abcdef1234567890",
  "fail_to_pass": ["tests/test_bug.py::test_expected_behavior"],
  "pass_to_pass": ["tests/test_existing.py::test_existing_behavior"],
  "test_patch": "diff --git a/tests/test_bug.py b/tests/test_bug.py\n...",
  "selected_test_files_to_run": ["tests/test_bug.py", "tests/test_existing.py"]
}
```

tests/test.sh (verifier entrypoint)
Applies test_patch, runs tests, parses output, writes reward.
```
# apply test_patch
# run /tests/run_script.sh
# parse with /tests/parser.py
# write /logs/verifier/reward.txt
```

tests/run_script.sh (runs your tests)
```
python3 -m pytest -v --tb=short --no-header -rA
```

tests/parser.py (parses test output)
```
RESULT_LINE = re.compile(r"^(PASSED|FAILED|ERROR|SKIPPED)\s+(\S+)")
```

## Picking a Base Commit
- The base commit must be the broken state.
- It must exist in git history (use git cat-file -e <commit> to confirm).
- The fix must apply cleanly to that commit without fuzz.

## Tests That Actually Pass Review
- Tests must run code and assert behavior, not grep files.
- fail_to_pass tests must fail at base and pass after the fix.
- pass_to_pass tests must pass before and after the fix.
- Test names in config.json must match pytest --collect-only output exactly.
- Do not use nondeterministic checks (time, random, network).
- Test coverage for the task must be >= 80% using the standard tool for the language.

## Validation Pipeline (All Stages)
Similarity and repo saturation checks run early:
- Cosine >= 0.75 OR Levenshtein >= 0.70 means rejected.
- If you hit repo saturation, you are too similar to your last 5 tasks.

Stage 1: AI text check
- instruction.md must read like a human wrote it.

Stage 2: Quality review
- Must be Accept or Strong Accept on all 11 criteria.

Stage 3: Image build
- Dockerfile must pass static rules and build in the cloud.

Stage 4: NOP check
- fail_to_pass must FAIL on the base repo.

Stage 5: Oracle check
- fail_to_pass and pass_to_pass must PASS with the reference solution.

Stage 6: Easiness + difficulty probe
- If the agent solves it in one attempt or 5+ out of 10, it is too easy.
- 0 out of 10 is unverifiable; the acceptable range is 1 to 4 out of 10.

## Rubric in Plain English
- Verifiable: deterministic tests.
- Well specified: instruction is enough to reproduce the outcome.
- Solvable: doable within repo scope and time.
- Genuinely difficult: non-obvious root cause.
- Behavioral: tests run the code and verify outcomes.
- Outcome-first: instruction says what, not how.
- Aligned: tests match instruction exactly.
- Human quality: short, natural writing, no AI tone.
- Fair: no hidden requirements.
- Anti-cheat: cannot pass by stubs or hardcoding.
- Reproducible: same result every run.

## What Works vs What Does Not
What works:
- Real bugs with a subtle cause.
- Tests that call the code and assert real outputs.
- Instruction that is short, clear, and outcome-focused.
- Changes that touch multiple files but stay coherent.

What does not:
- Tests that grep source or check variable names.
- Instructions that include pseudocode or step-by-step fixes.
- Tasks that pass without a fix or fail even with the fix.
- Trivial one-line changes or pure formatting tasks.

## Common Failures and Fixes
Repo and build issues:
- .git history not found: remove .git from .dockerignore, fix zip layout, ensure branch is main.
- Image build fails: pin packages, use --no-install-recommends, and avoid forbidden RUN patterns.
- RuntimeError: 1 / Trials: 0: check for VOLUME wiping /app, inherited healthcheck, or missing git in the base image.

Similarity issues:
- Repo saturated or similarity rejection: rewrite the task from scratch and change the CRUD type.

Null or oracle issues:
- Null passes: bug already fixed or tests too weak.
- Oracle fails: patch does not apply cleanly or tests are mismatched.
- test_patch adds 0 tests: diff was empty or test names do not start with test_.
- ModuleNotFoundError: set PYTHONPATH or add a root conftest.py.

Reward file issues:
- RewardFileNotFoundError: create /logs/verifier/reward.txt as root in Dockerfile, and write reward 0 at the top of test.sh before any command that can crash.

## Difficulty Calibration
- Good tasks require multi-file reasoning or a subtle edge case.
- Bad difficulty comes from vague instructions or broken tests.
- If 5+ of 10 runs solve it, raise difficulty by adding realistic edge cases.
- If 0 of 10 solve it, fix the tests or the instruction.

## Case Study: Listing Soft Delete
Problem:
- Delete endpoint returns 204 but listing still shows in the feed.
- Hard delete prevents moderator review.

Expected behavior:
- Delete returns 204.
- Public reads return 404.
- Listing hidden from normal queries but still in the database for admins.

Implementation shape:
- Add is_deleted and deleted_at to Listing.
- Add a default manager that filters deleted listings.
- Add a soft_delete method and use it in the service.
- Keep admin access through all_objects.

Why it passes review:
- Clear instruction, no solution leaks.
- Tests verify real behavior at service and API layers.
- Existing behavior tests remain in pass_to_pass.

## Final Pre-Submit Checklist
- Instruction is 2 to 3 short paragraphs, no AI tone.
- Tests run the code and match the instruction.
- fail_to_pass fails on base, passes with fix.
- Harbor nop and oracle both run cleanly.
- Dockerfile rules satisfied and .git preserved.
- Zip size and file count limits respected.

## Help, Escalation, and Timing
Use the Silver help tool before posting in public threads:
- https://experts.afterquery.com/projects/silver/help

Official entry point if you lose access:
- https://experts.afterquery.com/projects/silver

Respect shared resources:
- Debug locally before running on the AQ server.
- Limit Opus runs to 5 per task.
- Keep the 5:1 ratio of full task runs to tasks in ready-for-review.

Review timing:
- Repo reviews run daily around 12 AM EST (plus or minus 2 hours).
- Task reviews typically land within a few hours, with a 9 PM EST cutoff most days.

Contacts:
- Repo upload and review status: Daily help threads, then Betty C Tai if no response after 3 US business days.
- Task validation and technical issues: Daily help threads.
- Careers and referrals: afterquery.com/careers

You will move faster by fixing issues locally, not by waiting in Slack.
