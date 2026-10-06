---
name: negatives-retro
description: Scorecard for the negative-first workflow. Audits how /negatives is performing — reads the cross-project run ledger, checks git history for deleted or weakened negative tests, probes whether the test fence has teeth, collects regression catches and escaped bugs, and emits a verdict. Use when the user says "negatives retro", "audit the fence", "is the negatives skill working", or asks to evaluate negative tests or the negatives skill in a project.
argument-hint: [optional feature name or module path to focus on]
---

# Negatives Retro — is the fence working?

Evaluation only: this skill NEVER modifies tests, code, or invariants. Findings become recommendations; any change goes through /negatives, which is append-only.

Ledger: `~/.claude/negatives-runs.jsonl` (Windows: `%USERPROFILE%\.claude\negatives-runs.jsonl`). Each line is a JSON object with a `type` field: `"run"` (written by /negatives Phase 5) or `"retro"` (written by this skill, Step 5).

## Step 1 — Read the ledger

Read every line. Summarize:

- Runs per project; failure modes and approved invariants per run
- `checkpoint_edits` distribution — if it is 0 across ALL runs, flag it: the human checkpoint may be rubber-stamping. The control point only has value if judgment is occasionally exercised
- Any `red_check` that is not "clean"
- Prior `retro` lines: the trend of catches vs escapes over time

If the ledger is missing or empty, say so and continue with Steps 2–4 for the current repo only.

## Step 2 — Repo audit (current project)

- Locate the negative tests (`@pytest.mark.negative`, or the stack's mapped idiom) and count them
- Registry check: approved-invariant table and pinned error contract present in the module docstring; deferred long tail documented as a backlog
- Append-only compliance, if this is a git repository:
  - `git log --diff-filter=D --name-only -- <negative test paths>` — deleted test files
  - `git log -p -- <negative test paths>` — scan history for removed `assert` lines, added `skip`/`xfail` marks, loosened expected statuses or detail strings
  - Every hit is a finding: commit, author, what was weakened
- If it is NOT a git repository, append-only is UNVERIFIABLE — report that as a finding in itself and recommend version control

## Step 3 — Catches and escapes (the ground truth)

Evidence first: search git history and any CI logs for later changes where a negative-test failure drove a fix (a catch). Then ask the user exactly two questions:

1. Since this fence landed, has any negative test failed on a later change? (catches)
2. Have any bugs been found in the fenced features since — and which red-team category does each map to? (escapes)

Never infer, pad, or guess these numbers. No answer = "unknown", reported as unknown.
Every confirmed escape produces a mandatory recommendation: append a new invariant + test for it via /negatives.

## Step 4 — Teeth probe

Default (cheap, dependency-free): the sabotage probe. Run it ONLY when the worktree is clean (or after copying the target file to a temp backup):

1. Temporarily break ONE security-critical line in the fenced module — drop a tenant filter from a query, widen a field whitelist, remove an auth check
2. Run the negative suite: it MUST go red. Record which tests fired
3. Revert immediately and re-run to confirm green before doing anything else

A sabotage the suite does not catch is a HIGH finding: the fence has a hole at exactly the invariant that mattered.

Thorough option (Python, with the user's consent to install/run): `mutmut run` scoped to the fenced modules, time-boxed. Surviving mutants on tenant-filter, whitelist, or auth lines are HIGH findings.

## Step 5 — Scorecard and verdict

Output a table — `Metric | Value | Signal (good/bad/unknown)` — covering: runs logged, checkpoint edits, red-check integrity, negative test count, append-only compliance, teeth probe result, catches, escapes, rough cost per run.

Verdict per fenced feature:
- **STRONG FENCE** — teeth proven (probe caught), no escapes
- **WEAK FENCE** — a sabotage survived, or a confirmed escape exists
- **UNVERIFIED** — no git history, no CI signal, and no user answers to Step 3

End with the single highest-value recommendation, then append one line to the ledger:

`{"type":"retro","ts":"<ISO date>","project":"<repo dir>","catches":<n or "unknown">,"escapes":<n or "unknown">,"teeth":"caught|survived|skipped","appendonly":"clean|violations|unverifiable"}`

## Hard rules

- Never modify, weaken, or delete a test — even one you judge broken. Report it; changes go through /negatives.
- Sabotage probes require a clean worktree or a temp backup, an immediate revert, and a green re-run confirmed before the retro continues.
- Never fabricate catches or escapes; "unknown" is a valid and honest cell in the scorecard.
