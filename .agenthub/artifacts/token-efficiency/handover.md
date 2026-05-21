# AgentHub Token Efficiency Lite — 任務交接清單

**任務名稱：** Claude Code Runtime Validation for RTK Token Efficiency  
**執行日期：** 2026-05-21  
**執行者：** Claude Code (claude-sonnet-4-6)  
**分支：** `claude/agenhub-token-efficiency-validation-w6tiv`  
**Repo：** ch010060/opencode  

---

## 1. 任務背景

驗證 Claude Code 作為 AI runtime 能否安全受益於 RTK 的 compact shell output 機制，同時保留 raw 輸出作為 debug fallback。

**驗證邊界（不可越界）：**
- 不啟用 RTK global hook（不執行 `rtk init -g`）
- 不修改 production code
- 不修改 runtime profiles
- RTK 僅作為 explicit wrapper 使用

---

## 2. 工具識別：重要陷阱記錄

### ⚠️ npm `rtk` 與官方 RTK 是兩個完全不同的工具

| 屬性 | npm `rtk` (錯誤) | rtk-ai/rtk (正確) |
|------|-----------------|------------------|
| 安裝指令 | `npm/bun install -g rtk` | `brew install rtk` 或 install script |
| 作者 | Cliffano Subagio | rtk-ai |
| 功能 | Semver 版本管理 (release toolkit) | CLI proxy，壓縮 shell output |
| GitHub | github.com/cliffano/rtk | github.com/rtk-ai/rtk |
| 與本任務相關 | ❌ 完全無關 | ✅ 正確工具 |

**教訓：** 任何文件或 CI script 中，凡提到安裝 RTK，必須明確指向 rtk-ai/rtk，**絕對不可使用 `npm install rtk` 或 `bun install rtk`。**

### 正確安裝方式

```bash
# macOS（推薦）
brew install rtk

# Linux / CI 容器（推薦固定版本）
RTK_VERSION=v0.40.0 curl -fsSL \
  https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh

# 驗證安裝
rtk --version  # 應輸出：rtk 0.40.0
```

**不要使用：**
```bash
# ❌ 這會裝到錯的工具 (cliffano/rtk)
npm install -g rtk
bun install -g rtk
```

---

## 3. RTK 工具簡介（rtk-ai/rtk v0.40.0）

- **本質：** Rust 單一執行檔，<10ms overhead
- **原理：** 攔截指令輸出，套用 filtering / grouping / truncation / deduplication，再回傳給 AI context
- **平台：** Linux x86_64 (`x86_64-unknown-linux-musl`)、macOS、ARM64
- **官網：** https://www.rtk-ai.app/
- **Release：** https://github.com/rtk-ai/rtk/releases

---

## 4. 驗證環境

| 項目 | 內容 |
|------|------|
| Runtime | Claude Code，遠端雲端容器 |
| 模型 | claude-sonnet-4-6 |
| OS | Linux 6.18.5 / x86_64 |
| Shell | bash |
| 目標 Repo | ch010060/opencode（Bun monorepo，18 個套件，TypeScript/TSX） |
| Git 狀態 | 驗證全程為 clean working tree |
| RTK 安裝路徑 | `~/.local/bin/rtk`（驗證後已刪除） |
| Bun 版本 | 1.3.11 |

---

## 5. 驗證結果：各指令比較

### 5.1 `git status`

```
# Raw
On branch claude/agenhub-token-efficiency-validation-w6tiv
Your branch is up to date with 'origin/...'
nothing to commit, working tree clean

# RTK
* claude/agenhub-token-efficiency-validation-w6tiv...origin/...
clean — nothing to commit
```

**節省：27.7%（13 tokens）**  
**資訊完整性：完整。** 語意等價，可讀性更高。

---

### 5.2 `git log`

```
# Raw (--oneline)
c347ef7 docs: update token efficiency report...

# RTK
c347ef7 docs: update token efficiency report with definitive rtk binary ident...
  Re-validated with rtk v4.2.0 installed globally...
  [+2 lines omitted]
```

**節省：17.6%（71 tokens）**  
**資訊完整性：完整。** 主旨截斷於 ~70 字元，附 body 摘要，比 `--oneline` 資訊量更多。

---

### 5.3 `git diff`（小範圍，1 個檔案）

**節省：9.8%（296 tokens）**  
**資訊完整性：完整 + 更多。** RTK 輸出 stat + inline condensed diff（比 `--stat` 資訊更豐富）。

---

### 5.4 `git diff`（大範圍，5 commits，101 個檔案）

| 模式 | Tokens | 節省 |
|------|--------|------|
| Raw `git diff HEAD~5` | ~63,700 | — |
| `rtk git diff HEAD~5` | ~11,500 | **82%（52,200 tokens）** |

**關鍵：** RTK 輸出末尾會附上 escape hatch 提示：
```
... (more changes truncated)
[full diff: rtk git diff --no-compact]
```
AI 看到這行就知道如何取得完整 diff，**零摩擦切換**。

---

### 5.5 `ls`

| 模式 | 節省 |
|------|------|
| `ls -la` (raw) | — |
| `rtk ls` | **72.6%（609 tokens）** |

RTK 輸出：目錄優先、檔名 + 人類可讀大小，**移除權限位元、inode、owner、時間戳**。  
⚠️ 需要權限或時間資訊時，改用 raw `ls -la`。

---

### 5.6 `json`（package.json）

| 模式 | 節省 |
|------|------|
| `rtk json package.json` | **42.3%（955 tokens）** |
| `rtk json --keys-only` | 更高（僅顯示 key 結構，值以型別代替） |

適合快速了解設定檔結構。需要實際值時使用預設模式；需要完整原始 JSON 時用 raw `cat`。

---

### 5.7 `tsc`（TypeScript 編譯錯誤）

`rtk tsc` 在此環境輸出了 **30 個錯誤，全部保留**，無截斷。  
**節省：~0%（正確行為）**——錯誤本身就是訊號，RTK 不壓縮 type errors。

```
github/index.ts(1,19): error TS2307: Cannot find module 'bun'...
github/index.ts(2,18): error TS2591: Cannot find name 'node:path'...
... (全 30 條錯誤均完整輸出)
```

---

### 5.8 整體 Session 統計（rtk gain）

```
Total commands:    8
Input tokens:      70.3K
Output tokens:     16.1K
Tokens saved:      54.2K  (77.1%)
Total exec time:   1.4s   (avg 170ms)
```

| 指令 | 次數 | 節省 tokens | 節省 % |
|------|------|------------|-------|
| rtk git diff HEAD~5 | 2 | 52,200 | 82.0% |
| rtk json | 2 | 955 | 42.3% |
| rtk ls | 1 | 609 | 72.6% |
| rtk git diff HEAD~1 | 1 | 296 | 9.8% |
| rtk git log | 1 | 71 | 17.6% |
| rtk git status | 1 | 13 | 27.7% |

---

## 6. 指令分類：可壓縮 vs 不可壓縮

### ✅ 可用 RTK 包裝的指令

| 指令 | RTK 形式 | 預期節省 | 備註 |
|------|---------|---------|------|
| git status | `rtk git status` | ~28% | 語意等價 |
| git log | `rtk git log` | ~18% | 附 body 摘要 |
| git diff（小） | `rtk git diff` | ~10% | stat + inline diff |
| git diff（大） | `rtk git diff` | ~82% | 附 `--no-compact` 提示 |
| ls | `rtk ls` | ~73% | 不含 metadata |
| JSON 檢視 | `rtk json` | ~42% | 可選 `--keys-only` |
| 測試執行 | `rtk test` / `rtk err` | 高 | 僅顯示 failures |
| TypeScript | `rtk tsc` | ~0% | 全錯誤保留，正確行為 |

### ❌ 不應使用 RTK 包裝的指令

| 指令 / 情境 | 原因 | 正確做法 |
|------------|------|---------|
| Full diff（code review） | RTK 截斷大段 hunk | `rtk git diff --no-compact` 或 raw `git diff` |
| Stack traces | Frame chain 是診斷關鍵，截斷即失去根因 | Raw command |
| `terraform plan` | 每一行 +/-/~ 都是 load-bearing | Raw only |
| 安全掃描輸出 | 每個 finding 都可能關鍵 | Raw only |
| DB migration plan | 不可逆操作，必須全覽 | Raw only |
| Conflict resolution diff | 需要所有 context lines | Raw `git diff --no-color` |
| `ls -la`（需要權限） | RTK 移除權限位元 | Raw `ls -la` |
| JSON configs（需要實際值） | `--keys-only` 會隱藏值 | `rtk json`（不加 `--keys-only`）或 raw `cat` |

---

## 7. Debug Fallback 協議

RTK 的 escape hatch 是 **`--no-compact` flag**，且會主動印在輸出末尾：

```bash
# RTK compact 模式（預設）
rtk git diff HEAD~5

# Raw fallback（完整 unified diff）
rtk git diff HEAD~5 --no-compact

# 完全繞過 RTK
git diff HEAD~5
```

**原則：** Escape hatch 是 zero-friction 的。看到 `[full diff: rtk git diff --no-compact]` 就直接執行，不需要修改任何設定。

---

## 8. Token Efficiency Policy（已驗證版本）

```
1. 例行性查看（status, log, ls, json 結構）→ 優先使用 rtk 包裝
2. Debug / 需要完整資訊 → 使用 --no-compact 或 raw command
3. 不啟用 global shell hook（不執行 rtk init -g），除非另行審核
4. Raw logs 保存至 .agenthub/artifacts/token-efficiency/
5. 以下類型不壓縮：stack traces、terraform plans、full diffs、JSON configs（實際值）
```

---

## 9. 最終建議

**`ACCEPT_FOR_PROJECT_SCOPED_USE`**

升級自第一輪的 `ACCEPT_WITH_CAVEAT`。RTK 工具身份已確認，真實數據驗證完成。

| 項目 | 結論 |
|------|------|
| Policy 框架 | ✅ 正確且實用 |
| RTK 節省效果 | ✅ 真實、可測量（session 77.1%） |
| 資訊完整性 | ✅ RTK 保留所有 actionable signal |
| Debug fallback | ✅ `--no-compact` 零摩擦，RTK 主動提示 |
| Global hook | ⛔ 未啟用，維持 explicit wrapper 模式 |

---

## 10. 尚未驗證的項目（後續需補）

| 項目 | 原因 | 建議做法 |
|------|------|---------|
| `rtk lint`（oxlint） | 此容器未安裝 oxlint | 在有完整 toolchain 的環境重跑 |
| Dirty working tree | 驗證全程 clean tree | 在有未提交變更的環境測試 `rtk git status` / `rtk git diff` |
| `rtk err bun test` | `@opentui/solid/preload` 缺失 | 修復 preload 後補測 |
| Global hook 行為 | 未啟用（邊界限制） | 需另開審核流程後再驗證 |

---

## 11. 產出物清單

| 檔案 | 路徑 | 說明 |
|------|------|------|
| 完整驗證報告 | `.agenthub/artifacts/token-efficiency/validation-report.md` | 技術細節、原始數據、rtk gain 輸出 |
| 本交接清單 | `.agenthub/artifacts/token-efficiency/handover.md` | 此文件 |
| Git branch | `claude/agenhub-token-efficiency-validation-w6tiv` | 所有 commits 在此分支 |

**Commits：**
```
69c31c5  docs: complete RTK v0.40.0 validation — upgrade to ACCEPT_FOR_PROJECT_SCOPED_USE
c347ef7  docs: update token efficiency report with definitive rtk binary identification
8d98b7e  feat: add AgentHub token efficiency validation report for Claude Code runtime
```

---

## 12. 接手者注意事項

1. **安裝時確認工具身份：** 裝完後執行 `rtk --help`，確認第一行出現 `A high-performance CLI proxy...`，而非 `Release the project`（後者是錯的 npm 套件）。

2. **不要全域啟用 hook：** `rtk init -g` 會修改 Claude Code 的 shell 前置設定，需另行審核後才能啟用。

3. **Escape hatch 是你的朋友：** 任何時候需要完整輸出，在 rtk 指令後加 `--no-compact` 即可，不需要重新執行原生指令。

4. **大 diff 效益最顯著：** 跨多個 commits 的 diff（`git diff HEAD~N`）是 RTK 節省最多的場景（82%）。小 diff 節省有限但仍正確。

5. **TypeScript 錯誤不會被截斷：** `rtk tsc` 保留所有 `file(line,col): error` 行，可放心使用。
