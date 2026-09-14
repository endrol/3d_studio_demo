# 3D Studio Demo — Claude Code setup

Project conventions live in [AGENTS.md](AGENTS.md). Read it first; it is authoritative for scope,
delegation, scene/evaluation invariants, and working conventions, and it applies to Claude sessions
unchanged. This file only records what is specific to running the project under Claude Code.

Current status and the milestone checklist live in [README.md](README.md). Implement the milestone
the user requests. A checked-in roadmap is not authorization to run every step, install
dependencies, launch renders, or publish anything.

## Claude-specific rules

Read the file when the work touches it.

| When the task involves | Read |
| --- | --- |
| running Blender, headless renders, `bpy`, GLB export | `.claude/rules/blender.md` |
| splitting work across author / builder / evaluator roles | `.claude/rules/delegation.md` |

## Role mapping from the Codex setup

`.codex/agents/*.toml` pin OpenAI models and do not load in Claude Code. `.claude/agents/*.md` are
the equivalent roles with the same responsibilities, boundaries, and return contracts:

| Codex role | Claude subagent | Model / effort |
| --- | --- | --- |
| `scene_author` (`gpt-6-astra`, high) | `scene-author` | `opus`, `high` |
| `blender_builder` (`gpt-5.6-terra`, medium) | `blender-builder` | `sonnet`, `medium` |
| `scene_evaluator` (`gpt-6-astra`, high, read-only) | `scene-evaluator` | `opus`, `high`, no write tools |

The allocation intent carries over: stronger reasoning for spatial interpretation and evaluation, a
cheaper model for constrained construction work. It has not been benchmarked on a room. Keep both
configurations in sync when a role's responsibilities change; if they disagree, AGENTS.md wins and
the divergence is reported rather than silently resolved.

## Verified environment facts

Checked on 2026-09-12 on the development Mac:

- Blender 5.0.1 at `/Applications/Blender.app/Contents/MacOS/Blender`, headless render confirmed
  working (EEVEE, 640x480, ~1.8 s).
- Reference photos under `sample_data/` are readable as images directly, so render-vs-photo
  comparison needs no external vision API.
- `uv sync` and `uv.lock` are still incomplete; see the README setup note before assuming the
  environment is fully provisioned.
