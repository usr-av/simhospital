# AV Fork — Incremental Work Log

Fork: `usr-av/simhospital` (upstream: `google/simhospital`)

---

## What was added / changed

### 1. Go module support (`go.mod`, `go.sum`)

The upstream repo is Bazel-only. Running `go run ./cmd/simulator/...` fails because there is no `go.mod`.

Added `go.mod` (module `github.com/google/simhospital`) and generated `go.sum` via `go mod tidy`. This lets you build and run the simulator with a plain Go toolchain — **no Bazel required**.

**Why Bazel is broken on modern toolchains:**
- Bazel 5.4.1 (the version the Makefile/CI targets) crashes on Java 26 (`jvm.out` is empty — JVM startup failure).
- Bazel 9 (latest via `bazelisk`) cannot resolve `@io_bazel_rules_go` because the repo uses the legacy `WORKSPACE` format, not Bzlmod.

**Go run workaround:**
```bash
go run ./cmd/simulator/... \
  --pathways_dir=configs/pathways \
  --pathway_names=<your_pathway> \
  --pathway_manager_type=deterministic \
  --max_pathways=1 \
  --sleep_for=100ms \
  --dashboard_address=:8001 \
  --output=stdout
```

> `--dashboard_address=:8001` avoids conflict if port 8000 is occupied by another process.

---

### 2. Example A01 → A34 pathway (`configs/pathways/my_pathways.yml`)

Added `admit_then_merge` pathway that demonstrates a two-patient admission + merge sequence:

| Step | Event | HL7 message |
|------|-------|-------------|
| 1 | `temp_patient` admitted to Renal ward | `ADT^A01` |
| 2 | `main_patient` merged with `temp_patient` | `ADT^A34` |
| 3 | Discharge | `ADT^A03` |

**Note on A34 vs A40:**
The pathway produces `A34` (merge within same account/encounter), not `A40` (merge across different accounts). To get `A40`, the two patients need distinct prior encounters before the merge step. This pathway does not set that up.

**Run it:**
```bash
go run ./cmd/simulator/... \
  --pathways_dir=configs/pathways \
  --pathway_names=admit_then_merge \
  --pathway_manager_type=deterministic \
  --max_pathways=1 \
  --sleep_for=100ms \
  --dashboard_address=:8001 \
  --output=stdout
```

**Pathway YAML (`configs/pathways/my_pathways.yml`):**
```yaml
admit_then_merge:
  percentage_of_patients: 1
  persons:
    main_patient:
      first_name: "John"
      surname: "Doe"
    temp_patient:
      first_name: "Unknown"
      surname: "Doe"
  pathway:
    - use_patient:
        patient: temp_patient
    - admission:
        loc: Renal
    - delay:
        from: 0s
        to: 0s
    - use_patient:
        patient: main_patient
    - merge:
        parent: main_patient
        children: [temp_patient]
    - delay:
        from: 0s
        to: 0s
    - discharge: {}
```

---

### 3. README update

Added **"Install & run from source (macOS/Linux)"** section to `README.md` covering:
- Bazelisk install via Homebrew
- `bazelisk` run command (for users with a working Bazel/Java combo)
- Note that the repo has no `go.mod` by default

---

## Simulator flags — key ones for local dev

| Flag | Purpose | Default |
|------|---------|---------|
| `--pathway_manager_type=deterministic` | Run pathways in fixed order (vs probabilistic) | `distribution` |
| `--max_pathways=N` | Stop after N pathways (process exits cleanly) | `-1` (run forever) |
| `--sleep_for=100ms` | Poll interval for due events | `1s` |
| `--dashboard_address=:PORT` | HTTP dashboard port | `:8000` |
| `--output=stdout\|file\|mllp` | Where HL7 messages go | `stdout` |
| `--output_file=PATH` | File path when `--output=file` | `messages.out` |

---

## Known issues / gotchas

| Issue | Detail |
|-------|--------|
| Port conflict on repeated runs | A stale simulator process holds `:8000` and `:9095`; new run fails immediately (metrics server can't bind → errgroup cancels context → 0 messages). Use `lsof -i :9095` to detect; `kill <PID>` to clear. |
| `Person` struct fields | YAML key is `surname`, not `last_name`. Other valid fields: `first_name`, `age`, `date_of_birth`, `gender`, `address`, `nhs`, `mrn`. |
| Merge produces A34, not A40 | See note above. |
| Bazel + Java 26 | Bazel 5.x crashes silently. Use the `go run` approach instead. |
