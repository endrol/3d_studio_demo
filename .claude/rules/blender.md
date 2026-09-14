# Blender

Verified on the development Mac, 2026-09-12. Re-verify on a different machine before relying on
paths or versions.

## Invocation

Blender runs headless; there is no GUI control path in this project.

```bash
/Applications/Blender.app/Contents/MacOS/Blender --background --python build_scene.py -- <args>
```

Everything after the bare `--` reaches the script as `sys.argv`; Blender consumes the rest. Record
the Blender version, input scene revision, asset versions, and seeds with every run.

## Version facts

- Blender 5.0.1, build date 2025-12-16.
- The render engine enum is `BLENDER_EEVEE`, not `BLENDER_EEVEE_NEXT`. 5.x accepts only
  `('BLENDER_EEVEE', 'BLENDER_WORKBENCH', 'CYCLES')`; the `_NEXT` name from 4.2–4.5 raises
  `TypeError` at assignment. Prefer `BLENDER_WORKBENCH` for geometry-only checks where shading does
  not matter — it is faster and its output does not vary with lighting changes.

## Process boundary

Blender uses its own bundled Python. Packages installed by uv are not importable inside it, and
nothing should assume otherwise. The uv environment owns schema validation, orchestration, and
tests; Blender receives validated scene JSON on disk or via argv and returns files plus stdout.

Validate scene data on the uv side. A builder script that has to defend against malformed input is
a sign the boundary contract leaked.

## Writing builder scripts

- Start from a known state: `bpy.ops.wm.read_factory_settings(use_empty=True)`. The default startup
  scene ships a cube, camera, and light that will otherwise appear in renders and geometry checks.
- `bpy.ops.render.render(write_still=True)` needs `scene.render.filepath` set first; in background
  mode nothing is written without `write_still`.
- Keep room-specific coordinates in scene data, never in the script. Preserve stable object IDs as
  Blender object names so geometry checks, evaluator findings, and viewer visibility controls all
  address the same thing.
- Reject unsupported geometry or a missing asset with an actionable error naming the object ID and
  the field. Do not substitute a placeholder and continue.

## Geometry readback

Deterministic checks read the built scene back rather than trusting the builder's own log:

```python
for o in bpy.data.objects:
    if o.type == 'MESH':
        d = o.dimensions
        print(f"CHECK {o.name} dims=({d.x:.3f},{d.y:.3f},{d.z:.3f}) "
              f"loc=({o.location.x:.3f},{o.location.y:.3f},{o.location.z:.3f})")
```

`dimensions` reports world-space extent after scale, which is what measured constraints are stated
in. Compare geometry and transforms against tolerances; never require byte-identical `.blend` or
`.glb` files for repeatability.

## Renders for evaluation

Camera-matched preview renders are evidence, so they are cheap and reproducible by default: small
resolution, fixed samples, fixed seed, deterministic file names carrying the candidate revision.
Match the camera to the reference view before attributing any discrepancy to geometry. Stay inside
the render budget assigned for the task.
