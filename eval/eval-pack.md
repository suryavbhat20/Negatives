# Eval Pack — `/negatives` skill A/B test

Goal: measure how many real failure modes Claude covers **without** the skill vs **with** it, on the identical task, judged against an answer key written before either run.

---

## Rules (don't skip these)

1. **Never say "edge cases", "negatives", or "what could go wrong" in the prompt.** The whole point is measuring *unprompted* coverage. The prompt below is deliberately plain, like a real ticket.
2. **Two fresh sessions, clean git state.** Run B must not see Run A's code. Use a scratch branch; `git reset --hard` between runs.
3. **Answer key stays hidden** until both runs finish. Don't paste it into either session.
4. Same model, same effort setting, both runs.

---

## Protocol

1. **Run A (control)** — skill NOT installed yet (or set it `"off"` via the `/skills` menu). Fresh session in a scratch folder/branch. Paste the prompt below. Let it finish completely.
2. Save the output (code + tests), note rough duration. `git reset --hard`.
3. **Run B (treatment)** — install `SKILL.md` + `red-teamer.md`. Fresh session. Paste the *identical* prompt. Approve the Phase 2 invariant table when it stops (approve as-is, don't add hints).
4. Score both against the answer key.
5. Optional: if scores are close, run each once more — single runs have variance.

---

## The prompt (paste verbatim in both runs)

```
Implement a FastAPI endpoint: POST /tasks/{task_id}/link-email
Request body: {"email_id": "<id>"}

It should fetch the email from the MongoDB "emails" collection,
copy subject, sender, and receivedAt into the task document as a
_sourceEmail sub-document, and return the updated task.
Async with Motor. Write tests with pytest. Assume multi-tenant data
(documents have a tenant_id field) and JWT auth middleware already
exists and puts user info on request.state.
```

---

## ANSWER KEY — 15 golden negatives (do not reveal to either session)

| # | Category | Negative case |
|---|----------|---------------|
| 1 | Tenancy **CRITICAL** | `email_id` belongs to another tenant → gets linked anyway (cross-tenant leak into task) |
| 2 | Tenancy **CRITICAL** | `task_id` belongs to another tenant → foreign task modified |
| 3 | Input | Malformed ObjectId (email_id or task_id) → must be 400/422, not a 500 crash |
| 4 | Input | Valid-format `email_id` that doesn't exist → 404 |
| 5 | Input | Task doesn't exist → 404 |
| 6 | State | Task **already has** a `_sourceEmail` → behavior defined + tested (409 or explicit overwrite policy) |
| 7 | Concurrency | Same request sent twice concurrently (double-click) → idempotent, no corruption |
| 8 | Race | Email deleted between the read and the task update → no stale/orphan link written |
| 9 | Integrity | Email doc missing `subject`/`sender`/`receivedAt` (schema drift) → no KeyError 500, defined defaults |
| 10 | Integrity | `receivedAt` as string vs datetime / timezone-aware handling |
| 11 | Limits | Subject is 50KB → capped/truncated before denormalizing (document bloat) |
| 12 | Input | Unicode/emoji/RTL in sender name survives the round trip |
| 13 | Security | Response contains ONLY the 3 fields — never the full email body (PII minimization) |
| 14 | External | Mongo write fails after a successful read (partial failure) → clean 5xx, no half-written state |
| 15 | Security | Read-only role can't link (403) — authz beyond just authn |

---

## Scoring

Per item: **2 pts** = covered by a test that would actually fail without the handling · **1 pt** = handled in code but untested · **0 pts** = absent. Max 30.

**Hard gate:** missing #1 or #2 (tenancy) = automatic FAIL for that run, whatever the total. Those are the "one bad day" items.

Also record per run:
- Total score /30 and which categories were blind spots
- Duration + rough token/cost overhead of Run B (the skill isn't free — you're buying coverage with tokens)
- **False confidence check:** did Run A claim it "handles edge cases" while missing tenancy?
- **Test honesty check:** pick 2 negative tests from Run B, comment out the handling code, confirm the test actually goes red

## Skill-behavior checklist (Run B only)

- [ ] Skill triggered (auto or via `/negatives`)
- [ ] Delegated Phase 1 to `red-teamer` (fresh-context pass)
- [ ] STOPPED at Phase 2 and waited for approval — didn't barrel into code
- [ ] Tests written and confirmed failing BEFORE implementation
- [ ] Ran `pytest -m negative` in the implement loop
- [ ] Invariant table included a tenancy invariant without being told

## Verdict guide

- Run B ≥ 2× Run A's score AND catches both tenancy items → ship it to the repo
- Run B barely better → the skill body needs sharpening (usually the description or Phase 1 categories)
- Run B better but painfully slow/expensive → keep it manual-only: add `disable-model-invocation: true` so it runs only when you type `/negatives`

## Optional round 2 (tests category 7 — LLM-specific)

Prompt: "Implement an endpoint that takes a natural-language question, uses an LLM to generate a MongoDB filter for the emails collection, executes it, and returns matching docs."
Golden negatives: LLM output missing `tenant_id` filter, `$where`/operator injection via the generated filter, prompt injection through the user question, malformed JSON from the LLM, unbounded result set.

---

## Results (first run)

Same model and effort (Claude Fable 5, max) for both runs. Both passed the tenancy gate.

| # | Golden negative | Run A (no skill) | Run B (skill) |
|---|---|---|---|
| 1 | Foreign email invisible | 2 | 2 |
| 2 | Foreign task not writable | 2 | 2 |
| 3 | Malformed ObjectIds → clean 4xx | 2 | 2 |
| 4 | Email not found | 2 | 2 |
| 5 | Task not found | 2 | 2 |
| 6 | Re-link behavior | 2 | 1 |
| 7 | Concurrent double-click | 0 | 0 |
| 8 | Read-then-write race | 0 | 0 |
| 9 | Schema drift → null | 0 | 2 |
| 10 | `receivedAt` timezone | 1 | 1 |
| 11 | 50 KB subject cap | 0 | 0 |
| 12 | Unicode round-trip | 0 | 0 |
| 13 | PII whitelist | 2 | 2 |
| 14 | DB failure → clean 503 | 0 | 2 |
| 15 | Authz beyond authn | 1 | 1 |
| | **Total** | **16/30** (11 tests) | **19/30** (36 tests) |

What the score hides:

- Run B found two attack classes outside the key: MongoDB operator injection (`{"email_id": {"$ne": null}}`) and mass assignment (`tenant_id`, `_id` or `_sourceEmail` smuggled through the body).
- Run B tested each case more deeply: a pinned error contract, "no write happened" assertions on every rejected request, no-leak checks on 503s, and Hypothesis property tests.
- Run B found the race conditions (#7, #8) but silently deferred them. The fix: the checkpoint now lists every deferred item, and only the user can defer write-path race and idempotency invariants.
