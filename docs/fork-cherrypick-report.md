# Fork cherry-pick report (vs `google/simhospital`)

- `origin`: `usr-av/simhospital`
- `upstream`: `google/simhospital`
- Last full scan: **2026-04-25**

---

## How to resync (run this whenever upstream gets new commits)

```bash
# 1. Fetch latest upstream
git fetch upstream

# 2. Check if upstream moved
git log master..upstream/master --oneline

# 3. Rebase our work on top of upstream (preferred over merge)
git rebase upstream/master

# 4. Re-scan forks for new commits ahead of the new upstream HEAD
gh api repos/google/simhospital/forks --paginate \
  --jq '.[] | .full_name' | while read fork; do
    result=$(gh api "repos/$fork/compare/google:master...${fork/:master/}:master" \
      --jq '{ahead: .ahead_by}' 2>/dev/null)
    ahead=$(echo "$result" | jq -r '.ahead')
    [ "$ahead" -gt 0 ] 2>/dev/null && echo "$ahead  $fork"
done | sort -rn

# 5. For each fork worth examining:
git remote add <alias> https://github.com/<fork>
git fetch <alias> master --depth=10
git log master..<alias>/master --oneline   # see what's new
git show <sha> --stat                       # inspect a commit
git cherry-pick <sha>                       # apply it
```

---

## Full fork scan results (2026-04-25)

101 forks total. 13 ahead of upstream. 4 private/404.

| Fork | Ahead | Theme | Action |
|------|-------|-------|--------|
| `dallen-statrad` | 74 | Bazel → Bzlmod migration (no functional changes) | Skip |
| `jordan83` | 29 | GCP bucket pathway loading, PD1 fixes, cloudbuild | Partial — see log |
| `Arend-melissant` | 12 | Azure CosmosDB persistence backend | Skip (too niche, messy commits) |
| `karthiknukala` | 4 | Resp rate config, vitals/test separation, demo pathway | **Applied** |
| `alonbg` | 3 | JSON tags on HL7 schema structs | **Applied** |
| `webamboos` | 3 | Bazel build fixes, README | Skip (Bazel-specific) |
| `qining` | 3 | JSON/base64 file output, busy-wait runner "fix" | Skip — see notes |
| `Health-Tech-Innovators` | 2 | Bulk Bazel build files, bulk ADT pathway | Skip (large blast radius) |
| `henrydwright` | 2 | Local build fixes (Bazel) | Skip |
| `chrisdutz` | 2 | Full Bazel → go build migration (100+ files touched) | Skip (we did this more cleanly) |
| `DinhLucent` | 1 | Benchmark stub + doc file | Skip (low value) |
| `bitcrshr` | 1 | Full Bazel removal + go modules | Skip (we did this more cleanly) |
| `hyhossein` | 1 | `.DS_Store` commit | Skip (noise) |

---

## Applied cherry-picks

### 2026-04-25

| Commit | Fork | Message | Files | Notes |
|--------|------|---------|-------|-------|
| `633e222` | `karthiknukala` | test case | `configs/pathways/pathways.yml` | Adds `demo_pathway`: admission + vitals autogen + discharge |
| `bc2be85` | `karthiknukala` | generate resp rate value | `configs/hl7_messages/order_profiles.yml` | Adds `value: 1` + `value_type: NM` to MDC_RESP_RATE; without this, resp rate produces no numeric value |
| `fd7529d` | `karthiknukala` | distinguish test from vitals | `configs/pathways/pathways.yml` | Adds `trigger_event: R32` to the lab result step in demo_pathway, separating it from vitals |
| `a0f7f9a` | `alonbg` | add json tags and omitempty | `pkg/hl7/schema.go` | Adds `json:"...,omitempty"` tags to all 1329 HL7 struct fields; enables clean JSON serialization of parsed HL7 |

### Skipped with notes

| Commit | Fork | Reason |
|--------|------|--------|
| `03b3b91` | `jordan83` | "Remove PD1 from A08" — already not present in our A08 (conflict resolved as empty, skipped) |
| `d7a53ee` | `jordan83` | "Remove PD1 from A31" — initial file commit, conflicts completely |
| `d5df48d` | `jordan83` | "GCP bucket pathway loading" — adds `pkg/files/files.go` + WORKSPACE Bazel deps; worth revisiting if GCP pathway loading is needed |
| `aef62dc` | `qining` | "Do not wait for sleep" — replaces `time.After(sleepFor)` with `default` (busy-wait). 100% CPU usage; not safe for production |
| `cbb383a` | `qining` | "JSON/base64 file output" — changes `fileSender` to emit `{"data":"<base64>"}` instead of raw HL7; breaks existing file output format |

---

## Forks worth revisiting

### `jordan83/simhospital` — GCP pathway loading
`d5df48d` adds `pkg/files/files.go` (155 lines) and `pkg/pathway/parser.go` changes to load pathway configs from GCS buckets. The WORKSPACE change is Bazel-specific and not needed. The Go code changes are self-contained.

**To apply when needed:**
```bash
git remote add jordan83 https://github.com/jordan83/simhospital
git fetch jordan83 master
# Review the diff first
git show d5df48d -- pkg/files/files.go pkg/pathway/parser.go
# Cherry-pick without WORKSPACE changes
git cherry-pick d5df48d
git restore WORKSPACE   # discard WORKSPACE hunk
```

### `Arend-melissant/simhospital` — Azure CosmosDB persistence
12 commits implementing CosmosDB as a state backend. Messy commit history but the capability (pluggable persistence) is interesting. Requires significant cleanup before applying.

### `chrisdutz/simhospital` / `bitcrshr/simhospital` — Bazel removal
Both forks do full Bazel→go migration. We did a cleaner version (go.mod without ripping out BUILD.bazel files). If BUILD.bazel maintenance becomes a burden, revisit `chrisdutz/19618ab5`.

---

## Known pre-existing issues (not introduced by cherry-picks)

| Issue | Detail |
|-------|--------|
| `pkg/hl7/schemas/*` don't compile with `go build ./...` | Sub-packages in `pkg/hl7/schemas/210`, `220`, etc. declare `package hl7` but reference types from parent `pkg/hl7` package. This is a Bazel-only layout. Pre-existing; `go build ./cmd/...` compiles clean. |
| `pkg/message/messages.go` A31 still has PD1 | Jordan83's A31 fix was an initial-file-level commit and couldn't be cherry-picked cleanly. A31 still includes PD1 segment in our repo. Fix manually if needed: remove the `pd1` block in `BuildUpdatePersonADTA31`. |
