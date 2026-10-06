---
name: negatives
description: Negative-first development — red-team a feature before implementing it, distill failure modes into invariants, write failing negative tests, then gate implementation on that suite. Whenever the user asks to implement, add, build, or create a new endpoint, feature, background job, or module — including a pasted requirement or Jira ticket (e.g. PROJ-123) — OFFER this skill with one yes/no question before writing any implementation code; never start it without the user's yes. Runs immediately when the user types /negatives or says "negatives", "red team this", "edge cases first", or "what could go wrong". Not for bug fixes or refactors.
argument-hint: [feature description or path to spec]
---

# Negative-First Development

Never write the happy path first. Work through these phases in order. Do not skip or merge phases.

## Feature under analysis

$ARGUMENTS

If no argument was given, use the feature currently being discussed in the conversation. If neither exists, ask the user for a one-paragraph spec and stop.

## Phase 0 — Permission gate

If the user did NOT explicitly invoke this skill (they typed `/negatives` or used one of its trigger phrases), ask exactly one yes/no question before anything else: *"This adds new behavior — run the negative-first workflow (red-team → invariants → failing tests) before implementing? Yes / No, just implement."* On anything other than a clear yes, stop and proceed with the user's original request normally. Never begin Phase 1 on your own initiative.

## Phase 1 — Red team (adversarial mode only)

Attack the spec. Do NOT think about implementation in this phase.

If a `red-teamer` subagent is available, delegate this phase to it, so the adversarial pass runs in a clean context with no anchoring on an implementation plan. Pass the full spec, the repo-grounding instruction, and the category list below verbatim in the delegation prompt — this list is the source of truth; the subagent definition's own compressed list is only a fallback for standalone use. Otherwise do it inline.

Before brainstorming, ground in the repo: Grep for similar existing handlers/services and their tests, and harvest failure patterns this codebase has already hit.

Generate 15–30 concrete failure modes covering ALL of these categories:

1. **Input & boundary** — empty/None/missing fields, wrong types, huge payloads, unicode/emoji, injection strings, off-by-one on limits and date ranges
2. **Security & tenancy** — cross-tenant data leakage, RBAC bypass, IDOR, unauthenticated paths, secrets or PII in logs
3. **State & concurrency** — duplicate/replayed requests, idempotency, race conditions, retry after partial success
4. **External dependencies** — timeout, rate-limit (429), 5xx, malformed response, slow response, DB failover mid-operation
5. **Data integrity** — schema drift, nulls where code assumes values, timezone/DST, encoding, stale cache vs source of truth
6. **Scale & limits** — pagination edges, N+1 queries, payload/token limits, memory on large batches
7. **LLM-specific** (when the feature touches an LLM) — malformed or hallucinated JSON output, prompt injection via user or document content, nondeterministic responses breaking downstream assumptions, context overflow

## Phase 2 — Distill invariants (human checkpoint)

Select the 5–10 highest-risk items and rewrite each as an **invariant**: a property that must ALWAYS hold, not a single example. ("Response never contains another tenant's documents" — not "tenant B's id returns 403".)

Present TWO lists at the checkpoint:

1. **Approved candidates** — a table: `ID | Invariant | Category | Why it matters`
2. **Deferred long tail** — every remaining Phase 1 finding, one line each, so the user sees exactly what is being parked

**Stop and wait for the user to approve, edit, or promote deferred items before writing any test or code.** Deferral is the user's decision, never the model's — nothing gets parked silently. Reviewing these lists is the human's main control point in the whole workflow.

## Phase 3 — Make them executable

For each approved invariant, write a failing test BEFORE any implementation exists. Pure wiring needed for tests to import and collect (app factory, conftest fixtures, fakes) may be created now; routes, middleware, handlers, and business logic may not — they ARE the feature.

- Location: `tests/negatives/test_<feature>.py`
- Copy the approved invariant table and the pinned error contract into the test file's module docstring — chat is ephemeral, and append-only enforcement needs the registry to live in the repo
- Mark every test `@pytest.mark.negative` and reference its invariant ID in the docstring; register the `negative` marker in pytest.ini (under `--strict-markers` an unregistered marker is exactly the collection error this phase forbids)
- Prefer property-based tests (Hypothesis) for input-shaped invariants; example-based tests for flow/state invariants. Build the client/db inside property tests rather than via function-scoped fixtures (Hypothesis health-check clash; cross-example state contamination)
- Mock external dependencies to force their failure modes (timeouts, 429s, garbage payloads)
- Pin the error contract: a test whose expected status equals the framework's route-missing default (e.g. FastAPI's 404 `{"detail": "Not Found"}`) passes vacuously while the feature is absent. Assert on distinct detail strings or error codes so "feature absent" and "resource not found" are distinguishable
- Run the suite and confirm every new test FAILS for the right reason (feature absent) — not from import or collection errors. A test that PASSES at this stage is a broken test: fix the test before proceeding
- Exception: if a test passes because existing infrastructure already enforces the invariant (common in brownfield), prove it CAN fail — temporarily sabotage the enforcement, observe red, revert — then keep it, labeled "already held" in its docstring
- In non-Python projects, keep every phase and rule but map these mechanics to the project's test stack: the `negative` marker becomes the stack's tag/filter idiom, `tests/negatives/` its equivalent location, Hypothesis its property-based equivalent (fast-check, PropEr, etc.)

## Phase 4 — Implement inside the fence

Only now write the feature. Loop: implement → `pytest -m negative` → fix, until green. Then run the full test suite.

## Phase 5 — Log the run

Append ONE line to `~/.claude/negatives-runs.jsonl` (Windows: `%USERPROFILE%\.claude\negatives-runs.jsonl`; create the file if missing) so `/negatives-retro` can evaluate this workflow across projects:

`{"type":"run","ts":"<ISO date>","project":"<repo dir name>","feature":"<one line>","failure_modes":<Phase 1 count>,"invariants_approved":<Phase 2 count>,"checkpoint_edits":<how many invariants the user edited/added/rejected/promoted at the checkpoint>,"red_check":"clean|dirty","tests_green":<final negative test count>,"duration_min":<rough wall clock>,"notes":"<unusual events, or empty>"}`

Report the numbers honestly: `checkpoint_edits: 0` means the user approved as-is; `red_check` is "clean" only if every new test failed for the right reason before implementation. Never skip logging because a run went badly — bad runs are the most valuable entries in the ledger.

## Hard rules

- NEVER delete, weaken, skip, or xfail a negative test to get to green. If a test looks genuinely wrong, stop and ask the user.
- Negative tests are append-only for the life of the project.
- Keep at most the ~10 approved invariants in working context; the deferred long tail lives in the test file as a documented backlog (comment block), converted to append-only tests when items are promoted.
- If the feature touches tenant-scoped data, a tenant-isolation invariant is mandatory even if the spec doesn't mention it.
- If the feature performs writes, invariants for idempotency, concurrent duplicate requests, and read-then-write races are mandatory candidates in the approved table — they may go to the long tail ONLY if the user explicitly rejects them at the checkpoint. Never auto-defer a race condition.