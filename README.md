# 3D Studio Demo

Reconstruct an editable 3D space from photos, using an explicit scene description and an iterative evaluation loop.

The core experiment is whether a vision model can translate photographs into useful spatial data: objects, dimensions, relationships, materials, and uncertainty. A deterministic Blender Python builder should be able to reconstruct the space from that data without interpreting the photographs itself.

**Status:** configuration and a minimal uv Python scaffold, plus a local four-photo benchmark in the git-ignored `output/` folder. The benchmark has versioned scene JSON, a stdlib validator, a Blender builder script, and a fallback GLB exporter. Native Blender execution is blocked by a startup crash. No model API integration, automated evaluator, or browser viewer has been implemented.

## Local four-photo benchmark

`output/scene.json` describes 393 editable primitives inferred from four overlapping living/kitchen photos in `sample_data/input/normal/`: `16349619_居室.jpg`, `16349620_居室-16263970.jpg`, `16349621_居室-16263976.jpg`, and `16349591_キッチン-16263997.jpg`. All dimensions are estimates; no measurements or held-out view were supplied. References and hashes are recorded in `output/reference_manifest.json`.

`output/benchmark.glb` is a Blender-importable asset with semantic object/group IDs and 11 embedded wood textures. `output/cutaway.svg` is a geometry diagram, not a rendered fidelity check. The fallback exporter passed structural checks, primitive bounds/transform checks, malformed-input rejection, and an identical repeated export. It omits optional bevel modifiers. Reports and the retained initial candidate are under `output/validation.json`, `output/evaluation.json`, and `output/revisions/r00/`.

Native `.blend` generation and photo-matched previews are incomplete: Blender 5.0.1 crashes during Metal initialization before executing Python, including an approved retry. Specialized agents also reached their usage limit before final independent evaluation. The result is `needs_evidence`, with a separate native-tool failure; it is not an accepted reconstruction. The retained builder can be run in a working Blender environment:

```bash
/Applications/Blender.app/Contents/MacOS/Blender --background --python-exit-code 1 --python output/build_scene.py -- output/scene.json output/native
```

The existing `.venv/bin/python` was used for this dependency-free fallback because `uv sync` still hits the previously recorded macOS crash. All benchmark artifacts and scripts are local and ignored by git; this does not establish the reusable schema/builder milestones below.

## Python setup

Use the existing uv project for all Python packages. `.python-version` selects Python 3.12; `pyproject.toml` declares project metadata and dependencies. Add packages when their implementation milestone needs them.

```bash
uv sync
uv run main.py
```

`main.py` is currently the generated greeting, so this is only an environment smoke check. Use `uv add <package>` for runtime dependencies and `uv add --dev <package>` for development tools. Keep the generated `uv.lock` in version control; `.venv/` is already ignored. No third-party dependencies have been added yet.

The uv environment will run scene validation, orchestration, and tests. Blender runs geometry scripts using its own bundled Python and `bpy`. Packages installed by uv are not automatically available inside Blender; the planned process boundary is validated scene JSON. Blender 5.0.1 was found at `/Applications/Blender.app/Contents/MacOS/Blender` on the initial development Mac.

Setup check on 2026-09-12: uv created `.venv` with Python 3.12.8, and `.venv/bin/python main.py` passed. Full sync and lockfile generation remain incomplete: installed uv 0.9.1 hit cache access restrictions, then a macOS SystemConfiguration panic with `--no-cache`, including on an approved retry. This matches a reported [uv sandbox issue](https://github.com/astral-sh/uv/issues/17484). Retry `uv sync` in a regular terminal; if the panic persists, update uv before retrying. No global tool upgrade or permission change was made for this setup.

## Intended workflow

```mermaid
flowchart TD
    A[Photos and measured dimensions] --> B[Scene author]
    B --> C[Versioned scene.json]
    C --> D[Deterministic Blender Python builder]
    D --> E[Geometry checks and preview renders]
    A --> F[Scene evaluator]
    E --> F
    F --> G{Acceptance criteria met?}
    G -->|Yes| H[Export GLB and explore in Three.js]
    G -->|Actionable issues within budget| B
    G -->|Missing evidence or budget exhausted| I[Save partial result and unresolved issues]
```

The scene description is the source of truth. Model-generated Blender code is possible, but our first experiment will separate spatial interpretation from geometry construction. Geometry or asset references must be sufficiently specific for the builder; descriptions such as “a nice chair” leave reconstruction decisions unresolved.

Codex subagents help us develop and exercise this workflow. Their configuration does **not** implement an autonomous application pipeline or start API jobs. A runtime orchestrator is a later milestone.

## Codex setup

- [AGENTS.md](AGENTS.md) defines project conventions, delegation, and reconstruction rules.
- [.codex/config.toml](.codex/config.toml) enables subagents and caps concurrent child threads at three, in addition to the main agent.
- [.codex/agents/scene_author.toml](.codex/agents/scene_author.toml) defines the spatial interpretation and revision role.
- [.codex/agents/blender_builder.toml](.codex/agents/blender_builder.toml) defines the deterministic construction role.
- [.codex/agents/scene_evaluator.toml](.codex/agents/scene_evaluator.toml) defines an independent evaluator that does not edit the candidate or acceptance criteria.

Each specialized role pins its model and reasoning effort:

| Role | Model | Reasoning | Purpose |
| --- | --- | --- | --- |
| Scene author | `gpt-6-astra` | `high` | Interpret ambiguous spatial evidence and revise the scene. |
| Blender builder | `gpt-5.6-terra` | `medium` | Implement and debug construction from an explicit contract. |
| Scene evaluator | `gpt-6-astra` | `high` | Compare reference views and renders, and reason about fidelity. |

This initial allocation prioritizes spatial interpretation and evaluation while using a less costly model for constrained builder work. It follows the official [model-selection guidance](https://developers.openai.com/tracks/building-agents#how-to-choose), checked on 2026-09-11; it has not yet been benchmarked on a room. Separate author/evaluator threads can still share model blind spots, so measured checks and reserved reference views remain necessary.

Edit `model` and `model_reasoning_effort` in the relevant role file to change its assignment. Role-file settings take precedence over spawn-time choices and user-level agent defaults. The main session model is not pinned by this repo. Running the completed Blender Python builder locally requires no model call.

Open a new Codex session in this repository to load the project setup. Project configuration loads only for a trusted project; use Codex's normal trust flow if prompted. Custom-role availability depends on the client. If a role is unavailable, the main agent can pass the corresponding instructions, model, and reasoning effort to a standard subagent and report that fallback. Account access to the chosen models still needs verification when spawning them; unavailable models must be reported rather than silently substituted.

Examples for future sessions:

> Work on milestone 2 only. Use scene_author to propose the contract and scene_evaluator to critique whether it is sufficient and testable. Return the unresolved decisions before implementing the builder.

> Implement milestone 3 from the agreed scene contract. Delegate the builder to blender_builder and independently check its output before updating the checklist.

Configuration follows the official [subagent documentation](https://learn.chatgpt.com/docs/agent-configuration/subagents) and [configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference), consulted on 2026-09-10. Installed CLI at setup: `codex-cli 0.153.4`.

## Step-by-step roadmap

Check a task only when its deliverable exists and its relevant checks pass. This roadmap records direction; it does not authorize running every milestone automatically.

### 0. Establish the workflow

- [x] Agree on explicit scene data, deterministic construction, and iterative evaluation.
- [x] Add project Codex configuration and specialized agent instructions.
- [x] Record the milestones and acceptance approach in this README.
- [x] Initialize a uv project with Python 3.12 and establish dependency-management conventions.
- [ ] Complete `uv sync`, generate `uv.lock`, and verify `uv run main.py`.

### 1. Choose one reference room

- [x] Select one room and collect several overlapping photos or walkthrough frames.
- [x] Record available measurements and assign stable IDs to reference images.
- [x] Identify unseen areas and reserve an independent reference view for the final check when coverage allows. (Unknowns recorded; no held-out view within the selected four.)
- [x] Choose local storage for private references and generated artifacts; keep them out of git.

Done when the input set, known dimensions, and missing evidence are explicit.

### 2. Define the scene and evaluation contracts

- [ ] Define a versioned Pydantic/JSON Schema scene contract with stable object IDs.
- [ ] Specify meters, coordinate axes, origin, rotation convention, and transform semantics.
- [ ] Describe room boundaries, openings, objects, geometry or asset references, materials, and cameras/lights.
- [ ] Distinguish measured, inferred, and unknown values, with reference-image evidence.
- [ ] Define structured evaluation findings: object ID, issue, severity, evidence, suggested correction, and unresolved uncertainty.
- [ ] Define acceptance criteria and numerical tolerances before evaluating candidates.
- [ ] Create a small synthetic fixture that exercises the contract without an API call.

Done when a builder can construct the fixture without guessing missing semantics and an evaluator can identify concrete violations.

### 3. Build deterministically in Blender

- [ ] Set up Python dependencies and verify the local Blender executable/version.
- [ ] Implement scene validation and a minimal builder for walls, openings, and simple furniture.
- [ ] Reject unsupported geometry/assets with actionable errors.
- [ ] Produce a Blender scene, camera-matched previews, and a GLB export.
- [ ] Verify repeatability using the same input, asset versions, Blender version, and seeds; compare geometry/transforms rather than binary file identity.

Done when the synthetic fixture builds and exports successfully without a model call.

### 4. Interpret reference images

- [ ] Connect a vision-capable model to the scene contract through structured outputs.
- [ ] Generate a first room description with evidence and explicit uncertainties.
- [ ] Render the description through the existing builder without room-specific code changes.
- [ ] Record model/prompt versions, input IDs, and the scene revision for reproducibility.

Done when photographs produce a valid, inspectable first reconstruction; visual accuracy is evaluated separately.

### 5. Evaluate and revise

- [ ] Implement deterministic checks for measured dimensions, object transforms, invalid geometry, and relevant collisions/clearances.
- [ ] Compare source photos with matched-camera renders for layout, missing objects, proportions, and materials.
- [ ] Distinguish camera-alignment errors, scene-data errors, and builder defects before proposing fixes.
- [ ] Exercise one author/evaluator cycle with explicit scene revisions and structured findings.
- [ ] Implement a bounded orchestrator with saved candidates, evaluation history, and stopping outcomes.
- [ ] Check the selected candidate against the reserved reference view without using that view to tune it.

Done when a run can finish as `accepted`, `needs_evidence`, or `budget_exhausted`, retaining its best candidate and unresolved issues. A failed tool/API operation must be reported separately, never counted as acceptance.

### 6. Explore in the browser

- [ ] Build a minimal local Three.js frontend that loads `output/benchmark.glb`, with no backend or model API calls required.
- [ ] Add drag-to-rotate, scroll-to-zoom, and kitchen, living-room, and overhead camera buttons.
- [ ] Preserve object/group identity and add wall/ceiling visibility controls for cutaway views.
- [ ] Show an object's name and estimated dimensions when clicked, using scene data and its measurement/inference labels.
- [ ] Verify the exported scene and basic interaction in a browser.
- [ ] Add walking controls only after the core viewing experience works.

Done when a saved reconstruction is navigable in a local webpage without Blender knowledge or a live model call.

### 7. Test generalization

- [ ] Reconstruct a second room using the same contract and builder.
- [ ] Record where new assets suffice and where the contract or builder needs extensions.
- [ ] Compare fidelity, correction count, runtime, and model cost using the same evaluation procedure.

Done when we can explain what generalizes, what fails, and what input coverage the workflow requires.

## Evaluation principles

Geometric correctness and visual resemblance are separate. Hard checks should use measured constraints and explicit tolerances. Visual findings must cite reference images and candidate views; a single aesthetic score cannot establish reconstruction accuracy.

Start the pilot with an initial candidate and at most three revision cycles. Set the API spending and rendering budgets before enabling an automated run. These are planned controls, not implemented enforcement. Stop early when missing observations prevent a defensible correction, and retain the best candidate if a revision regresses.

Passing means satisfying the declared checks within observed coverage. It does not establish the geometry of unseen areas. Keep unknowns visible instead of converting plausible guesses into measurements.

## Inspiration

- [Original X post](https://x.com/rpnickson/status/2097488440489116111)
- [Studio in Bricks](https://studio-in-bricks.rpn24.chatgpt.site/)
- [Studio interior demo](https://studio-in-bricks.rpn24.chatgpt.site/interior)
- [Architectural visualization with Astra](https://developers.openai.com/blog/architectural-visualization-with-astra)

Inspection confirmed that the interior demo loads a Blender-exported GLB through Three.js. Its original prompts and any intermediate scene schema remain unverified. Reference implementations and assets are not copied into this project.
