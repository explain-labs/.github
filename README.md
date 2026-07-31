# explain-labs/.github

This repository serves two purposes for the Explain Labs organization:

1. **Org landing page.** `profile/README.md` is rendered by GitHub on the [explain-labs organization homepage](https://github.com/explain-labs). It is the shared "how to use these repositories" page.
2. **Project-wide documentation.** `docs/` holds documentation that applies across repositories rather than belonging to any single one.

## Layout

```
.github/
├── profile/
│   └── README.md              ← rendered on the org homepage
└── docs/                      ← historical: the JS / Python / Rust pipeline
    ├── DEVELOPMENT_PIPELINE.md
    ├── PORTING.md
    └── REQUIREMENTS.md
```

## Active repositories

- [`explain-engine`](https://github.com/explain-labs/explain-engine) — the framework-agnostic simulation engine, mounted elsewhere as a git submodule
- [`explain-ui`](https://github.com/explain-labs/explain-ui) — the Vue 3 web app built on it

Archived, read-only predecessors: [`explain-user-js`](https://github.com/explain-labs/explain-user-js) and [`explain-user-python`](https://github.com/explain-labs/explain-user-python).

The files under [`docs/`](docs/) describe the earlier three-implementation pipeline and are kept for reference; they do not describe the current `explain-engine` + `explain-ui` layout.
