# negatives

**Negative-first development for Claude Code.**

## The idea

AI writes code, but it doesn't make code reliable. The compiler, the type checker and the test runner do that. They are AI's best friend: they never guess, and they say exactly what is wrong.

So the real loop isn't "English in, code out". It's a probabilistic generator working against a deterministic checker:

```
generate → check → regenerate until it passes
```

The checker only catches what you've written down. AI rarely gets the happy path wrong. What it misses are errors of omission: the cross-tenant read, the malformed id, the duplicate request. It misses them because nothing put them in context.

This skill writes those cases down first, as failing tests, so the checker can hold the AI to them:

1. **Spec the darkness.** A read-only red-team subagent attacks the feature before any code exists.
2. **You choose.** The worst failure modes become invariants. You approve them, which takes two minutes instead of reviewing 400 lines of code.
3. **Make them executable.** Each invariant becomes a failing test.
4. **Code inside the fence.** Claude implements until the tests pass. The tests are append-only and are never weakened to get to green.

> Vanilla Claude gives you good tests. This skill gives you an adversary.

## Does it work?

A/B-tested on the same plain ticket against a hidden 15-item answer key: **16/30 without the skill, 19/30 with it.** The skill also found two attack classes that weren't in the answer key, MongoDB operator injection and mass assignment. Details: [eval/eval-pack.md](eval/eval-pack.md).

## Install

```bash
git clone https://github.com/suryavbhat20/Negatives.git
cp -r Negatives/.claude/skills/* ~/.claude/skills/
cp Negatives/.claude/agents/red-teamer.md ~/.claude/agents/
```

Then, in a new Claude Code session:

```
/negatives <feature spec or ticket>
/negatives-retro        # is the fence actually working?
```
