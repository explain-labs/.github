# Porting a model: JavaScript → Python → Rust

This is the working manual for taking a new model from `explain-user-js` through to `explain-user-python` and (if needed) `explain-user-rs`. It assumes the model has already been developed and validated interactively in the JS engine, and you now want to make it available to research scripts and (eventually) long-running Rust simulations.

The manual is deliberately mechanical. The contract that makes the three implementations interchangeable — the JSON `model_definition` plus the `BaseModelClass` shape — is small and well-defined; if you stick to it, porting is mostly translation, not design.

For the bigger-picture rationale of why the pipeline exists in three stages, see `DEVELOPMENT_PIPELINE.md`.

## When to port what

You don't always need all three. Use this table to decide:

| Goal                                                             | Stops at |
|------------------------------------------------------------------|----------|
| Try out a new physiology model interactively, see waveforms      | JS       |
| Write a paper, run analysis on simulated data, train an ML model | Python   |
| Run thousands of simulations, parameter sweeps, multi-hour runs  | Rust     |

A model lives in JS as soon as it exists. It moves to Python once anyone wants to run it from a notebook or feed its output into the scientific Python stack. It moves to Rust only when Python is too slow for the experiment — typically when you'd want hundreds of runs or hours of simulated time.

## Step 0 — preconditions

Before starting any port, the JS model must be:

- Stable. Don't port a model whose physiology you're still tuning — the cost of re-doing the Python and Rust ports is higher than the cost of waiting.
- Wired in. The class must be exported from `explain-user-js/src/explain/ModelIndex.js` and at least one scenario in `explain-user-js/public/model_definitions/` must reference it.
- Verified by eye in the running app. Run a scenario that uses it, watch the waveforms, confirm it does what you expect.

If any of those isn't true, finish in JS first.

---

## Stage 1 — JavaScript (the canonical implementation)

This stage isn't part of the port — it's the source of truth. Two pieces of the JS work matter for the downstream ports:

1. The class file under `explain-user-js/src/explain/{base_models,component_models,device_models}/<ClassName>.js`. Use the `new-model-scaffolder` agent (in `.claude/agents/`) to generate the skeleton if you're starting from scratch — it produces the correct `BaseModelClass` shape, registers the export in `ModelIndex.js`, and avoids drift from the existing convention.

2. The instance entry in one or more `public/model_definitions/*.json` scenarios. This JSON is what the Python and Rust engines will load unchanged.

The pattern that needs to survive the ports:

- Static `model_type` string matching the class name.
- Static `model_interface` array describing UI-editable fields.
- Constructor that sets every persistent field to a default (independent properties, dependent properties, non-persistent factors, persistent `_ps` factors, scaling `_scaling_ps` factors).
- `calc_model()` containing the physiology, optionally split into helper methods (`calc_resistance`, `calc_flow`, etc.).
- Convention: underscore-prefixed fields (`_comp_from`, `_prev_flow`) are local/internal and excluded from serialized state.

`base_models/Capacitance.js` and `base_models/Resistor.js` are the cleanest worked examples and are good templates for "what good looks like" before you start translating.

---

## Stage 2 — Python port (`explain-user-python`)

The Python port mirrors the JS source one-to-one. The two engines load the same scenario JSON and produce the same physiology, so the goal is a faithful translation, not a redesign.

### File and naming conventions

| JS                                              | Python                                          |
|-------------------------------------------------|-------------------------------------------------|
| `src/explain/base_models/Capacitance.js`        | `explain/base_models/capacitance.py`            |
| `src/explain/component_models/Heart.js`         | `explain/component_models/heart.py`             |
| `src/explain/device_models/Ventilator.js`       | `explain/device_models/ventilator.py`           |
| Class `Capacitance`                             | Class `Capacitance` (PascalCase preserved)      |
| `static model_type = "Capacitance"`             | Class attribute `model_type = "Capacitance"`    |
| `static model_interface = [...]`                | Class attribute `model_interface = [...]`       |
| `constructor(model_ref, name = "")`             | `def __init__(self, model_ref, name: str = "")`|

File names are snake_case; class names are PascalCase and identical to JS. `model_type` strings are always identical across all three implementations — they're the lookup key in every scenario JSON.

### Translation rules

The translation is mostly mechanical. The conventions used throughout the existing port:

- `this.foo` → `self.foo`. No exceptions.
- `Math.pow(x, 2)` → `x ** 2` (or `(stretch) ** 2` — match what's already in the file).
- `this.calc_*()` helper methods translate one-to-one as `self.calc_*` methods on the class.
- `model_interface` can be trimmed in the Python copy — the existing port keeps only the fields needed for the Python ecosystem (no UI uses it). Compare `Capacitance.js` (~80-line interface) to `capacitance.py` (~6 entries). Don't drop entries that are used by `build_prop: true` semantics if you need full fidelity, but for typical research use the trimmed form is fine.
- Underscore-prefixed fields stay underscore-prefixed.
- `volume_in` / `volume_out` keep the same signatures, including the "amount that could not be removed" return convention.
- Use `from __future__ import annotations` at the top so type hints work cleanly.

### Steps

1. **Create the file** at the path matching the JS location (snake_case filename, same subfolder). Start with the docstring shown in `capacitance.py`:
   ```python
   """ClassName — Python port of src/explain/<subdir>/ClassName.js
   
   <one-paragraph description of what the model does and the equations it implements>
   """
   ```

2. **Translate the class.** Use `Capacitance` and `Resistor` as the reference templates — they cover the most common patterns (factor tiers, volume transfer, component lookup by name).

3. **Register in `model_index.py`.** Add an `import` at the top and an entry in the `MODEL_INDEX` dict. The dict key must be the same string as `model_type`. Existing entries follow alphabetical order within each category section — keep that.

4. **Run a scenario through it.** From `explain-user-python`:
   ```sh
   python3 main.py
   ```
   `main.py` runs the `term_neonate` scenario for 30 simulated seconds and writes a CSV under `output/`. Add a watch tuple for one of your new model's properties and confirm the value moves the way it does in the JS app.

5. **Sanity-check by comparing.** If your model already exists in a JS scenario and produces a known waveform, run the same scenario in Python with a small `sample_every_s` and overlay the two CSVs. Numerical drift over a few seconds should be at the floating-point noise floor — if it's not, you've translated something wrong.

The Python port is finished when its CSV output for the same scenario matches the JS output's behavior to the eye. The Rust pipeline relies on the Python output being correct, so don't move on until this step is solid.

---

## Stage 3 — Rust port (`explain-user-rs`)

The Rust port uses code generation, not hand translation. The struct definitions are auto-generated from the Python source by inspecting `__init__` defaults and scanning the scenario JSON files. You only hand-write the per-step physiology function and the wiring around it.

The full workflow is documented in `explain-user-rs/CLAUDE.md` ("Workflow for adding/modifying a model_type") — what follows is the porting-flavored version, with pointers to the right slash commands and helpers.

### One-time orientation

Read these once before starting:

- `explain-user-rs/CLAUDE.md` — architecture summary, snapshot-then-mutate pattern, the helper functions in `step.rs` (`blood_read`, `chamber_vol`, `valve_flow`, etc.).
- `explain-user-rs/USAGE.md` — build commands, engine API, common watch tuples.
- `explain-user-rs/README.md` — oracle bundle format, tolerance bands per quantity type.

### Steps

1. **Make sure the Python port runs.** The Rust codegen reads from `../explain-user-python` — it inspects `__init__` defaults to figure out struct field types and defaults. If the Python class isn't in `MODEL_INDEX`, codegen won't see it.

2. **Regenerate the structs:**
   ```sh
   cd explain-user-rs
   python3 codegen/inventory.py        # only if Python source changed
   python3 codegen/generate.py
   ```
   Or use the `/codegen` slash command, which runs both and reports the diff. The output is `crates/explain-core/src/generated/{mod.rs, models.rs}` — never hand-edit these files. If the generated struct has the wrong type (e.g. an `i64` field that should be `f64`, or a `serde_json::Value` that should be a typed `IndexMap`), fix the relevant table in `codegen/generate.py` (`SCALAR_DTYPE_OVERRIDES`, `TYPED_DICT_FIELDS`, `TYPED_LIST_FIELDS`, `RESOLVED_REFS`, `EXTRA_FIELDS`) and re-run.

3. **Wire the new type into `state.rs`:**
   - Add a new ID type in the `new_key_type!` block (e.g. `pub struct LymphaticVesselId;`).
   - Add a `SlotMap<LymphaticVesselId, LymphaticVessel>` arena field on `State`.
   - Add a `LymphaticVessel(LymphaticVesselId)` variant to the `ModelRef` enum.
   - Search for `Mob2` in `state.rs` — `CLAUDE.md` flags it as the canonical pattern to copy.

4. **Add a load-time reference resolver** (only if your model has cross-model references, like a Resistor's `comp_from` / `comp_to`). Add a `resolve_<modeltype>_refs` pass to `load.rs` and wire it into the resolution pipeline **after** the model types it depends on. Same constraint exists for Blood and Gas in Python — the order matters.

5. **Write the step function.** Add `step_<modeltype>(state, id)` to `step.rs`, then add a match arm in `step_one`. Two patterns to follow:
   - **Snapshot-then-mutate** when one step touches multiple arenas. Destructure all needed fields into locals first, then write back at the end. The borrow checker depends on this.
   - **`SlotMap::get_disjoint_mut`** when you need mutable access to two entries in the same arena (e.g. a Resistor reading both endpoints in the BloodCapacitance arena).
   
   Use the existing helpers — don't reinvent. The CLAUDE.md "Helpers in step.rs worth knowing about" section is the inventory.

6. **Stub first, then fill in.** The CLAUDE.md flags this and it's worth following: commit a `{}` no-op step body first and run the validator. Drift unchanged confirms the plumbing is right (load, codegen, dispatch all working). Then add the physics body and re-validate. This isolates plumbing bugs from physics bugs, which look very different in a drift report.

7. **Build and validate:**
   ```sh
   cargo build --release
   source .venv/bin/activate
   maturin develop --release -m crates/explain-py/Cargo.toml   # or /rebuild-py
   python3 oracle/compare.py                                    # or /validate
   ```
   `oracle/compare.py` runs the same scenario through both the Python and Rust engines and reports per-column drift plus the speedup. The `/validate` slash command summarizes the report. Drift on watched physiology columns must stay within the per-quantity tolerance bands documented in `README.md` (volumes 1e-9, pressures 1e-6, flows 1e-9, blood pH 5e-5, etc.).

### What "validated" means

The Rust port of a model is finished when:

- It loads from the same scenario JSON the Python engine loads.
- `oracle/compare.py` reports drift within tolerance for every column belonging to that model.
- State-machine integers (e.g. `cardiac_cycle_state`) and boolean flags (e.g. `_systole_running`) match exactly — no floating-point slack.

If drift on a column you weren't expecting suddenly appears after your change, the failing column almost always points at one of: an init-default mismatch (Python defaults to a different value than the Rust struct), a dropped non-persistent factor reset (you forgot the `c.foo_factor = 1.0;` line at the end of the step), a step-order bug (your model resolved before something it depends on), or a borrow-pattern bug (read-after-write that should have been read-before-write).

---

## Common pitfalls across all three stages

A few mistakes show up repeatedly during ports. They are easier to avoid than to debug.

The `model_type` string must be **identical** across JS, Python, and Rust. It's the lookup key in `MODEL_INDEX` / `model_index.py` / the `match` in `step.rs`, and it's what scenario JSON files reference. A typo here produces a mysterious "model not found" at load time in whichever engine you forgot to update.

Field names must also be **identical** across all three. The Python side is loaded with `setattr(self, key, value)` straight from the JSON, and the Rust side is generated from Python's `__init__` defaults. Renaming a field in just one place breaks the contract silently — JS still works, Python silently grows a new attribute, Rust may fail codegen or refuse to load the scenario.

`init_model` always takes **a list of `{key, value}` dicts**, not a dict. This is preserved across JS and Python so the loader's shape is shared 1:1. If you forget and accept `**kwargs`, your component models that recurse through `self.components` will break.

**Underscore-prefixed fields are local.** They don't get serialized into model state, they don't need to round-trip through JSON, and the Rust codegen filters them out of inventories. Use them for cached references, intermediate calculations, and previous-step snapshots — not for anything the UI or a scenario JSON needs to read.

**Insertion order matters.** Both JS (`for const k in obj`) and Python 3.7+ (`dict.values()`) iterate in insertion order, which is what the step loop relies on. Rust uses `IndexMap` and a precomputed `StepPlan` for the same reason. The `serde_json` `preserve_order` feature is non-negotiable in the Rust port — without it, struct field order goes alphabetical and resistors start reading stale neighbor pressures. The README in `explain-user-rs` documents this with a specific drift fingerprint (`AA.vol` drift of exactly 8.4e-9) so you can recognize it.

**Blood and Gas auto-discover other models** by scanning `model_type` at load time. Their declarations must come **last** in the scenario JSON. This applies in both Python and Rust.

---

## End-to-end checklist

Use this as a per-port punch list. The model is fully ported when every box is checked.

**JavaScript (the source of truth):**

- New file under `src/explain/{base,component,device}_models/<ClassName>.js`
- Exported from `src/explain/ModelIndex.js`
- Referenced by at least one scenario in `public/model_definitions/`
- Verified by eye in the running app

**Python:**

- New file under `explain/{base,component,device}_models/<class_name>.py` (snake_case)
- Class name PascalCase, `model_type` string identical to JS
- Constructor sets every persistent field to a default
- `calc_model` reproduces the JS arithmetic
- Imported and registered in `explain/model_index.py`
- `python3 main.py` runs the scenario without error
- Output for one watched property matches the JS waveform to the eye

**Rust (only when you need long runs / sweeps):**

- `python3 codegen/inventory.py && python3 codegen/generate.py` regenerates cleanly with no surprise type mismatches
- ID type, arena, and `ModelRef` variant added to `state.rs`
- Load-time ref resolver added to `load.rs` (if the model has cross-references)
- `step_<modeltype>` written in `step.rs`, wired into the `step_one` match
- `cargo build --release` succeeds
- `maturin develop --release` rebuilds the Python extension cleanly
- `python3 oracle/compare.py` reports drift within per-quantity tolerance bands for every column belonging to the new model
- Speedup vs Python is in the same ballpark as other models (~10–20×)

Once the Rust validator passes, the model is available everywhere: in the JS app for interactive work, in Python notebooks for research, and in the Rust engine (via `explain_rs`) for long-running experiments — all backed by the same scenario JSON files and producing the same physiology.
