# Claude + Codex 协作工作流

Two-LLM 开发协作 protocol。Claude（主对话）+ Codex（CLI）+ Owner（oversight + 最终裁决）。

把这份文档放进项目 `docs/`，或在新会话开头粘给 Claude，它就能按这套规则跑。

---

## 前提

- **Claude Code CLI** 已就绪（主对话载体）
- **Codex CLI** 已装并登录。推荐 user-local 安装（agent 后续 no-sudo 自更新）：

  ```bash
  # 一次性 bootstrap（每台机器只做一次）
  npm config set prefix "$HOME/.local"
  npm install -g @openai/codex@latest
  export PATH="$HOME/.local/bin:$PATH"                       # 当前 shell 立刻生效
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.profile  # 之后的 shell；多数 Ubuntu 默认已做
  command -v codex   # 确认指向 ~/.local/bin/codex 而非系统旧版
  codex login        # ChatGPT 订阅或 API key 均可
  codex exec "hello" # 验通

  # 后续升级 — 任意 cron / supervisor 可调用，不需要 sudo
  npm install -g @openai/codex@latest
  echo hello | codex exec  # smoke
  ```

  **PATH 注意**：cron / systemd 不加载 `~/.profile`。被 cron 调用的脚本必须在顶部 `export PATH="$HOME/.local/bin:$PATH"`，否则会回退到 system-wide 的旧版 codex（如果有）。

  **自更新建议**：cron / supervisor 调一个 wrapper（伪代码）：
  ```bash
  cur=$(codex --version | awk '{print $2}')
  latest=$(npm view @openai/codex version)
  [ "$cur" = "$latest" ] && exit 0
  npm install -g @openai/codex@"$latest"
  echo hello | codex exec >/dev/null 2>&1 \
    || npm install -g @openai/codex@"$cur"   # smoke 失败回滚
  ```

- 一个 Git 仓库 + GitHub 仓（`gh` CLI 已装并 auth）+ CI（任何能在 PR 上跑测试的 CI 都行）
- Owner 同意把"merge gate"交给 Codex（可随时撤回）

---

## 角色契约

| Actor | 职责 |
|---|---|
| **Claude** | 主对话；实施代码；处理评审 finding；提 PR；Codex APPROVE 后 squash-merge |
| **Codex (CLI)** | 独立第二意见。仅在 Phase 2 (pre-PR review) 和 Phase 5 (final review/merge gate) 出现。或在 Mode 2 作并行分析者 |
| **Owner** | 任务边界 / 模式选择最终决定权；prod oversight；Phase 5 escalate threshold 命中后的升级目标（threshold 见下方 Mode 1 5-phase 块）；Mode 2 中无法收敛的 disagreement 的裁决人 |

---

## 两种 task 模式

Claude 在任务入口**显式声明走哪种**，Owner 可以一句话否决重选。

| Trigger（用户原话信号） | 模式 |
|---|---|
| "修 / 实施 / 加 / refactor / fix / write" | **Mode 1 — Linear** |
| "分析 / 设计 / 评审 / audit / X 是不是 OK / X 还是 Y" | **Mode 2 — Parallel** |
| 模糊 | Claude 默认 Mode 1，注明可切 |

---

## Mode 1 — Linear (5-phase, dev/fix)

```
Phase 1   Claude 实施 + 写测试 + 跑本地测试套件
Phase 2   Codex pre-PR review (codex exec + git diff against base branch)
          → 输出 finding 表 + PASS / NEEDS_FIXES / BLOCK
Phase 3   Claude 处理 finding（accept / reject / defer），按需改代码
          → 本地准备好 PR body（finding accept/reject 表）— 还不开 PR
Phase 3.5 Claude push 前自查 4 项（详见 §Phase 3 收尾 self-check）
          — 任何一项命中就回 Phase 3 改完再走 3.5
Phase 4   Claude push + 用 `gh pr create --draft` 开 draft PR
          + 等 CI 绿 (gh run watch <id> --exit-status)
Phase 5   Codex final review (codex exec, 单 token APPROVE/REJECT
          独立成行 — 后面可跟 1 行 reason 但不参与 gating 解析)
          APPROVE → Claude 标 ready + squash-merge + delete-branch
                    CI 自动部署（如有 auto-deploy 流水线）
          REJECT  → 回 Phase 3。escalate 阈值是 finding-cost-aware：
                    - 第 1-2 轮 REJECT：自决继续
                    - 第 3 轮 REJECT：评估 fix 工作量：
                      * < 10 min fix（多为 hygiene / log-level / typo）：
                        自决继续到第 4 轮
                      * ≥ 10 min fix 或涉及设计权衡 / 风险评估：
                        escalate Owner
                    - 任何第 5 轮 REJECT：强制 escalate（保留硬上限）
```

**硬规则**

- Phase 5 auto-merge 仅因为 Owner 显式授权才生效；Owner 任何时候可以撤回。
- Merge 后**强制观察窗**：Owner 看监控（Sentry / Datadog / 自建告警），有问题立刻回滚。建议 30 分钟。
- **prod-critical path 黑名单**（即使 Codex APPROVE，Claude 也必须 surface 给 Owner 二次确认）。每个项目自己定，常见类目：
  - 风控 / 限额 / 安全相关代码
  - secret / 配置 / 环境变量加载逻辑
  - 涉及金钱 / 资产移动的核心调用
  - 数据库 schema migration
  - CI / 部署脚本 / 回滚脚本
- **Phase 2 vs Phase 5 prompt 必须不同**。Phase 2 抓 detail；Phase 5 看 PR coherence + regression + 项目 hygiene + 输出单 token verdict（独立成行可机读 — 后面那行 reason 仅给 human reader 看，不参与 auto-merge gating）。重复就是浪费 token。

### Phase 3 收尾 self-check（push 前必走，~10-15 min）

Phase 3 改完代码到 Phase 4 push 之间是规则空白。下面 4 条自检能把 Phase 5 round 数从典型 3-4 压回 1-2，ROI 高。**这 4 条不能替代 Codex**（独立判断价值还在），目标是降低 round 数 / wall time。

1. **Finding 逐条复述自验** — 每条 Phase 2 finding 都列出"本次改动满足 finding 哪几个具体行为契约"，逐字对照 finding 原文。"accept" ≠ "修透"。
   - **典型坑**：finding 原文写 "Append each confirmed id immediately after it is deemed canceled, **before optional logging/enrichment or more loop work**." Phase 3 改完把 append 移到了 logging/enrichment **之后**，accept 时没复述 "before optional logging/enrichment" 这个限定，Phase 5 R1 直接 reject。逐字对照能 catch。
2. **Hygiene grep** — 在 commit 前跑（命令对 worktree 工作树扫描，包括 unstaged + staged 改动）：
   ```bash
   git diff "$(git merge-base origin/main HEAD)" -- <src dirs> | grep -nE "^\+.*\bpass\b|^\+.*logger\.debug.*exc"
   ```
   命中即修。**注意**：用 `merge-base` 而不是 `origin/main...HEAD`——后者只覆盖已 commit 的 HEAD，看不到你刚改完还没 commit 的代码（也就是你最该自检的那部分）。grep 模式按项目自己的 hygiene 规则补 — 上面那两个匹配 "swallow exception" 和 "logger.debug 吞错"，是典型的 fail-silent 反模式。
   - **典型坑**：新写 helper 用 `except OSError: pass` 三处，Phase 5 R2 直接 reject。一个 grep 命令就能 catch。
3. **Semantic uplift audit** — 列出 diff 里所有"新加的 `if old_var: ...`" 或 "`state[k] = something_load_bearing`"，**回看 old_var / state[k] 之前是怎么 set 的**，相关 except 路径 / log level 是否仍合适。pre-existing 的 `logger.debug` 在新语义下可能变 load-bearing 但被吞掉。
   - **典型坑**：加了 `refresh_ok_by_resource[key] = False` 让 pre-existing `logger.debug("refresh failed ...")` 路径变成新 GC 逻辑的 input；改完没回头看 log level，Phase 5 R3 reject — 如果 refresh 失败被 debug 吞掉，GC 决定基于错误状态。
4. **Mini Phase 5 self-review** — 拿 Phase 5 prompt 对自己 diff 走一遍 mental review，特别看 "我自己看着也别扭" 的位置：variable naming 像不像 production 风格？新加的 try/except 是否能解释 "为什么这里需要 catch"？docstring 是否反映了真实行为？

**ROI 估算**：~15 min self-check 对应可省 ~25 min Codex round（每轮 ~10 min review + ~5 min push/CI/wait）。把 4 轮压到 2 轮就是净 ~25 min/PR。

---

## Mode 2 — Parallel (analyst + multi-round challenge, design/audit)

```
Step 1  Claude 和 Codex 各自独立分析（Codex 用 codex exec 跑 background；
        Claude 在 foreground 自己看代码）
Step 2  双向 cross-challenge：每方 review 对方的 finding，每条标
        AGREE / AGREE_BUT_RESEVERITY:Pn / DISAGREE + 一句理由
Step 3  合并：保留双方共识 + 主动让步 + drop 双方都同意的误报；
        无法收敛的 disagreement 升级 Owner 裁决
        记下双方各自 drop 了几条 — 是 cross-check 真实有效的质量信号
```

**约束**

- Round 上限 **2 轮 challenge**。再多 = 边际收益骤降 + token 浪费。
- 双方 prompt 必须 self-contained — Codex 没有 conversation history，必须从 prompt 读项目背景（关键文档路径 / 规则 / 输出格式）。
- Mode 2 不出 PR；它是分析阶段。如要修，进入 Mode 1。
- 不要让 Codex 当自己 finding 的"裁判"。给它的 challenge prompt 必须独立，不要附带它原本的 finding。

---

## 模式切换

任务可能从 Mode 2 → Mode 1（评审完决定改）。Claude 在切换时显式说明并重新走 Mode 1 五个 phase。

不应反向（Mode 1 中途切 Mode 2）— 已经在写代码了说明已经知道要做什么；Mode 2 是探索期工具。

---

## 并行多 PR pipeline（同 session 多任务加速）

当一个 session 要推多个独立 PR 时，3 个并行手段叠加可以让总 wall time 接近"做最长那个"而不是"全部相加"。每个手段单独都安全，叠加用按下表决定。

| 手段 | 怎么用 | 何时不用 |
|---|---|---|
| **A. Worktree-isolated agent 实施 PR B** | 主 Claude 在自己 worktree 写 PR A，同时 spawn 一个 sub-agent 用 worktree isolation 写 PR B；agent 在自己的 git worktree 实施 + 跑测试 + commit（**只本地 commit，不 push**），把 branch 名报回主 Claude 由主 Claude 统一 push / 开 PR / 跑 Codex | PR A 和 PR B 改同一个文件 / 互相依赖 / 共享一段难拆的 helper |
| **B. 并行 Codex Phase 2 / Phase 5** | 每个 `codex exec` 创建独立 session id，背景跑 2-3 个互不干扰；用不同 output 文件 (`/tmp/codex-finding-A.md` / `-B.md`) + 一个 Monitor 同时盯 | Codex CLI 已知 auth 在抖（连 smoke 都过不去）— 这种时候并行也不会更快 |
| **C. Pipeline 重叠** | Codex 在 review PR A 的时候，主 Claude 实施 PR B；Codex 在 final-review PR A 的时候，主 Claude 处理 PR B 的 Phase 2 finding | Phase 2 / Phase 5 的 review 出 P0/BLOCK，需要立刻回头改 PR A — 这时不要继续 push PR B |

**Agent worktree 注意：**
- Agent 启动时 `git fetch origin main` + 显式从 `origin/main` 创分支：`git checkout -b <branch> origin/main`。Agent 写完后主 Claude 主动 rebase 一次（main 在 agent 工作期间通常已经动）。
- Agent prompt 里**不要让 agent 自己 push** — push / open PR / 跑 Codex 由主 Claude 集中做，避免两边都开 codex exec 撞 auth。
- Agent 报告必须带：commit sha、worktree path、改动文件 + 行数、design 决策一句话、跑测试的结果。
- **Rebase 后必须重跑测试再开 Phase 2**。Agent 报上来的 "测试 OK" 是 rebase 之前的状态；rebase 解 conflict 时可能引入新 regression（典型场景：agent 改的函数被 main 上的另一 PR 重命名/重构了，conflict 解决看似 OK 实际行为变了）。主 Claude 必须在自己跑一次测试套件后才生成 Phase 2 diff / 推 PR — 把 agent 的测试结果当 hint，不当 gate。

**并行 Phase 5 的隐性失效（重要）：** 多 PR 并行跑 Phase 5 时，PR A 的 APPROVE 被消费 + merge 后会推动 origin/main，让"还没消费"的 PR B 的 APPROVE 实际上是对**旧 base** 出的判断 — gate 失效。规则：**消费每个 Phase 5 verdict 前，重 fetch origin/main，对比生成 Phase 5 diff 时的 SHA。如果动了，必须 rebase 当前分支 → 重跑测试 → 重新 generate diff → 重跑 Phase 5**，把 verdict 续上。Phase 5 diff 文件命名带 base SHA（例：`/tmp/diff-B-base-04e3d91.diff`）方便比对。

**实战参考：** 一个 session 串行做估计 50 分钟，并行做 22 分钟落地两个 PR — 接近 2× 加速。Codex 抓出 5 条 finding 跨 2 PR，**100% 是自审 + agent 自审都漏的**（包括"两条 client 实现路径只修了一条"+"次级清理路径实测行为与设计意图脱钩"）。这两个 anti-pattern 在 单串行 + 单 self-review 工作流里 100% 会上线。

---

## Codex CLI 调用模板

> **术语注**：下面的 "Monitor" 和 "task-notification" 是 Claude Code 内置的 background watcher / 事件机制（启动一个长跑命令，stdout 每行触发一次提醒）。如果你不用 Claude Code，等价做法是任意 `until [ -s /tmp/codex-finding.md ]; do sleep 5; done` 这种轮询 + 进程存活检查的 portable shell 循环 — 只要能在"输出落地"或"进程退出"时往下走就行。下文按 Claude Code 的术语写，请按你的工作环境翻译。

```bash
# Pre-PR review (Phase 2) / Final review (Phase 5)：
# 1) 永远 fetch 上游 + 用 origin/main 做 diff base（见下方"diff 必须用 origin/main"）
git fetch origin main
git diff origin/main...HEAD > /tmp/diff.diff
# 2) 极简 prompt
cat > /tmp/codex-prompt.md <<'EOF'
... 极简 prompt（见下方模板 — 不超过 10 行）
EOF
codex exec --output-last-message /tmp/codex-finding.md - < /tmp/codex-prompt.md \
  > /tmp/codex.log 2>&1 &     # background，等 task-notification

# Mode 2 parallel analyst：同样模式，但 Claude 在 foreground 也自己读代码。
# 非 main base（少见 — 长期 feature branch 等）：把上面 origin/main 换成
# origin/<your-base>，规则不变。但 base 不能是本地 stale 引用。
```

**坑**：`codex review --base <branch> "<prompt>"` 报 `--base 和 PROMPT 互斥`。**总是用 `codex exec`** + 显式 `git diff` 给它。

**diff 必须用 origin/main，不是本地 main：** `git fetch origin main` 之后本地 `main` 还停在旧 commit，`git diff main...HEAD` 的 merge-base 算出来包含 base→origin/main 的 delta（也就是别人在你工作期间已经 merge 进 main 的别的 PR），整个 diff 被污染，Codex review 的内容里夹杂大量与你无关的代码。**必须 `git diff origin/main...HEAD`**（或先 `git rebase origin/main` 再 diff 本地）。一次 Phase 2 review 中 1/3 的发现是 Codex 在批最近已 merge 的另一个 PR 的代码 — 浪费一整轮 review。

### Prompt minimalism（**最重要的一条**）

第二个 LLM 的价值是**独立判断**。Prompt 里多写一句 "specific things to scrutinize" / "我已经检查了 hygiene #2" / "这里我选 setdefault 是因为…"，Codex 就只能确认你的 framing — 抓不到你自审漏掉的东西。

实证：同一个 PR 跑了两版 prompt：

| Prompt | 行数 | 输出 |
|---|---|---|
| editorialized：context + self-summary + "review focus #1-7" + hygiene 自查清单 | ~150 | transport 中断；零 finding（即便没断也只会 confirm 我的 framing）|
| **极简：diff 路径 + hygiene 文件指针 + 输出格式** | **~10** | **2 条真 P1**（状态标签集合漏校验 + 并行 test runner 的 fixture isolation） |

两条都是 self-review 没看到的。这就是独立判断的 ROI。所以 prompt 模板**只允许下面两种最小形态**，谁加 "review focus #N" 谁回去重写。

### Phase 2 prompt 模板（pre-PR review）

```
Pre-PR review for /tmp/diff.diff.

Context: <这个 PR 闭环的 finding/audit 一句话>。Read the diff and the
actual files it touches.

Scope/relevant files (paths only, no analysis):
  <path/to/alternative_impl.py>
  <path/to/related_module.py>

Project hygiene rules: <hygiene doc 路径>。

Output: finding table (severity / file:line / issue / suggested fix)
+ single-line PASS / NEEDS_FIXES / BLOCK at the end.
```

允许扩展的只有两处：
- **Context** — 一句话告诉 Codex severity 基线（audit P0 修复 vs 小重构差很多）
- **Scope/relevant files** — 仅 paths，不写解释，不写 "review focus"。用于挡住 "Codex 漏看了 alternative implementation / 工厂切换备选" 那个回归。**特别是同一接口有多个 client 实现**（如 stdlib-based + SDK-based 两套），必须把所有 alternative impls 列进 Scope，否则 Codex 只 review 你 diff 里出现的那个，另一条静默 fail-open 上线。

**不要写自己的分析、不要列 review focus、不要预先提自己已经检查了什么**。"Scope" 只能列文件名，列了一句话以上就走回头路。

### Phase 5 prompt 模板（final review / merge gate）

```
Final review for the full PR diff against origin/main.

Run first (Claude prepares this before invoking codex):
  git fetch origin main
  git diff origin/main...HEAD > /tmp/diff.diff

Project hygiene rules: <hygiene doc 路径>。

Output: a single line containing exactly one token (APPROVE or REJECT),
followed optionally by a separate line with one short reason.
The first line is the binding gate — auto-merge parses it as
`head -n 1 | grep -E '^(APPROVE|REJECT)$'`.
```

Phase 5 review **整个 PR**，不是只 review 最新一个 commit — `git diff origin/main...HEAD` 覆盖从分叉点开始的所有 commit。只 review 最新 commit 会静默漏掉前面的 commit（多 commit PR 的典型坑）。始终 `git fetch` 后用 `origin/main` 做 base，原因和 Phase 2 一样：本地 `main` 可能 stale，会把上游别的 merged PR 的 delta 算进 diff。

Phase 5 比 Phase 2 还短 — 这一关就是 binding 决议，Codex 自己读 diff 自己判断。把 Phase 2 finding 列到 prompt 里反而会让它 anchor 在那批，漏掉 Phase 2 没抓的 cross-commit issue。

**为什么允许第二行 reason：** 实测 Codex 给 reason 对人审计很有用（例：`APPROVE\nNo blocking regression found; focused tests pass`）。但 reason 不参与 auto-merge gating — 第一行必须是裸 token，否则脚本无法机读。**不要把 reason 和 token 写在同一行**（曾考虑过 "APPROVE: <reason>" 之类的格式，会让 grep 失败）。也允许 Codex 只输出 bare token 不带 reason — Phase 5 的最简形态正合规。

---

## Codex 进度诊断（看到 `Auth(TokenRefreshFailed)` 时的判断流程）

`Auth(TokenRefreshFailed("Failed to parse server response"))` 这行经常出现在 stderr，但**多数时候 Codex 还在跑**。早 kill 会浪费一整轮 review。

**在 kill 前先分两种 mode：**

| | **Mode A — 还在做（常见）** | **Mode B — 真 hung** |
|---|---|---|
| stdout (`codex.log`) | 持续累加 `exec ... succeeded in Nms` 行 | 0 字节 / 完全静止 |
| `~/.codex/sessions/<today>/` | 有当天新 jsonl，mtime 在更新 | 没有当天文件（session 没建起来）|
| `ps -p <pid> -o stat,time` | CPU time 在累加 | `S` 状态 + `0:00.x` CPU 几分钟不动 |
| 处理 | **什么都不做**，让 Monitor 的 `until [ -s file ]` 自己等 — 真 review 通常 2-5 分钟才出 verdict | `kill <pid>` → `codex logout && codex login` → 重跑或 surface Owner |

**快速诊断 oneliner：**

```bash
# 还有 exec/succeeded 行 = 还在跑
tail -20 /tmp/codex.log | grep -E "exec|succeeded"
# 当天 session dir 有新文件 = 还在跑（注意：sessions 用本地日期分桶，
# 不是 UTC — 用 -u 在 UTC/local 跨日时会看错目录误报为 hung）
ls -lt ~/.codex/sessions/$(date +%Y/%m/%d)/ 2>/dev/null | head -3
```

**反模式（每条都吃过亏）：**
- 看到 stderr 那行 `Auth(...)` 就声称 "Codex 不可用"，立刻走 self-review 兜底
- 用 `Read` / `cat` poll `--output-last-message` 文件 — 那是终态文件，过程中是空的；只看 Monitor 事件
- 用 `codex.log`（stdout 重定向）判断 verdict — verdict 写在 `--output-last-message`，不是 stdout
- 在 PR body / commit message 里抢先写 "Codex unavailable" prose，结果 Monitor 后来真的 fire 了，PR body 还得回去改

**经验：等 Monitor 比 kill 重跑便宜得多。** Phase 2 给 5 分钟、Phase 5 给 3 分钟的等待预算 — Monitor 的 `until` 谓词自己处理。

### 503 outage（"等而非 reauth"）

Codex CLI 偶尔会同时返回 `Proxy connection failed: HTTP CONNECT failed with status 503` 和间歇性 `Auth(TokenRefreshFailed("invalid_grant: Grant not found"))`。**这是 chatgpt.com 后端的 transient outage，不是本地 auth grant 真失效**。

- **不要立刻 `codex logout && codex login`** — reauth 会把活跃 session 中断 + 不会修上游 outage
- Kill 当前卡住的 codex exec，**等 10-15 min**，跑 smoke (`echo hello | codex exec`) 验证再继续 Phase 2/5
- 如果手头 PR 必须立刻 merge，可以 surface Owner manual review（带 hygiene self-check 作 fallback）
- 重 reauth 仅在 24h+ 持续 outage 时考虑

实证：一次 14:34 outage start → 14:55 smoke 通 → 之后 Phase 5 一次过 APPROVE。

---

## Audit trail（每个 PR 必须有）

PR body 必含：
1. `## Summary` — 1-3 句改动 + 为什么
2. `## Phase 2 Codex review accept/reject 表` — 每条 finding 怎么处理 + 理由
3. `## Test plan` — checkbox list

Phase 5 完成后：Claude `gh pr comment <pr>` 加一条 review 决议留痕（含 round 数 + verdict + prompt/finding 文件路径）。

本地保留（session 期内）：
- `/tmp/codex-*-prompt.md` — sent prompt
- `/tmp/codex-*-finding.md` — 模型输出
- `/tmp/codex-*.log` — full transcript

---

## 真实踩过的坑（吃过亏才记的）

| Symptom | Root cause | Fix |
|---|---|---|
| `codex review --base <branch> "..."` 报错 | `--base` 跟 `[PROMPT]` 互斥 | 用 `codex exec` + 显式 git diff |
| 本地测试全过 / CI fail | 某个测试依赖本地没装 → SkipTest 跳过了 bug | 在 push 前 install 所有 CI requirements 跑一遍 |
| `attempts["n"] == 0` 在 mock 测试 | side_effect callable 签名没匹配 mocked method 的 args | 总是 `def fn(*args, **kwargs):` |
| `gh pr merge` 报 "main is used by worktree" | 本地 `git checkout main` 在 worktree env 失败 | 无影响 — `gh pr view --json state` 验证已 merge |
| 后台 task notification 没等就主动 poll | 误用了 ScheduleWakeup / sleep loop | 启动 background task 后什么都别做，等 task-notification |
| Codex 漏看一整个 module / alternative implementation | Codex 没自动扩 scope | Phase 2 prompt 加 `Scope/relevant files: <paths only>` 块（paths-only — 不写分析、不列 review focus），把 alternative impls / 工厂切换备选用文件名列出来；详见 §"Phase 2 prompt 模板" |
| Phase 5 重复 Phase 2 的 finding | 两个 phase prompt 没区分清楚 | Phase 2 抓 detail；Phase 5 抓 coherence + regression + hygiene + 单 token 决议 |
| Codex 严重度判断和 Claude 不一致 | 两边 prior 不同 | Mode 2 challenge protocol 解决；Mode 1 中升级 Owner |
| Codex 输出和我自己的分析高度一致，没抓到新问题 | Prompt 里把自己的 framing / "review focus" / 自审清单都喂他了，杀掉了独立判断 | Prompt 极简化（见 §"Prompt minimalism"）— 只给 diff 路径 + hygiene 指针 + 输出格式 |
| `Auth(TokenRefreshFailed)` 出现就以为 Codex 死了 | 那行实际是 transport 层重连噪音，Codex 主流程多数还在跑 | 先看 stdout 是否还在累加 `exec ... succeeded` 行 + `~/.codex/sessions/<today>/` 是否有新 jsonl，再决定 kill；详见 §"Codex 进度诊断" |
| 同时 503 + TokenRefreshFailed → 立刻 reauth | 后端 transient outage 误判成本地 auth 真失效 | 等 10-15 min 跑 smoke 验通再继续；详见 §"503 outage" |
| Phase 5 verdict 一直没出 → 提前在 PR body 写 "Codex unavailable" → Monitor 后来 fire 真 verdict 出来了 → PR body 回炉 | 没等 `--output-last-message` 文件落地就改写 PR body / commit message | 在 Monitor 事件 fire 前不要写 "verdict 状态"语句到任何持久化产物（PR body / commit message / 代码注释）|
| Codex Phase 2 finding 中 1/3 是别的已 merge PR 的代码 | `git diff main...HEAD` 用了 stale 本地 main，merge-base 算到旧 commit，把上游别的 merged PR 的 delta 算进来 | `git fetch origin main` 后用 `git diff origin/main...HEAD`；或先 `git rebase origin/main` 再 diff |
| Sub-agent worktree 用 `git checkout -b <branch> main` 本地 main stale，工作期间 main 已经动；agent push 时 conflict 一片 | Agent 启动早，主 Claude 那边 main 后来 merged 别的 PR，agent 没 fetch | Agent prompt 显式要求 `git fetch origin main && git checkout -b <branch> origin/main`；agent 完事后主 Claude 主动 rebase + force-with-lease push |
| 同一接口多套 client 实现（stdlib + SDK）只修了 diff 里看见的那条 | Phase 2 prompt 没在 Scope 列出所有实现路径，Codex review 的范围被 diff 局限 | Phase 2 Scope 块强制列出所有 alternative impls；项目根记一份"接口 → 多实现路径"清单方便填 |
| Phase 2 finding 字面量没读透，accept 后改一半，Phase 5 同一条再 reject | "accept finding" 当成 "修了" 但没逐字复述 finding 的限定词 | Phase 3 收尾对每条 finding 列"本次改动满足的具体行为契约"，逐字对照原文；详见 §Phase 3 收尾 self-check #1 |
| 新代码里写了 `except OSError: pass`，Phase 5 抓出来违反 hygiene 规则 | 写新代码时没自审 hygiene；prompt-engineering 教训传递有上限 | Phase 3 收尾 hygiene grep（`git diff $(git merge-base origin/main HEAD)` — 用 merge-base 而非 `origin/main...HEAD` 才能扫到 uncommitted worktree）；详见 §Phase 3 收尾 self-check #2 |
| 改了 pre-existing 代码的语义（让某 except 路径变 load-bearing）但没动 log level | semantic uplift 盲点：新加 `state[k] = X` 时只看自己加的代码，没回头审 setter 们 | Phase 3 收尾 semantic uplift audit — 列 diff 里所有"新加的 if/state assign"，回看输入变量来源 + 相关 except 路径 / log level；详见 §Phase 3 收尾 self-check #3 |
| Phase 5 跑了 4 轮才 APPROVE，但 R2/R3 都是 1 行 fix（hygiene），按固定 3 轮 cap 应该 escalate Owner | escalate threshold 没考虑 fix cost，固定轮数 cap 反而把"已经 99% 完成的 PR"卡住等 Owner | escalate threshold 改 finding-cost-aware：第 3 轮 REJECT 时评估 fix 工作量，< 10 min 自决继续；详见 §Mode 1 5-phase Phase 5 段 |

---

## 适合的场景

判定标准：**"另一个独立模型再看一眼" 能降低风险或减少 Owner 工作量** 就值得跑。代表性场景包括 — 单人 / 小团队 / 开源 maintainer 缺人二审；资金 / 合规 / 安全相关代码；大 refactor 或 DB migration；预发布 audit 或架构选型（Mode 2）；review 另一个 LLM 写的代码；以及本 protocol 自身的 README / runbook 改动（"docs-only" 豁免不适用）。

完整应用场景列表 + 具体例子见 README：[zh](README.zh.md#适合的场景) / [en](README.md#where-this-fits)。

---

## 何时 *不* 用这套

- 1 行 typo / log message / comment — 直接 commit
- 时间紧急的 incident hotfix — 走 manual fast path，事后再 audit
- Owner 对改动有 100% 信心 + diff 很小 — workflow 有 token cost，不必杀鸡用牛刀
- 文档 only 改动 — Phase 2/5 通常无价值

**例外（必须走完整流程）**：本 protocol 自身的 README / runbook / 资产改动。它们是其他人采用这套工作流的入口，"文档 only" 豁免**不适用** — 一旦在这里出错，错误会扩散到所有 adopter。曾经有过一次例外被遗漏 — 把 protocol 自身的 README/runbook 当成普通 docs-only 跳过 Phase 2/5；事后由 Owner 抓出来补完整流程，这条规则就是从那次教训写进来的。

---

## 演进规则

- **拒绝率 > 30%** on Phase 5 → 说明 Phase 2 review 不够深 or Phase 5 prompt scope 过宽，调
- **同一坑命中两次** → 加到本文档 + memory
- Codex CLI 升级带来 behavior 变化 → 重测 prompt template

---

## 启用 checklist（首次设置）

1. [ ] Codex CLI 装到 user-local + 登录验证 (`npm config get prefix` = `$HOME/.local`，`codex exec "hello"` 通)
2. [ ] `~/.local/bin` 在 PATH 里（login shell 由 `~/.profile` 自动；cron 脚本必须显式 export）
3. [ ] 项目内有 CLI 自更新机制（cron 或 supervisor 调 wrapper script，详见上方"自更新建议"块）— 失败自动 rollback
4. [ ] `gh` CLI 已 auth + 能创 PR
5. [ ] CI workflow 配置好 PR 触发
6. [ ] 项目根放本文档（或新会话粘给 Claude）
7. [ ] 项目内定 prod-critical path 黑名单
8. [ ] Owner 显式说："Codex APPROVE → 自动 merge → CI 自动部署，我会监控告警"
9. [ ] 第一次跑用一个低风险 task 验证整链

---

## 第一次 ship 后 review 触发

- Claude 报告每个 phase 的进度（commit sha / CI run id / verdict）
- Owner 复盘：Codex 抓出来的东西真不真？拒绝率？哪些坑该加进本文档？
- 调整后再放更高风险的 task 进来
