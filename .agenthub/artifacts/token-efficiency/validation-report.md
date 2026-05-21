# AgentHub Token Efficiency Lite — Claude Code Runtime Validation Report

**Date:** 2026-05-21
**Validator:** Claude Code (claude-sonnet-4-6)
**Branch:** claude/agenhub-token-efficiency-validation-w6tiv

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

## 3. Raw vs Compact Observations

### `git status`
| Mode | Output | Token cost |
|------|--------|------------|
| Raw (`git status`) | Multi-line: branch name, tracking info, "nothing to commit, working tree clean" | ~15 tokens |
| Compact (`git status --short`) | Empty (clean tree) — zero output | ~1 token |

**Observation:** For a clean working tree, `--short` produces zero output — perfectly compact and unambiguous.
For dirty trees, `--short` uses 2-char status codes (`M `, `??`, etc.) which are well-understood and much more compact.

### `git diff --stat` vs `git diff --name-only`
| Mode | Output (last commit) | Token cost |
|------|----------------------|------------|
| `--name-only` | 2 file paths, ~60 chars | ~15 tokens |
| `--stat` | 2-line table with +/- counts | ~50 tokens |
| Full diff | Full unified diff | hundreds of tokens |

**Observation:** For routine change inspection, `--name-only` is sufficient. `--stat` adds change-volume signal at modest cost.

### `git diff --stat HEAD~5..HEAD`
- 101 files changed, 11350 insertions(+), 712 deletions(-)
- Raw stat: ~120 lines / ~600 tokens
- Compact (`--name-only`): 101 lines / ~200 tokens — still long, but actionable

### Lint command (`bun run lint`)
- `oxlint` not installed in this environment — exit 127
- Short error line was self-describing without compression

### Tests (`bun test`)
- `error: preload not found "@opentui/solid/preload"` — short, unambiguous, no compression needed

---

## 4. Debug Quality

### What compact output preserves
- File change lists (`--name-only`) — full fidelity for navigation
- Status codes (`--short`) — sufficient for triage
- Exit codes + first error line — adequate for most failures

### What compact output loses
- Line-level diff context (which lines changed)
- Error stack traces (truncation loses frame chain)
- Structured data semantics (JSON, YAML, Terraform plans)
- Full lint/type-check output when multiple simultaneous errors exist

---

## 5. Commands Safe to Compact

| Command | Compact form | Rationale |
|---------|-------------|-----------|
| `git status` | `git status --short` | 2-char codes preserve all state info |
| `git diff` (routine) | `git diff --name-only` or `--stat` | File list sufficient for orientation |
| `git log` | `git log --oneline -N` | One line per commit is readable |
| `git branch` | `git branch --list` | Already compact |
| `ls -la` | `ls -1` | Directory listing rarely needs metadata |
| Build success | Exit 0 + last summary line | Success is binary |
| Test pass count | Summary line only | Pass/fail counts sufficient |

---

## 6. Commands Unsafe to Compact

| Command | Reason |
|---------|--------|
| Full `git diff` | Line context required for code review / understanding changes |
| Stack traces | Frame chain is diagnostic — truncation loses root cause |
| `terraform plan` | Every line is load-bearing; surprises hide in the diff |
| JSON/YAML configs | Structured data must be read in full; partial reads cause misinterpretation |
| Type-check errors (`tsc`) | Multiple errors with file:line:col — each line is a separate actionable item |
| Security scan output | Every finding is potentially critical |
| Full lint output (first pass) | All errors must be visible to fix them in one round |
| DB migration plans | Schema changes are irreversible; full output required |

---

## 7. Recommendation

**`ACCEPT_WITH_CAVEAT`**

The token efficiency policy is sound and well-targeted for Claude Code's usage patterns. The distinction between
"routine inspection" (compact) and "debugging/review" (raw) maps cleanly onto how Claude Code actually uses
shell output. The policy correctly identifies high-risk categories (stack traces, diffs, plans, JSON) for raw preservation.

### Why not ACCEPT_FOR_PROJECT_SCOPED_USE
`rtk` binary is not present in this environment. The RTK explicit wrapper aspect of the validation could not
be tested. The policy's compact output benefits are achievable with standard git flags (`--short`, `--stat`,
`--name-only`, `--oneline`) without RTK, but RTK's role as an explicit wrapper layer is unvalidated here.

---

## 8. Caveats

1. **RTK not available.** `rtk` is not in PATH and no temporary binary was provided. RTK-mediated compression
   was not exercised. All observations are based on native git flag equivalents.

2. **Lint/typecheck tools not installed.** `oxlint` is absent from this container. The lint comparison
   could not be completed. Policy should be re-validated in a full-toolchain environment.

3. **No dirty working tree.** The working tree was clean throughout. The most important use case — compact
   output during active development with many modified files — was not exercised live.

4. **Compact != Truncated.** The policy must distinguish:
   - *Compact by design* (e.g., `--short`, `--oneline`) — information-preserving
   - *Truncated* (e.g., `head -20`) — information-destroying
   Only the former is acceptable for routine use.

5. **Per-command policy, not global.** A global shell hook compressing all output would be unsafe.
   The explicit prohibition on `rtk init --global` is correct and should be enforced structurally.

6. **Fallback path must be zero-friction.** The debug/raw fallback must require no configuration change
   mid-session. An explicit `--raw` flag or env var is preferable to editing hook configs under pressure.

---

## Raw Log References

- `git status` raw: `On branch claude/agenhub-token-efficiency-validation-w6tiv\nnothing to commit, working tree clean`
- `git diff --name-only HEAD~1`: `footer.prompt.tsx`, `stream.transport.ts`
- `git diff --stat HEAD~5..HEAD`: 101 files, 11350 insertions, 712 deletions
- lint: `bun run lint` → exit 127, `oxlint: command not found`
- test: `bun test` → `error: preload not found "@opentui/solid/preload"`
