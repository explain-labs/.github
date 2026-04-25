# explain-labs/.github

This repository serves two purposes for the Explain Labs organization:

1. **Org landing page.** `profile/README.md` is rendered by GitHub on the [explain-labs organization homepage](https://github.com/explain-labs).
2. **Project-wide documentation.** `docs/` holds documentation that applies across all three Explain implementations, rather than belonging to any single one.

## Layout

```
.github/
├── profile/
│   └── README.md              ← rendered on the org homepage
└── docs/
    ├── DEVELOPMENT_PIPELINE.md  ← why the project has three implementations
    ├── PORTING.md               ← how to port a new model JS → Python → Rust
    └── REQUIREMENTS.md          ← per-implementation, per-OS install requirements
```

## Implementation repos

The actual engine code lives in three separate repositories:

- [`explain-user-js`](https://github.com/explain-labs/explain-user-js) — JavaScript / Vue 3 (canonical, interactive)
- [`explain-user-python`](https://github.com/explain-labs/explain-user-python) — Python (research, papers, ML)
- [`explain-user-rs`](https://github.com/explain-labs/explain-user-rs) — Rust (long-running simulations, sweeps)

See the [docs](docs/) for how the three fit together and how to contribute changes that span them.
