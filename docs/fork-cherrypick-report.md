# Fork-ahead cherry-pick report (vs `google/simhospital`)

This repo is a local clone of Simulated Hospital with:

- `origin`: `usr-av/simhospital`
- `upstream`: `google/simhospital`

Current finding: **`usr-av/simhospital` is not ahead of `google/simhospital`** (no additional commits on `origin/master` vs `upstream/master`).

## Forks ahead of `google/simhospital`

Scan scope: top recently-updated / starred forks (to keep API calls reasonable).

- **`dallen-statrad/simhospital`**: ahead by **74**
- **`karthiknukala/simhospital`**: ahead by **4**
- **`webamboos/simhospital`**: ahead by **3**
- **`Health-Tech-Innovators/simhospital`**: ahead by **2**
- **`DinhLucent/simhospital`**: ahead by **1**

## Recommended cherry-picks (capability-focused)

### 1) Add a basic “clinical benchmark / validation” hook

Fork: `DinhLucent/simhospital` (ahead by 1)

- **Why it advances capability**: introduces a small “benchmark/validator” surface that can be used for regression checks on generated output (useful before/after pathway/config changes).
- **Risk**: low-to-medium (new package + doc file; should not affect simulator runtime unless wired in).

Cherry-pick candidate:

- `36a5f2e38aeba96afd17950262d1c2f28f2e61c7` — “feat: add clinical benchmark framework and project intelligence”
  - touches: `pkg/benchmarks/validator.go`, `PROJECT_INTELLIGENCE.md`

### 2) Improve generated content realism (order profile / vitals distinctions)

Fork: `karthiknukala/simhospital` (ahead by 4)

- **Why it advances capability**: small config-level enhancements can improve message realism without major code changes.
- **Risk**: low (config-only + build convenience).

Cherry-pick candidates:

- `bc2be85f12ffa01cfc724e6db04b57965f12551d` — “generate resp rate value”
  - touches: `configs/hl7_messages/order_profiles.yml`
- `fd7529da1e69e356850afb6f3f63f82be6b0fa44` — “distinguish test from vitals”
  - touches: `configs/pathways/pathways.yml`
- `633e2229e8af5daaac3728cf387839531b396c02` — “test case”
  - touches: `configs/pathways/pathways.yml`

Optional / convenience only:

- `f235a76ae699232233945a673458623e720d2927` — “Makefile”

### 3) “Make it build again” type fixes (helps adoption, not core functionality)

Fork: `webamboos/simhospital` (ahead by 3)

- **Why it advances capability**: improves developer experience by reducing build friction (especially in modern toolchains/containers).
- **Risk**: low, but still needs local validation.

Cherry-pick candidates:

- `3cde88daa137b828c297102edbe9124561fc0d16` — “fix: add missing dep”
  - touches: `WORKSPACE`
- `21ddcb5ca9aa62eaf9e4669d40c16a66cfa7a01a` — “fix build”
  - touches: `.bazelversion`, `.devcontainer/devcontainer.json`, `.gitignore`

Doc-only:

- `52ffaa07bc091e6dd6c11311477ca48ac286d8ad` — “chore: update readme”

## Not recommended to cherry-pick (as-is)

### Large Bazel migration / Bzlmod stubs without a clean narrative

Fork: `dallen-statrad/simhospital` (ahead by 74)

The sampled commits are overwhelmingly Bazel/Bzlmod migration attempts, “stub” scaffolding, and repository URL tweaks. This might be valuable *if* you explicitly want to migrate off `WORKSPACE`, but it’s **not a focused capability lift** and would likely be time-consuming to reconcile.

Examples (not exhaustive):

- `4cc37ab4fe889a10c0cbf23795e58caa8666bd86` — “Create MODULE.bazel”
- `4dfd5efec734012683eccaa11c01848f758f8577` — “Remove WORKSPACE (migrate to Bzlmod) …”
- many “stub / stubs / more” commits with unclear intent and broad build-system surface area

### Very large “add everything” build-file drop

Fork: `Health-Tech-Innovators/simhospital` (ahead by 2)

One commit adds a huge volume of Bazel build files and generated `bazel-*` outputs. This is hard to review/cherry-pick safely and is mostly build-system expansion rather than simulator capability.

- `dd1c16ad3e47709392aca1f7d273f2ee459ccc41` — “Add Bazel configuration and build files …”
- `1aa89ab86a08c835e0d205e67dad9f752a49a216` — includes `go.mod`/`go.sum` and output artifacts; likely not intended for upstreaming

## How to apply (suggested workflow)

1) Add the fork remote(s) you intend to cherry-pick from (example for `DinhLucent`):

```bash
git remote add dinhlucent https://github.com/DinhLucent/simhospital
git fetch dinhlucent
```

2) Cherry-pick the commit(s):

```bash
git cherry-pick 36a5f2e38aeba96afd17950262d1c2f28f2e61c7
```

3) Repeat for other forks/commits you want.

## Local note (your working tree)

Your local working tree currently contains uncommitted/untracked build artifacts and module files (e.g. `MODULE.bazel`, `MODULE.bazel.lock`, `go.mod`, `go.sum`, `bazel-*` dirs). It’s best to clean/commit/ignore these before attempting cherry-picks so conflicts are easier to reason about.

