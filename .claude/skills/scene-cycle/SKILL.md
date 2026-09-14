---
name: scene-cycle
description: Run one bounded author -> build -> evaluate reconstruction cycle for a room, using the scene-author, blender-builder, and scene-evaluator subagents. Use when asked to reconstruct a room, produce or revise a candidate, or evaluate an existing candidate against the reference evidence.
---

# Scene cycle

The bounded reconstruction loop from README milestones 4 and 5. Read AGENTS.md and
`.claude/rules/delegation.md` first; they own the rules this procedure executes.

## Preconditions

Stop and say which is missing if any of these does not exist yet. Do not improvise a substitute.

- A versioned scene contract, and the findings contract for evaluation (milestone 2).
- A builder that constructs and exports the synthetic fixture without a model call (milestone 3).
- Acceptance criteria and numerical tolerances, written down **before** this run starts.
- A reference input set with stable image IDs, recorded measurements, and known coverage gaps
  (milestone 1).

## Before starting

Agree and record, in the run directory:

- the budget — initial candidate plus at most three revisions by default;
- the render limit;
- which reference view is **reserved** and must not be used to tune the candidate.

## The cycle

1. **Author.** Delegate to `scene-author`: produce or revise scene data for revision N. Give it
   the reference image IDs, measurements, the previous revision and the evaluator findings it must
   resolve, and its allowed write paths. It returns the revision, evidence, and unresolved issues.

2. **Validate.** Validate the scene data against the contract in the uv environment before any
   Blender call. A contract violation goes back to the author; it is not the builder's problem.

3. **Build.** Delegate to `blender-builder`: construct revision N, run geometry checks, produce
   camera-matched previews for the reference views, export GLB. It returns Blender version, seeds,
   asset versions, check results, and output paths.

4. **Freeze.** Snapshot the candidate — scene data, geometry-check output, renders, GLB — under an
   immutable revision directory. Nothing modifies it after this point.

5. **Evaluate.** Delegate to `scene-evaluator` with the frozen path, the reference images, and the
   acceptance criteria verbatim. It reads the reference images itself. It returns structured
   findings and an acceptance recommendation. It changes nothing.

6. **Decide.** The main agent resolves the outcome against the evidence:
   - criteria met → check against the **reserved** view, then report `accepted`;
   - actionable findings and budget remains → route each finding to its owner (scene errors to the
     author, construction defects to the builder) and start revision N+1;
   - a finding needs a view or measurement that does not exist → `needs_evidence`;
   - budget exhausted → `budget_exhausted`.

   Retain the best candidate, not merely the last one.

## Rules that do not bend

- A failed tool, render, or check is a failure. It is never counted as a pass.
- Camera mismatch is ruled out before any discrepancy is attributed to geometry.
- Passing means the declared checks passed within observed coverage. It says nothing about the
  geometry of unseen areas, and the report must say so.
- Report `accepted`, `needs_evidence`, and `budget_exhausted` as distinct outcomes, with the
  unresolved issues attached.
- The main agent updates the README checklist, and only once deliverables and checks exist.
