# Explain Development Pipeline

Explain is a physiological simulation engine for neonatology, modeling cardiac and respiratory systems as a network of small typed components (capacitances, resistors, time-varying elastances, gas exchangers, diffusors) wired together by a JSON `model_definition`. The same engine exists in three implementations — JavaScript, Python, and Rust — each serving a different stage of the development lifecycle.

This document describes how those three implementations fit together as a single development pipeline.

## The three stages

### 1. JavaScript — develop and validate

New physiology is developed and tested in `explain-user-js/src/explain`. This is the only environment that combines all of the following in one place:

- A realtime Web Worker simulation loop (`ModelEngine.js`, 15 ms tick driving ~30 micro-steps per tick at the default `modeling_stepsize` of 0.5 ms).
- The interactive Vue 3 / Quasar UI with diagrams, animations, and live charts.
- `setPropValue` with tweening, `callModelFunction` for deferred method calls, and a `TaskScheduler` that lets the UI mutate the running model without racing the step loop.
- Direct, low-latency event subscriptions (`explain.on("rtf", …)`) for canvas-based rendering at ~66 Hz.

The result is a tight feedback loop: a change in `Heart.calc_model` produces a visibly correct waveform within seconds. That makes JS the only sensible place to judge whether a new model *behaves* correctly — not just runs.

### 2. Python — research, ML, and publishing

Once a model is stable in JS, it is ported to `explain-user-python`. The port is deliberately a 1:1 mirror:

- Same `BaseModelClass` contract.
- Same `{key, value}` `init_model` argument shape.
- The same JSON `model_definition` files load unchanged.

That makes the port mechanical and keeps the two implementations trustworthy as a pair. From the Python side, the model is just `Engine.load_file(path)` + `Engine.run(duration_s, watch=[...])` returning a list of dicts (or a CSV). That opens the door to the scientific Python stack:

- NumPy / Pandas for analysis.
- scikit-learn, PyTorch, etc. for ML on simulated cohorts.
- Matplotlib / seaborn for figures.
- Jupyter notebooks for reproducible papers.

This is the implementation used for academic work — the Python ecosystem is the lingua franca for journals, reviewers, and ML research.

### 3. Rust — long-running simulation and sweeps

When research demands runs that are too slow in Python (or even in JS) — parameter sweeps, sensitivity analyses, Monte Carlo experiments, RL training loops, multi-hour simulated ICU stays — the model is ported to `explain-user-rs`.

Per `USAGE.md`, the Rust engine runs roughly 18× faster than the Python engine and has zero runtime allocation in its hot loop, so wall-time is predictable across long runs. Architecturally it diverges from JS/Python: instead of a dictionary of polymorphic objects, every `model_type` gets its own `SlotMap<XId, X>` arena, and step dispatch is a `match` over a `Copy` `ModelRef` enum tag (`crates/explain-core/src/state.rs`, `step.rs`). The structs themselves are auto-generated from `codegen/inventory.json`.

The "oracle bundle" (`oracle/*.npz` + the CLI `validate` subcommand) is what makes the Rust port safe to use: every phase of the rewrite must reproduce the Python engine's traces within documented tolerance bands before it is accepted. The Rust engine is, by construction, the same model — just faster.

The `explain-py` crate (PyO3 bindings) means researchers do not have to leave their Python notebook to use it. Importing `explain_rs` in place of `explain` runs the same script against the same scenarios, just at Rust speed.

## Summary of leverage

Each stage of the pipeline adds a different kind of leverage without changing the underlying model:

| Stage      | Implementation         | Leverage                                     |
|------------|------------------------|----------------------------------------------|
| Develop    | `explain-user-js`      | Interactivity — see physiology in real time  |
| Research   | `explain-user-python`  | Ecosystem — NumPy, ML, papers, notebooks     |
| Scale out  | `explain-user-rs`      | Throughput — long runs, sweeps, predictability |

In one line: **JS for "is the physiology right and does it feel right", Python for "what does it mean and how do we publish it", Rust for "now run it a million times."**

## What makes the pipeline work

The contract that holds all three implementations together is small and explicit:

1. The JSON `model_definition` format — the same files load in all three engines.
2. The `BaseModelClass` shape — a static `model_type` and `model_interface`, plus `init_model(args)` / `step_model()` / `calc_model()`.
3. The two-pass build (instantiate everything, then init everything) so cross-component references resolve cleanly.

Stick to that contract and the porting cost is mechanical: Python is a near-line-for-line translation, and Rust is mostly codegen plus the hand-written `step_<type>` math. Break the contract and the pipeline gets expensive — every implementation has to be redesigned to accommodate the change.

This is why the JS engine is treated as canonical: it is where the contract is defined and where new physiology is shaped before the rest of the pipeline picks it up.
