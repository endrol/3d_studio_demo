# Project instructions

## Scope and direction

Read README.md for current status and the milestone checklist. This repository has configuration, planning, and a minimal uv Python scaffold; do not claim that planned components already exist.

The central artifact is an explicit, versioned scene description inferred from photos and measured dimensions. A deterministic Blender Python builder consumes it. A separate evaluator compares geometry and rendered views with the original evidence; revisions update scene data. A Three.js viewer presents the exported GLB.

Implement the milestone requested by the user. A checked-in roadmap is not authorization to execute every step, install dependencies, launch paid runs, or publish a site. Continue necessary work within the requested milestone without repeated permission questions.

## Delegation

- Use subagents for substantial independent investigations or implementations, and use a separate evaluator for reconstruction review. Keep small edits local.
- The main agent coordinates scope, shared contracts, file ownership, integration, and final acceptance. It owns README checklist updates.
- Use `scene_author`, `blender_builder`, and `scene_evaluator` when available; their role instructions live in `.codex/agents/`. If the client cannot select custom roles, read the relevant file and pass its instructions, model, and reasoning effort explicitly to a standard subagent. Report this fallback. If the configured model is unavailable, report it rather than silently substituting another model.
- Assign each task a concrete objective, input files/revision, output contract, acceptance checks, and allowed write paths. Do not let two agents edit the same files concurrently.
- Parallelize independent work. Run author → builder → evaluator in dependency order for each candidate; evaluate immutable candidate snapshots.
- Subagents return concise findings, changed paths, checks performed, and unresolved issues. They do not spawn further agents unless the main agent explicitly delegates that authority.
- The project cap is three concurrent child threads, in addition to the main agent; respect any lower host limit. Reuse completed agents when useful.
- Use the model and reasoning settings pinned in each role file: Astra/high for scene author and evaluator, Terra/medium for Blender builder. Other subagents inherit session/user defaults unless the user requests a change. Agent configuration is development assistance; an automated runtime pipeline must be implemented separately.

## Scene and evaluation invariants

- Use stable IDs and explicit units, axes, origin, and rotation/transform conventions. Version changes to the scene contract and update all consumers together.
- Preserve provenance: distinguish measured, inferred, and unknown values. Reference images and renders by stable IDs/paths; never present model confidence as calibrated probability without evidence.
- Scene data owns room-specific layout. The builder implements supported geometry and asset placement without interpreting photographs or silently inventing missing fields.
- Pin or record asset versions, builder/Blender versions, and random seeds for reproducibility. Keep critical geometry editable; preserve semantic group identity through export.
- Define acceptance criteria before the run. Evaluate geometry and visual resemblance separately. Match camera views before attributing a discrepancy to geometry.
- Evaluators inspect the original evidence and the exact candidate. They report actionable issues with object IDs and evidence; they do not edit candidates, relax criteria, or accept missing checks.
- The author handles scene corrections; the builder role handles construction defects. The main agent resolves conflicting findings.
- Bound revision cycles and cost/render budgets, save revision history, and retain the best candidate. Default pilot plan: initial candidate plus at most three revisions; automated spending/render limits must be set before execution.
- Report accepted, needs-evidence, and budget-exhausted outcomes distinctly. Tool failures are failures, not quality passes. Unseen geometry remains uncertain even when all observable checks pass.

## Working conventions

- Prefer Python/Pydantic for the scene contract and orchestration, Blender's Python API for construction, and Three.js for presentation. Add these only as their milestones require.
- Use the existing uv project for Python dependencies: `uv add` for runtime packages, `uv add --dev` for development tools, `uv sync` for setup, and `uv run` for project commands. Keep dependencies in pyproject.toml and include uv.lock in version control once generated; keep .venv ignored. Preserve the Python 3.12 pin unless a concrete requirement changes it.
- Blender scripts run in Blender's bundled Python, with `bpy`; the uv environment owns orchestration, schema validation, and tests. Pass validated scene data across the process boundary instead of assuming uv-installed packages are importable inside Blender.
- Keep private photos, generated renders, model binaries, and run outputs out of git. Use small synthetic fixtures for checked-in validation.
- Validate external scene input at the boundary. Test meaningful contract and geometry behavior; do not add tests just to mirror implementation.
- Update existing documentation and mark README tasks complete only after their deliverables and checks exist. Mention skipped checks and unresolved limitations.
- Do not commit or push unless explicitly requested. Preserve unrelated work and retain existing permission restrictions.
