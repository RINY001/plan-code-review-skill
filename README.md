# plan-code-review

A Claude Code skill that runs substantial build work as **PLAN → EXECUTE → REVIEW** across model tiers —
keeping the *intelligence* on the strong model (the plan) while handing the *bulk* (writing the code) and
the *review* to Sonnet. On a Claude Code **subscription** this stretches your quota.

## Why

On a subscription you don't pay per token — you have a weekly/session **usage limit**. Strong models
(Opus, Fable) burn that limit far faster than Sonnet. But the intelligence only really matters in one
place: the **plan**. So:

- **PLAN** on your strong model (high effort) — the whole build follows it.
- **EXECUTE** on Sonnet (low effort) — token-heavy, mechanical once the spec is tight.
- **REVIEW** on Sonnet (high effort) — measured to match a strong-model review at ~⅕ the quota.

## What it actually does (measured, honest)

Across 5 controlled A/B tests (small algorithm tasks up to a real-codebase integration):

- **Quota:** ~**45–50% less** than running the same plan→build→review loop entirely on the strong model.
- **Code quality:** usually **leaner** code for the same behaviour (e.g. 44 vs 92 lines on a real task).
- **Correctness:** **equal** — the routed output passed every held-out test the strong-model baseline did.

**It is NOT a correctness upgrade.** On tasks a strong model can one-shot correctly, a naive one-shot is
cheaper still — this pays off when you *deliberately* run a plan→build→review loop on substantial work, and
it's overkill on small/exploratory tasks (the skill skips those automatically).

## The flow

```
PLAN   [strong model: Opus / Fable · high effort]  ->  a spec written to disk
       - resolve every open question yourself (no round-trips)
       - ground every signature in file:line
       - success = a red-capable test (goes RED if wrong)
                    |  context fork: the subagent sees ONLY the spec
EXECUTE [Sonnet · low effort]  ->  writes the code, test-first (red -> green)
                    |
REVIEW  [Sonnet · high effort] ->  simplify -> correctness (verify real APIs)
                    |
DONE = red-capable test green + review clean
```

## Install

**As a plugin (recommended):**
```
/plugin marketplace add RINY001/plan-code-review-skill
/plugin install plan-code-review@plan-code-review
/reload-plugins
```

**Or just copy the skill folder:**
```
git clone https://github.com/RINY001/plan-code-review-skill
cp -r plan-code-review-skill/skills/plan-code-review ~/.claude/skills/
```

## Use

It triggers automatically on substantial build/refactor/migration work. You control the strong model with
`/model` (Opus default; the skill flags when Fable is worth it). Small tasks, chat and exploration bypass
it — as they should.

## Scope & honesty

- All-Claude — never delegates to any external API.
- Sonnet is the execution/review floor (no Haiku).
- The ~45–50% saving is vs the *same loop on the strong model*, not vs a one-shot. It won't halve your
  total usage — it makes the heavy build + review cheap while the plan stays smart.

MIT licensed.
