# negatives

**Negative-first development for Claude Code.** Before Claude writes a feature, an adversarial subagent attacks the spec. You approve the failure modes worth fencing, they become failing tests, and only then is the feature implemented, inside that fence.

> Vanilla Claude gives you good tests. This skill gives you an adversary.

## Why

Claude rarely gets the happy path wrong. What it misses are **errors of omission**: the cross-tenant read, the malformed id that becomes a 500, the duplicate request, the email with no subject. It misses them because nothing put them in context. Asking for "edge cases" in prose doesn't fix that, because prose is forgotten once code generation starts.

So the skill inverts the order:

1. **Spec the darkness first.** Enumerate how the feature breaks, leaks, or gets abused.
2. **Make it executable.** Every approved failure mode becomes a failing test before any feature code exists.
3. **Code inside the fence.** Implement until the negative suite is green. The suite is append-only from then on.

The human stays in the loop at the cheapest point. Reviewing ~8 invariants takes two minutes; reviewing 400 lines of generated code takes thirty.

## How it works

| Phase | What happens |
|---|---|
| 0. Permission gate | If Claude spots new behavior on its own, it asks one yes/no question before starting. Typing `/negatives` skips the question. |
| 1. Red team | The read-only `red-teamer` subagent runs in a clean context with no implementation plan to anchor on. It lists 15–30 concrete failure modes across 7 categories: input, security & tenancy, state & concurrency, external dependencies, data integrity, scale, and LLM-specific. |
| 2. Invariants (human checkpoint) | The top 5–10 become **invariants**: properties that must always hold ("a response never contains another tenant's documents"), not single examples. You see the approved table *and* every deferred finding, so nothing is parked silently. Claude stops until you approve. |
| 3. Failing tests | One test per invariant under `tests/negatives/`, marked `negative`. Input-shaped invariants get Hypothesis property tests. Every test must fail for the right reason before implementation starts. |
| 4. Implement | Implement, run `pytest -m negative`, fix, repeat until green. Then run the full suite. |
| 5. Log | Each run appends one JSON line to `~/.claude/negatives-runs.jsonl`, which `/negatives-retro` reads. |

Hard rules:

- Negative tests are never deleted, weakened, skipped, or xfailed to get to green.
- Tenant isolation is mandatory for tenant-scoped data, even when the spec doesn't mention it.
- For write features, idempotency and race-condition invariants can be deferred by the user, never by the model.

Non-Python projects keep every phase. The marker, test location, and property-testing library map onto the project's own stack.

## Install

Install the skill and the agent together. Phase 1 can run inline without `red-teamer`, but the clean-context pass is the point.

Global (every project on your machine), macOS/Linux:

```bash
git clone https://github.com/suryavbhat20/negatives.git
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r negatives/.claude/skills/* ~/.claude/skills/
cp negatives/.claude/agents/red-teamer.md ~/.claude/agents/
```

Windows (PowerShell):

```powershell
git clone https://github.com/suryavbhat20/negatives.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills", "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item -Recurse -Force negatives\.claude\skills\* "$env:USERPROFILE\.claude\skills\"
Copy-Item -Force negatives\.claude\agents\red-teamer.md "$env:USERPROFILE\.claude\agents\"
```

Per project (shared with your team through git): copy the same files into the repository's own `.claude/` folder.

Start a new Claude Code session after installing, then:

```
/negatives <feature spec, ticket text, or path to a spec>
/negatives-retro
```

## The evaluation

The skill was A/B-tested before it was trusted. Two fresh Claude Code sessions (Claude Fable 5, max effort) got the same plain ticket. The prompt never says "edge cases":

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

Both runs were scored against a 15-item answer key written beforehand and hidden from both sessions. A case scores 2 if a test covers it, 1 if the code handles it without a test, and 0 if it's absent. Missing either tenancy item fails the run outright. The full protocol and key are in [`eval/eval-pack.md`](eval/eval-pack.md).

| | Run A (no skill) | Run B (skill) |
|---|---|---|
| Score | 16 / 30 | 19 / 30 |
| Tests | 11 | 36 |
| Tenancy gate | pass | pass |

<details>
<summary>Per-item scores</summary>

| # | Golden negative | Run A | Run B |
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

</details>

### What the score hides

- **Two attack classes outside the answer key.** The skill found MongoDB operator injection (`{"email_id": {"$ne": null}}` matches an arbitrary email, possibly another tenant's) and mass assignment (smuggling `tenant_id`, `_id`, or `_sourceEmail` through the request body). Neither the vanilla run nor the answer key had them.
- **Depth per item.** Run B pinned an error contract, asserted that no write happened on every rejected request, checked that 503 bodies leak nothing, and used Hypothesis property tests over all malformed inputs instead of a few examples.
- **Cost.** The red-team subagent used about 15K tokens. The run went from 27 failure modes to 9 approved invariants to 36 test cases, and all 36 were confirmed failing before the feature existed.

### What the eval exposed, and the fixes

- **v1.0 → v1.1.** Run B *found* the race conditions (#7, #8) but silently parked them to stay under the ~10-invariant limit. Now the checkpoint shows the deferred list too, and only the user can defer write-path idempotency and race invariants.
- **v1.1 → v1.2.** A phase-by-phase review of the same run found six process gaps. The biggest: a negative test expecting a 404 passes *before the feature exists*, because the framework's route-missing default is also a 404. Phase 3 now requires a pinned error contract, and a test that passes at the red check counts as broken. The other five fixes:
  - a scaffolding boundary: test wiring may exist before implementation, routes and middleware may not
  - an "already held" rule for brownfield code: prove the test can fail by sabotaging the enforcement
  - the invariant registry copied into the test file
  - `negative` marker registration
  - one source of truth for the red-team categories, plus a mapping for non-Python stacks
- **Still missed by both runs:** concurrent double-submits, the read-then-delete race, the 50 KB subject cap, and unicode round-trips. The skill doesn't catch everything. It moves the misses from silent omissions to items you can see and decide on at the checkpoint.

## Is it working in your projects? `/negatives-retro`

Every run logs one line to `~/.claude/negatives-runs.jsonl`. `/negatives-retro` turns that ledger into a scorecard:

- **Ledger trends**, including whether the checkpoint is being rubber-stamped.
- **Append-only audit** of git history: deleted tests, new skip/xfail marks, loosened assertions.
- **Sabotage probe**: break one tenant filter, confirm the suite goes red, then revert.
- **Ground truth only you know**: regression catches and escaped bugs, by category.
- **A verdict per feature**: STRONG FENCE, WEAK FENCE, or UNVERIFIED.

## Repository layout

```
.claude/
├── skills/negatives/SKILL.md         the workflow
├── skills/negatives-retro/SKILL.md   the scorecard
└── agents/red-teamer.md              read-only adversarial subagent
eval/eval-pack.md                     A/B protocol and the hidden answer key
```
