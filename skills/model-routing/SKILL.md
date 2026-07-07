---
name: model-routing
description: "Run substantial build/change work as PLAN (strong model) → DELEGATE execution (Sonnet) → REVIEW (Sonnet) to keep the intelligence where it matters (the plan) while cutting quota on a Claude Code SUBSCRIPTION. Use BY DEFAULT when implementing, refactoring, migrating, or transforming something bulky enough to spec and hand off — a multi-file feature, a repetitive edit across many sites, a big migration, a long document. Skip for small, quick, or exploratory tasks (handoff overhead outweighs the win). All-Claude only — never delegate to OpenAI/Codex/any external API."
---

# Model routing — smart plan, cheap build, lower quota

On a Claude Code **subscription** you don't pay per token — you have a **quota / rate-limit**. Strong
models (Opus, Fable) burn it far faster than Sonnet. The intelligence only truly matters in **one** place:
the **plan**. So keep the strong model for PLAN, and send BOTH the token-heavy build AND the review to
**Sonnet**. Measured result: ~**45-50% less quota** than running the same plan→build→review loop entirely
on the strong model, with equal correctness and (usually) cleaner code.

**Model + effort per phase:**

| Phase | Model | Effort | Why |
|---|---|---|---|
| **PLAN / spec** | strong tier (Opus; Fable if hard) | **high** | This is where intelligence pays — the whole build follows the plan |
| **DELEGATE / execute** | **Sonnet** | **low** | Token-heavy; mechanical once the spec is tight — the big quota saving |
| **REVIEW** | **Sonnet** | **high** | Validated to match a strong-model review at ~5× less quota (give the weaker reviewer the reasoning budget) |

## Who picks which model

- **The strong tier for PLAN (Opus vs Fable) is the USER's choice**, set via `/model` or fast mode. You
  cannot switch the main-loop model. Opus is the default; Fable is an *escalation* the user opts into for a
  genuinely frontier-hard plan. If a plan looks that hard (or an Opus plan reads weak), **pause and flag**
  "this is hard enough that Fable would help — want to switch?" — never assume Fable.
- **You pick the SUBAGENT models: Sonnet for both execute and review.** Never Opus/Fable in a subagent
  (defeats the purpose), never Haiku (its only fit is trivial work, which isn't routed at all).
- **Only PLAN runs on the strong tier.** Sonnet-high review was measured to match a strong-model review
  (same correctness, comparable simplification) at roughly a fifth of the quota — so review does NOT need
  the strong model.

## When to route (check FIRST)

Route only when BOTH hold: (1) the execution is **bulky** (many files / lots of output / repetitive
transform / long draft), and (2) you can hand it off with a **self-contained spec**. Otherwise just do it
yourself — for a small or exploratory task the spec-and-review overhead eats the saving. Most of an
interactive session (chat, exploration, small fixes) never crosses this line, and shouldn't.

## PLAN — write a spec that is a handoff document

The PLAN→DELEGATE boundary is a **context fork, not a step**: the subagent opens a FRESH window and sees
ONLY the spec. So the spec must stand alone. Produce it on the strong tier, high effort:

1. **Resolve every open question yourself** with a recommended answer — hand the subagent decisions, not
   things to figure out. A spec full of "TBD" forces round-trips (which cost more quota than they save).
2. **Scope it to ONE vertical slice** — a complete path through all the layers it touches, independently
   verifiable — not a broad horizontal feature. Narrow enough to fit a cheap subagent's context.
3. **Ground every claim in `file:line`** (or a table row / exact path / exact signature). If you can't
   ground it, the subagent must not act on it. On a real codebase, put the EXACT existing signatures the
   executor needs into the spec, so it never re-reads the codebase or guesses.
4. **State the success criterion as a red-capable command** — one runnable test / curl / script that goes
   RED if the implementation is wrong. This is the spec's definition of done.
5. **Include, briefly:** one sentence of *why* (the goal, so the subagent optimizes the right thing); the
   named test seam; and which tools/skills the subagent should use.
6. **Prune before handing off:** only keep a new seam if the execution needs **two** real adapters (one
   adapter = hypothetical, drop it); only spell out an architectural decision if it's hard-to-reverse AND
   surprising-without-context AND a real trade-off (else it's just implementation, cut it).

**PLAN done when:** a fresh agent with only this spec file could produce the right output. Write the spec
to disk (e.g. `spec.md` in the work dir) so the fork is clean.

## DELEGATE — Sonnet, low effort

Spawn the executor via the Task/Agent tool pinned to Sonnet, pointing at the spec:
`Agent(prompt=<spec or its path>, model="sonnet", effort="low")`. It should NOT re-read the codebase — the
spec has every signature it needs. For independent slices, spawn several in ONE message. For a
deterministic multi-item fan-out, use the **Workflow** tool with per-agent `model="sonnet"`.

Tell the subagent to build **tracer-bullet style**: make the red-capable command fail first, then
implement until it goes green — one behavior at a time, not one big write. It returns the diff + the now-
green command.

## REVIEW — Sonnet, high effort

Run the review as a Sonnet subagent at **high** effort (the reasoning budget is what lets the cheaper model
match a strong review):

1. **Simplify** — reuse / clarity / naming / altitude cleanups (cleaner code first = fewer spurious
   correctness findings).
2. **Correctness** — hunt hard for bugs vs the spec; on a real codebase, verify every real API call against
   the actual source. Try to construct an adversarial input the implementation gets wrong.
3. **Keep the fix loop on Sonnet too** — feed findings back to a Sonnet subagent to fix, then re-review.
   Only escalate a fix to the strong tier if it needs judgement the cheaper model demonstrably lacks.

## Guardrails

- **Always review** — routing without review just ships a cheap model's mistakes faster.
- **Don't over-route** — a subagent for a two-line edit is net-negative. When unsure, do it yourself.
- **Self-contained spec or don't route** — if you can't spec it crisply, it isn't ready to hand off.
- **Strong model on PLAN only; Sonnet executes + reviews. No Haiku, no external APIs, ever.**

## Verification

The cycle ran correctly when: (1) the spec's **red-capable command is now green**, and (2) the review comes
back clean (or all findings fixed and re-reviewed). If either is missing, the loop isn't done.

## Honest scope

This is a quota + code-quality tool, not a correctness upgrade: on tasks a strong model can one-shot
correctly, the routed output is equally correct and usually leaner, at ~45-50% less quota than the same
loop on the strong model — but a naive strong-model one-shot is cheaper still. The win is real when you are
deliberately running a plan→build→review loop on substantial work; it is not magic on small tasks.
