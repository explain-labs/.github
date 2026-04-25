# Explain Labs

**Explain** is a physiological simulation engine for neonatology. It models the cardiac and respiratory systems as a network of compartments — capacitances, resistors, time-varying elastances, gas exchangers, and diffusors — wired together by a JSON `model_definition` and stepped at 2 kHz. Scenarios range from a single two-compartment toy model to the full 161-component term neonate.

The same engine exists in three implementations, each tuned for a different stage of work:

## The three implementations

### [`explain-user-js`](https://github.com/explain-labs/explain-user-js) — develop and validate

A Vue 3 / Quasar web app with the simulation running in a Web Worker. Realtime waveforms, interactive diagrams, live parameter tweening through a `TaskScheduler`. This is where new physiology models are developed and where you go to *see* the model behave.

### [`explain-user-python`](https://github.com/explain-labs/explain-user-python) — research, ML, and publishing

A 1:1 Python port of the JS engine. Same JSON scenarios load unchanged, same per-step physics. The engine itself has zero external dependencies — drop a `from explain import Engine` into a notebook and you get the whole scientific Python stack: NumPy, Pandas, Matplotlib, scikit-learn, PyTorch, Jupyter for paper figures.

### [`explain-user-rs`](https://github.com/explain-labs/explain-user-rs) — long simulations and parameter sweeps

A Rust port roughly 18× faster than the Python engine, with zero allocation in the hot loop. Used when Python and JS are too slow — Monte Carlo runs, sensitivity analyses, RL training loops, multi-hour simulated stays. Ships PyO3 bindings (`explain_rs`), so research scripts can swap engines without leaving the notebook. Validated continuously against a Python "oracle bundle" so the two engines provably agree on physiology.

## How they fit together

Develop in JavaScript because of the realtime UI; port to Python for research and publishing because that's the academic lingua franca; port to Rust when you need to run the same model thousands of times. The contract that makes this work — a JSON `model_definition` plus a small `BaseModelClass` shape — is identical across all three implementations.

For the full pipeline rationale, the porting workflow, and install requirements, see the project-wide docs in this org's [`.github`](https://github.com/explain-labs/.github) repository:

- [**DEVELOPMENT_PIPELINE.md**](https://github.com/explain-labs/.github/blob/main/docs/DEVELOPMENT_PIPELINE.md) — why the project lives in three implementations and which one fits which task.
- [**PORTING.md**](https://github.com/explain-labs/.github/blob/main/docs/PORTING.md) — step-by-step manual for taking a new model from JS to Python to Rust.
- [**REQUIREMENTS.md**](https://github.com/explain-labs/.github/blob/main/docs/REQUIREMENTS.md) — what to install per implementation, per operating system.

## Getting started

```sh
# Clone all three implementations
git clone https://github.com/explain-labs/explain-user-js.git
git clone https://github.com/explain-labs/explain-user-python.git
git clone https://github.com/explain-labs/explain-user-rs.git
```

Each repo's README covers its build commands and directory layout. Most contributors only need one or two of the three depending on what they're working on.

## Maintainer

**Tim Antonius**  
Neonatologist · Radboudumc Amalia Children's Hospital, Department of Pediatrics – Division of Neonatology  
[GitHub](https://github.com/Dobutamine) · [Email](mailto:timantonius@pm.me)
