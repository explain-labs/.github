# Explain Labs

**Explain** is a physiological simulation engine for neonatology. It models the cardiac and respiratory systems as a network of compartments — capacitances, resistors, time-varying elastances, gas exchangers, and diffusors — wired together by a JSON `model_definition` and stepped in real time. Scenarios range from a single two-compartment toy model to a full term neonate with congenital heart disease, a ventilator, and ECLS.

The project lives in two repositories: a framework-agnostic engine, and the web app that runs it.

## The repositories

### [`explain-engine`](https://github.com/explain-labs/explain-engine) — the physics

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21389097.svg)](https://doi.org/10.5281/zenodo.21389097)

The simulation engine itself: plain ES modules, no runtime or build dependencies, no framework. It runs inside a Web Worker (`ModelEngine.js`) and is driven from the main thread through the `Model` wrapper (`Model.js`). Every model component — `Heart`, `Breathing`, `Blood`, `Pda`, `Ventilator`, … — extends `BaseModelClass` and implements a single `calc_model()` step.

This repo also carries the canonical scenario definitions (`model_definitions/`), a headless Node harness under `scripts/`, and the physiological derivation for each model in [`docs/`](https://github.com/explain-labs/explain-engine/tree/main/docs). Read [`docs/ARCHITECTURE.md`](https://github.com/explain-labs/explain-engine/blob/main/docs/ARCHITECTURE.md) for the two-thread design and the worker wire protocol.

Consumers mount it as a **git submodule**.

### [`explain-ui`](https://github.com/explain-labs/explain-ui) — the app

The interactive web app: Vue 3 + Vite + TypeScript + PrimeVue + Tailwind, with `explain-engine` mounted as a submodule at `explain-engine/`. Realtime waveforms, interactive diagrams, live parameter tweening, monitor and ventilator screens. This is where you go to *see* the model behave.

The parameter-edit schema lives here rather than in the engine (`src/model-interface/`), which keeps the engine free of UI concerns. UI-layer documentation is in [`docs/ui/`](https://github.com/explain-labs/explain-ui/tree/main/docs/ui).

## Getting started

Almost everyone wants the app — it brings the engine along as a submodule. You need [git](https://git-scm.com) and [Node.js 20+](https://nodejs.org).

**macOS / Linux**

```sh
curl -fsSLO https://raw.githubusercontent.com/explain-labs/explain-ui/main/scripts/run-explain.sh
bash run-explain.sh
```

**Windows (PowerShell)**

```powershell
irm https://raw.githubusercontent.com/explain-labs/explain-ui/main/scripts/run-explain.ps1 -OutFile run-explain.ps1
powershell -ExecutionPolicy Bypass -File .\run-explain.ps1
```

The script clones the repo with its submodule, installs dependencies, and starts the dev server — open the URL it prints. Re-run it any time to update and restart.

By hand, the equivalent is:

```sh
git clone --recurse-submodules https://github.com/explain-labs/explain-ui.git
cd explain-ui
npm install
npm run dev
```

If you only need the physics — batch runs, sensitivity sweeps, embedding Explain in something that isn't this app — clone the engine on its own and drive it from Node:

```sh
git clone https://github.com/explain-labs/explain-engine.git
```

## Contributing a model

Students and contributors build new models and scenarios on their own branch, through a dedicated seam (`explain-engine/custom_models/` plus `src/model-interface/custom-registry.ts`) that stays empty on `main` so branches rebase cleanly. The walkthrough is in [**STUDENT_WORKFLOW.md**](https://github.com/explain-labs/explain-ui/blob/main/STUDENT_WORKFLOW.md), with a one-time setup script (`scripts/setup-student.sh` / `.ps1`).

In short: a new model is a class with a static `model_type`, an `init_model()`, and a `calc_model()`; an export line to register it; and a field list so its parameters become editable in the app.

## Background and history

The scientific background — the derivations and the validation work behind the models — is written up in [`explain-thesis`](https://github.com/Dobutamine/explain-thesis).

Earlier generations of the project live on in this org as archived, read-only repositories: [`explain-user-js`](https://github.com/explain-labs/explain-user-js), the Quasar-based predecessor of `explain-ui`, and [`explain-user-python`](https://github.com/explain-labs/explain-user-python), a Python port of the engine. The [`docs/`](https://github.com/explain-labs/.github/tree/main/docs) folder of this org's `.github` repository describes that older three-implementation pipeline (JavaScript / Python / Rust) and is kept for reference.

## Maintainer

**Tim Antonius**  
Neonatologist · Radboudumc Amalia Children's Hospital, Department of Pediatrics – Division of Neonatology  
[GitHub](https://github.com/Dobutamine) · [Email](mailto:timantonius@pm.me)
