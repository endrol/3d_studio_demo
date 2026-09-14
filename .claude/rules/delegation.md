# Delegation

AGENTS.md defines the delegation model. This file records how it is executed in Claude Code.

This file is the project's standing authorization to use subagents for the work described below.
It does not authorize the Workflow tool — see the last section.

## Two mechanisms

**Agent tool — the default.** Each call runs one subagent in its own context and returns a summary.
Independent calls in a single message run concurrently. Use it for the author / builder / evaluator
roles, and for any investigation whose intermediate output would flood the session.

**Workflow tool — only on explicit request.** Runs a deterministic JavaScript script that
orchestrates many subagents with schema-validated results, phases, and resumable caching. It fits a
bounded revision loop run unattended over several candidates. It can spawn a lot of agents, so it
is used only when the user asks for it in their own words. Until then, drive cycles with the Agent
tool and let the main agent hold the loop.

## Roles

`.claude/agents/` defines `scene-author`, `blender-builder`, and `scene-evaluator`. Address them by
name. Their instruction files carry the boundaries; the assignment carries the specifics.

The main agent owns scope, shared contracts, file ownership, integration, README checklist updates,
and final acceptance. No subagent marks a README task complete or declares its own work accepted.

## Assignment contract

Every task handed to a subagent states, explicitly:

- the objective, and the milestone it belongs to;
- input files and the exact candidate revision;
- the output contract — what to produce and in what shape;
- the acceptance checks that will be run against it;
- the allowed write paths.

Subagents return changed paths, checks performed, and unresolved issues. They do not spawn further
agents unless the main agent explicitly grants that.

## Concurrency and ordering

At most three child threads run at once, plus the main agent, matching `.codex/config.toml`.
Respect a lower host limit if one applies.

Within one candidate the order is author → builder → evaluator, strictly sequential; each stage
consumes the previous stage's output. Across candidates the pipelines are independent and run in
parallel. Never let two agents write the same paths concurrently.

The evaluator inspects an immutable snapshot. Freeze the candidate — scene data, build outputs,
renders — before evaluation starts, and do not repair it underneath a running evaluation. A fix
produces the next revision.

## Evidence and honesty

The evaluator reads the original reference images and the exact candidate, not the author's
summary. A tool or render failure is a failure, reported as such — never counted as a quality pass.
When two agents disagree, the main agent resolves it against the evidence rather than averaging
the findings or re-running until they agree.

Bound every cycle before starting: an initial candidate plus at most three revisions by default,
with render limits set in advance. Save revision history and retain the best candidate even when a
later revision regresses. Report the outcome as `accepted`, `needs_evidence`, or
`budget_exhausted`.
