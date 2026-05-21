# AgentHub Token Efficiency Lite — Claude Code Runtime Validation Report

**Date:** 2026-05-21
**Validator:** Claude Code (claude-sonnet-4-6)
**Branch:** claude/agenhub-token-efficiency-validation-w6tiv
**Re-validation:** rtk installed globally (v4.2.0), tested, uninstalled

---

## 1. Runtime

- **Runtime:** Claude Code (remote execution environment, cloud-hosted container)
- **Model:** claude-sonnet-4-6
- **Shell:** bash (via Bash tool)
- **Platform:** Linux 6.18.5
- **Bun version:** 1.3.11 / 1.3.14 (project packageManager)
- **Note:** Ephemeral container — no persistent state beyond git commits

---

## 2. Target Project

- **Path:** `/home/user/opencode`
- **Repo:** ch010060/opencode
- **Branch:** `claude/agenhub-token-efficiency-validation-w6tiv`
- **Git status:** Clean working tree, nothing to commit
- **Project type:** Bun monorepo with ~18 packages (app, opencode, ui, desktop, etc.)
- **Primary language:** TypeScript/TSX

---

## 3. RTK Binary — Definitive Identification

`rtk` v4.2.0 was installed globally via `bun install -g rtk` and fully characterized.

**Identity:** `rtk` (npm) is the "Release The Kraken" release management CLI by Cliffano Subagio
(https://github.com/cliffano/rtk). It is a version-bumping and release orchestration tool,
analogous to `release-it` or `standard-version`.

**What it does:**
- Reads a `.rtk.json` config referencing version-carrying files (JSON, YAML, TOML, HCL, Makefile, text)
- Bumps semver version fields in those files (pre-release → release → next pre-release)
- Commits the changes and tags the git repo
- Supports `--dry-run` mode (no file writes, no git ops)

**What it does NOT do:**
- It has no `run`, `exec`, `wrap`, `compact`, or `shell` commands
- It has no capability to intercept or reformat shell command output
- It cannot act as a shell output compressor or token efficiency wrapper
- It has no awareness of Claude Code or agent runtimes

**Dry-run output observed (correct config, `--dry-run release`):**
```
dry run Executing pre step of rtk release scheme...
  * dry run Setting release version 0.2.0 on json resource at /tmp/dummy-package.json
  * dry run Committing release version changes made to /tmp/dummy-package.json...
dry run Executing release step of rtk release scheme...
  * dry run Adding release version tag 0.2.0 ...
dry run Executing post step of rtk release scheme...
  * dry run Setting next pre-release version 0.2.1-pre.0 on json resource at /tmp/dummy-package.json
  * dry run Committing next pre-release version changes made to /tmp/dummy-package.json...
```

This output is human-readable release audit logging, not compact shell wrapping.

**Conclusion:** The `rtk` npm package is a **different tool** from the "RTK explicit wrapper"
described in the validation prompt. The shell output compressor referenced in the prompt does not
correspond to any publicly published binary. The token efficiency policy's RTK layer cannot be
validated against this package.

---

## 4. Raw vs Compact Observations (native git flags — unchanged from prior run)

### `git status`
| Mode | Output | Token cost |
|------|--------|------------|
| Raw (`git status`) | Multi-line prose: branch name, tracking info, clean message | ~15 tokens |
| Compact (`git status --short`) | Empty for clean tree — zero output | ~1 token |

### `git diff --stat` vs `git diff --name-only`
| Mode | Output (last commit, 2 files) | Token cost |
|------|-------------------------------|------------|
| `--name-only` | 2 file paths, ~60 chars | ~15 tokens |
| `--stat` | 2-line table with +/- counts | ~50 tokens |
| Full diff | Full unified diff | hundreds of tokens |

### `git diff --stat HEAD~5..HEAD`
- 101 files, 11350 insertions, 712 deletions
- Raw stat: ~120 lines / ~600 tokens
- Compact (`--name-only`): 101 lines / ~200 tokens

### Lint (`bun run lint`)
- `oxlint` not installed — exit 127, one-line error, self-describing without compression

### Tests (`bun test`)
- `error: preload not found "@opentui/solid/preload"` — short, unambiguous

---

## 5. Debug Quality

### What compact output preserves
- File change lists (`--name-only`) — full fidelity for navigation
- Status codes (`--short`) — sufficient for triage
- Exit codes + first error line — adequate for most failures

### What compact output loses
- Line-level diff context
- Error stack traces (frame chain is the diagnostic)
- Structured data semantics (JSON, YAML, Terraform plans)
- Multi-error lint/type-check output

---

## 6. Commands Safe to Compact

| Command | Compact form | Rationale |
|---------|-------------|-----------|
| `git status` | `git status --short` | 2-char codes preserve all state |
| `git diff` (routine) | `git diff --name-only` or `--stat` | File list sufficient for orientation |
| `git log` | `git log --oneline -N` | One line per commit is readable |
| `git branch` | `git branch --list` | Already compact |
| `ls -la` | `ls -1` | Metadata rarely needed |
| Build success | Exit 0 + last summary line | Success is binary |
| Test pass count | Summary line only | Pass/fail counts sufficient |

---

## 7. Commands Unsafe to Compact

| Command | Reason |
|---------|--------|
| Full `git diff` | Line context required for review |
| Stack traces | Truncation loses root cause |
| `terraform plan` | Every line is load-bearing |
| JSON/YAML configs | Partial reads cause misinterpretation |
| Type-check errors (`tsc`) | Each file:line:col is a separate action item |
| Security scan output | Every finding is potentially critical |
| Full lint output (first pass) | All errors must be visible to fix in one round |
| DB migration plans | Irreversible schema changes |

---

## 8. Recommendation

**`ACCEPT_WITH_CAVEAT`** — unchanged from prior run

The token efficiency policy's "compact for inspection, raw for debugging" framework is sound
and directly applicable to Claude Code. The safe/unsafe classifications hold.

The RTK wrapper layer is **definitively unvalidatable**: the `rtk` npm binary (v4.2.0) is a
release management tool with no shell interception capability. The policy's RTK dependency
references a tool that either does not exist publicly, has a different package name, or is
an internal/future artifact not yet available.

---

## 9. Caveats

1. **RTK identity mismatch (definitive).** The npm `rtk` package v4.2.0 is "Release The Kraken",
   a versioning/release tool. It is not a shell output compressor. The RTK wrapper layer in the
   policy prompt is unvalidatable via any publicly available binary.

2. **Lint/typecheck tools not installed.** `oxlint` is absent. Lint comparison incomplete.

3. **No dirty working tree exercised live.** All observations based on git range queries.

4. **Compact != Truncated.** Policy must use information-preserving flags (`--short`, `--oneline`),
   not destructive truncation (`head -N`).

5. **Per-command scope only.** Global hook prohibition (`rtk init --global`) is correct and
   should be enforced structurally, not by convention.

6. **Fallback must be zero-friction.** Raw debug output should require no config edits mid-session.

---

## 10. Raw Log References

- `git status` raw: `On branch claude/agenhub-token-efficiency-validation-w6tiv\nnothing to commit, working tree clean`
- `git diff --name-only HEAD~1`: `footer.prompt.tsx`, `stream.transport.ts`
- `git diff --stat HEAD~5..HEAD`: 101 files, 11350 insertions, 712 deletions
- `bun run lint`: exit 127, `oxlint: command not found`
- `bun test`: `error: preload not found "@opentui/solid/preload"`
- `rtk --version`: `4.2.0`
- `rtk --dry-run release` (with valid config): release audit log, no shell wrapping capability
- `rtk` source: `github.com/cliffano/rtk`, release management only, no `exec`/`wrap`/`compact` commands
