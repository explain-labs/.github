# Requirements

What you need installed to use the Explain modeling pipeline. Split per implementation (JavaScript, Python, Rust) and per operating system (macOS, Windows, Linux).

You don't need everything. Start with the implementation you actually want to use — `DEVELOPMENT_PIPELINE.md` explains which one fits which task. Most users will only need the JavaScript app; researchers add Python; long-running simulations add Rust.

## Quick lookup

| If you want to…                                              | Install                                                  |
|--------------------------------------------------------------|----------------------------------------------------------|
| Run the Explain web app interactively                        | A modern browser, that's it                              |
| Develop in / build the JS engine                             | Node 18+, Yarn (vendored 4.1.1)                          |
| Run the Python engine for research, ML, or papers            | Python 3.10+. The engine itself has zero external deps   |
| Plot, analyze, or train ML on Python output                  | Python 3.10+ plus NumPy/Pandas/Matplotlib/Jupyter (your call) |
| Build the Rust CLI for long simulations                      | Rust 1.75+ (via rustup), Cargo                           |
| Use the Rust engine from Python (`explain_rs`)               | Rust 1.75+, Python 3.8+, maturin 1.4+, NumPy             |

If you're new, the recommended starting set per OS is at the bottom.

---

## JavaScript implementation (`explain-user-js`)

The JS engine ships as a Quasar Vue 3 app with Vite. The simulation runs in a Web Worker, so anything that runs the bundle — a modern browser, an Electron build, or Node-driven dev server — works.

### Universal requirements

- **Node.js:** version 18, 20, 22, or 24 (the `engines` field in `package.json` lists `^16 || ^18 || ^20 || ^22 || ^24`; 16 still works but is past EOL — use 20 LTS or newer).
- **Yarn:** the repo ships Yarn 4.1.1 (Berry) at `explain-user-js/.yarn/releases/yarn-4.1.1.cjs` and uses the project-local binary via Corepack. You don't install Yarn globally — Corepack picks it up automatically. If you don't have Corepack, npm 6.13.4+ also works.
- **A modern browser** for running the dev server. Chrome, Firefox, Safari, Edge — Chromium-based gives you the best DevTools experience for the Web Worker.
- *(Optional)* **Electron** is in `devDependencies` and is auto-installed by `yarn`. Used only for desktop builds (`quasar dev -m electron`).

### macOS

Use Homebrew (`https://brew.sh`):

```sh
brew install node          # gets the latest LTS
corepack enable            # ships with Node 16.10+; activates Yarn from the repo
```

If you prefer pinned-version management, install `nvm` instead:

```sh
brew install nvm
nvm install 20             # or 22 / 24
nvm use 20
corepack enable
```

### Windows

Three good options, pick one:

1. **Official Node installer** from `https://nodejs.org` (LTS). Run it, then in PowerShell:
   ```powershell
   corepack enable
   ```
2. **Scoop:**
   ```powershell
   scoop install nodejs-lts
   corepack enable
   ```
3. **WSL2 + Ubuntu** if you want a Linux-like dev environment. Then follow the Linux instructions.

For Electron builds, Windows works fine with the standard installer — no extra build tools needed.

### Linux

```sh
# Debian / Ubuntu
sudo apt update && sudo apt install -y nodejs npm
sudo corepack enable

# Fedora
sudo dnf install -y nodejs npm
sudo corepack enable

# Arch
sudo pacman -S nodejs npm
sudo corepack enable
```

If your distro packages an older Node, use `nvm` (`https://github.com/nvm-sh/nvm`) to install a newer one without touching system packages.

### Verify

From `explain-user-js`:

```sh
yarn               # installs deps; first run takes a minute
yarn dev           # or: quasar dev — opens at http://localhost:9000
```

---

## Python implementation (`explain-user-python`)

The Python engine is deliberately minimal. The core engine — `explain/engine.py`, `explain/base_model.py`, every model class — uses only the standard library (`csv`, `json`, `pathlib`, `math`). There is no `requirements.txt` or `pyproject.toml` because there are no dependencies to pin.

The packages you'll likely add on top — NumPy, Pandas, Matplotlib, scikit-learn, PyTorch, Jupyter — are your research stack of choice, not project requirements. Install only what your work needs.

### Universal requirements

- **Python 3.10+.** The codebase uses `from __future__ import annotations` and modern type-hint syntax. Existing `__pycache__` directories show the project running on both 3.10 and 3.14, so anything in that range is fine.
- **pip** (ships with Python).
- **`venv`** module (ships with Python). Use a virtual environment so research dependencies don't pollute the system Python.

### Recommended research extras

These are not required to run the engine. Add the ones that match your workflow:

```
numpy          # required if you use Rust validation (oracle bundle is .npz)
pandas         # tabular analysis on the engine's CSV output
matplotlib     # plotting waveforms
scipy          # signal processing, statistics
scikit-learn   # classical ML on simulated cohorts
torch / jax    # deep learning on simulated cohorts
jupyter        # notebooks for paper figures and exploration
```

### macOS

Python 3 ships with macOS but is usually older and managed by Apple — don't install packages into it. Use Homebrew or pyenv:

```sh
brew install python@3.12
# or, for pinned-version management:
brew install pyenv
pyenv install 3.12
pyenv global 3.12
```

Then create a project-local venv:

```sh
cd explain-user-python
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
# optional research stack:
pip install numpy pandas matplotlib scipy jupyter
```

### Windows

Get Python from `https://www.python.org/downloads/` (check "Add Python to PATH" during install) or via Scoop / Chocolatey:

```powershell
scoop install python
# or
choco install python
```

Then in PowerShell:

```powershell
cd explain-user-python
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
# optional:
pip install numpy pandas matplotlib scipy jupyter
```

If `Activate.ps1` is blocked, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.

### Linux

```sh
# Debian / Ubuntu
sudo apt update && sudo apt install -y python3 python3-venv python3-pip

# Fedora
sudo dnf install -y python3 python3-virtualenv

# Arch
sudo pacman -S python python-pip
```

Then:

```sh
cd explain-user-python
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
# optional:
pip install numpy pandas matplotlib scipy jupyter
```

### Verify

```sh
cd explain-user-python
python3 main.py           # runs term_neonate for 30 sim seconds, writes a CSV under output/
```

---

## Rust implementation (`explain-user-rs`)

The Rust port has two build targets that need different toolchains:

- **Native CLI** (`explain-cli` + `explain-core`) — only needs Rust + Cargo.
- **Python extension** (`explain-py` → `explain_rs` package) — needs Rust + Python + maturin + a virtual environment.

Most people who care about Rust speed end up using both: the CLI for one-shot runs, and the Python extension for notebook-driven sweeps.

### Universal requirements

- **Rust 1.75+** (per `Cargo.toml`'s `rust-version = "1.75"`). Install via `rustup` — never use a distro package, because rustup is the only sane way to manage Rust toolchain versions.
- **Cargo** ships with rustup.
- **Python 3.8+** if you want the Python extension (per `crates/explain-py/pyproject.toml`'s `requires-python = ">=3.8"`). In practice, use the same 3.10+ you installed for the Python implementation.
- **maturin 1.4+** for building the Python extension. Install into the project venv: `pip install maturin`.
- **NumPy** if you want to use the oracle / validator (`oracle/*.npz` files). Install into the same venv: `pip install numpy`.
- A working **C/C++ toolchain** — needed by some Cargo dependencies for native compilation.

### macOS

```sh
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup default stable      # 1.75 or newer

# C/C++ toolchain (Xcode CLT covers it)
xcode-select --install

# Python venv for the extension
cd explain-user-rs
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install maturin numpy
```

**Important macOS quirk** (already documented in `explain-user-rs/CLAUDE.md`): a plain `cargo build --workspace` will *fail* to link the `explain-py` crate because PyO3's `extension-module` feature deliberately leaves Python's interpreter symbols unresolved. The workspace's `default-members` excludes `explain-py` so plain `cargo build` works, but if you want the Python extension you **must** build it through maturin:

```sh
maturin develop --release -m crates/explain-py/Cargo.toml
```

This sets the `-undefined dynamic_lookup` linker flag for you.

### Windows

```powershell
# Rust toolchain — installer at https://rustup.rs
# When prompted, accept the default "stable-msvc" host triple.

# Microsoft C++ Build Tools (the rustup installer offers to install these
# automatically on first run; if not, get them at:
# https://visualstudio.microsoft.com/visual-cpp-build-tools/)

# Python venv
cd explain-user-rs
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install maturin numpy
```

The macOS-specific PyO3 linker quirk does not apply on Windows — `cargo build` against `explain-py` works as long as Python is on PATH, but you should still use maturin for development because it handles the install-into-venv step.

### Linux

```sh
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup default stable

# Build essentials
sudo apt install -y build-essential pkg-config        # Debian/Ubuntu
sudo dnf groupinstall -y "Development Tools"          # Fedora
sudo pacman -S base-devel                             # Arch

# Python dev headers (needed by maturin / PyO3 for non-abi3 builds)
sudo apt install -y python3-dev                       # Debian/Ubuntu
sudo dnf install -y python3-devel                     # Fedora

# Python venv
cd explain-user-rs
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install maturin numpy
```

The Cargo.toml uses `pyo3 = { features = ["extension-module", "abi3-py38"] }`, so you build once against the stable ABI and the resulting wheel works on any Python 3.8+ — no matching of interpreter versions needed at build time.

### Verify

```sh
# Native CLI
cargo build --release
cargo run --release -p explain-cli -- check ../explain-user-python/model_definitions/term_neonate.json

# Python extension
source .venv/bin/activate
maturin develop --release -m crates/explain-py/Cargo.toml
python3 -c "from explain_rs import Engine; print(Engine())"

# Validator (round-trip Python vs Rust)
python3 oracle/compare.py
```

---

## Editor and tooling (optional but recommended)

These don't affect whether the project runs, only how pleasant the dev loop is.

- **VS Code** with the language extensions: Vue (Volar), Python, rust-analyzer. The repo includes `explain-user.code-workspace` which opens all three implementations side-by-side.
- **Git** for source control. macOS bundles it via Xcode CLT; Windows installer at `https://git-scm.com`; Linux: `apt install git` / `dnf install git` / `pacman -S git`.
- **Claude Code** (or another agentic assistant) to take advantage of the per-repo `CLAUDE.md` files and the existing slash commands (`/codegen`, `/validate`, `/rebuild-py` in the Rust crate; the `new-model-scaffolder` agent in the JS crate).

---

## Recommended starter sets

If you want a minimal install for each role:

**Just running the simulator (anyone, any OS):**
- A modern browser. Open the deployed app or `yarn dev` from a teammate's machine over the network.

**JS developer (someone adding new physiology):**
- Node 20+, Corepack, Git, VS Code with Volar.

**Researcher / academic (notebook-driven, papers, ML):**
- Python 3.12, venv, NumPy, Pandas, Matplotlib, Jupyter. Optionally scikit-learn or PyTorch. Git, VS Code with the Python extension.

**Long-simulation user (parameter sweeps, RL training, multi-hour runs):**
- Everything in the researcher set, plus Rust 1.75+ via rustup, maturin in the venv, a working C/C++ toolchain. Build the `explain_rs` extension once with maturin and import it in place of the Python `Engine`.

**Full-stack contributor (touches all three implementations):**
- All of the above. The `.code-workspace` file opens all three repos in one VS Code window, and `oracle/compare.py` becomes your regression gate when you change anything that crosses the boundary between Python and Rust.

---

## Troubleshooting one-liners

`yarn dev` fails with "command not found": run `corepack enable` once.

`maturin develop` fails with "Python.h not found" on Linux: install `python3-dev` (Debian/Ubuntu) or `python3-devel` (Fedora).

`cargo build --workspace` fails with linker errors mentioning `_PyBaseObject_Type` on macOS: don't use `--workspace`; build the Python extension via maturin instead. The default workspace members exclude `explain-py` for exactly this reason.

`oracle/compare.py` reports drift on every column: you forgot to rebuild the Python extension after a Rust change. Run `maturin develop --release -m crates/explain-py/Cargo.toml`.

`python3 main.py` says "missing: term_neonate.json": you're running it from the wrong directory. Run from `explain-user-python` itself, not from the repo root.

Activation script blocked on Windows: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.
