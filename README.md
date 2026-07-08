# model-router

A Claude Code skill that picks the right orchestration pattern for a task — and runs the expensive model to *think* once, cheap models to *execute* everything else.

Most people reach for the most powerful model on every task. That's the mistake. **More AI is not better AI.** An agent burns ~4x the tokens of a plain chat; multi-agent burns ~15x (Anthropic's own number). Orchestration is a cost/benefit gate, not a default.

This skill is that gate.

## The three plays

1. **The expensive model thinks — once.** The smartest model writes the full plan: move by move, what to do, what will fail, and what to do when it fails. You pay for that thinking one time.
2. **Cheap models execute.** They don't improvise and they don't decide. They run the contract that's already written.
3. **You verify against reality — the step nobody does.** AI agents say "done" whether or not they did anything. Check the actual artifact (does the file exist? do the tests pass?), never the transcript.

Like an architect: they don't lay bricks, but without the blueprint the bricklayers build the wrong thing.

## The four patterns

`model-router` runs a selector first and declares one line — `Pattern: PX — reason` — before doing anything:

| Pattern | When |
|---|---|
| **P0 — Inline** (default) | 1–2 steps, few files, or context is already loaded. No agents. |
| **P1 — Brief + mechanical executors** | N independent pieces, spec fully resolved, zero new judgment per piece. |
| **P2 — War-game + reasoning executors** | Code with business logic, multi-step, sequential dependencies. |
| **P3 — Research fan-out** | Breadth-first research: independent sub-questions, each investigable alone. |

If in doubt → P0. The most common mistake is orchestrating something a direct chat would have done cheaper.

## Proven in production

The P2 pattern ran three sprints of a voice assistant (JARVIS) end to end — three times, all green. The expensive model war-gamed each move; Sonnet executed move by move; every step was verified on disk before the next one launched.

## Install

Copy the skill into your Claude Code skills directory:

```bash
git clone https://github.com/benagesjustin-boop/model-router.git
cp -r model-router/.claude/skills/model-router ~/.claude/skills/
```

Then invoke it in Claude Code:

```
/model-router
```

Or trigger it in natural language: *"research expensive, build cheap"*, *"which pattern for this?"*, *"run the war-game"*, *"orchestrate this with subagents"*.

## The token rules (apply to every pattern)

- **Effort ≤ high, always.** xhigh/max give the same output and burn double.
- **Explicit model on every agent call** (`sonnet` or `haiku`) — never inherit the expensive one by default.
- **Verify on artifacts, never on the transcript.** Agents generate "completion language" regardless of real state.
- **Retry classified, not blind:** haiku fails → one retry on sonnet with the failure pasted in → still fails → stop. Never escalate to the expensive model to *execute*.

---

Built in public with Claude Code. Part of an ongoing "systems over motivation" body of work.
