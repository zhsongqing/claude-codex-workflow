# Claude + Codex 协作工作流

Two-LLM 开发协作 protocol。Claude（主对话）+ Codex（CLI）+ Owner（oversight + 最终裁决）。

把这份文档放进项目 `docs/`，或在新会话开头粘给 Claude，它就能按这套规则跑。

---

## 前提

- **Claude Code CLI** 已就绪（主对话载体）
- **Codex CLI** 已装并登录：`npm install -g @openai/codex`（≥ 0.129），`codex login`
- 一个 Git 仓库 + GitHub 仓（`gh` CLI 已装并 auth）+ CI（任何能在 PR 上跑测试的 CI 都行）
- Owner 同意把"merge gate"交给 Codex（可随时撤回）

---

## 角色契约

| Actor | 职责 |
|---|---|
| **Claude** | 主对话；实施代码；处理评审 finding；提 PR；Codex APPROVE 后 squash-merge |
| **Codex (CLI)** | 独立第二意见。仅在 Phase 2 (pre-PR review) 和 Phase 5 (final review/merge gate) 出现。或在 Mode 2 作并行分析者 |
| **Owner** | 任务边界 / 模式选择最终决定权；prod oversight；Phase 5 第 3 轮 REJECT 后的升级目标；Mode 2 中无法收敛的 disagreement 的裁决人 |

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
Phase 1  Claude 实施 + 写测试 + 跑本地测试套件
Phase 2  Codex pre-PR review (codex exec + git diff against base branch)
         → 输出 finding 表 + PASS / NEEDS_FIXES / BLOCK
Phase 3  Claude 处理 finding（accept / reject / defer），按需改代码
         → 提 draft PR，PR body 含 finding accept/reject 表
Phase 4  Claude push + 等 CI 绿 (gh run watch <id> --exit-status)
Phase 5  Codex final review (codex exec, 单 token APPROVE/REJECT)
         APPROVE → Claude 标 ready + squash-merge + delete-branch
                   CI 自动部署（如有 auto-deploy 流水线）
         REJECT  → 回 Phase 3，最多 3 轮，第 3 轮还 REJECT 升级 Owner
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
- **Phase 2 vs Phase 5 prompt 必须不同**。Phase 2 抓 detail；Phase 5 看 PR coherence + regression + 项目 hygiene + 输出单 token verdict。重复就是浪费 token。

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

## Codex CLI 调用模板

```bash
# Pre-PR review (Phase 2) / Final review (Phase 5)：
git diff <base-branch>...HEAD > /tmp/diff.diff   # 写到文件，prompt 里 reference
cat > /tmp/codex-prompt.md <<'EOF'
... self-contained prompt（含 file: paths + 项目 hygiene 规则引用 + 输出格式 spec）
EOF
codex exec --output-last-message /tmp/codex-finding.md - < /tmp/codex-prompt.md \
  > /tmp/codex.log 2>&1 &     # background，等 task-notification

# Mode 2 parallel analyst：同样模式，但 Claude 在 foreground 也自己读代码。
```

**坑**：`codex review --base <branch> "<prompt>"` 报 `--base 和 PROMPT 互斥`。**总是用 `codex exec`** + 显式 `git diff` 给它。

### Phase 2 prompt 模板（pre-PR review）

```
Pre-PR review for the diff at /tmp/diff.diff.

Context: <task summary in 1-2 sentences>. Background: <list of relevant
project docs / hygiene rules / files Codex should read>.

Review focus (in order):
1. Does the change actually solve the stated problem?
2. <task-specific risk #1>
3. <task-specific risk #2>
...

Skip: stylistic / docstring / formatting / discussion of unrelated issues.

Output format (strict):
### F1: <one-line title>
- severity: P0 | P1 | P2
- file: <path>:<L1-L2>
- 触发条件: 1-2 lines
- 后果: 1-2 lines
- 建议修法: 1-2 lines

End with one of: PASS | NEEDS_FIXES | BLOCK.
```

### Phase 5 prompt 模板（final review / merge gate）

```
Final review for merge gate. APPROVE triggers auto-merge to main.

Context: PR #<N> addresses <task>. Commits:
1. <sha> — <one-line>
2. <sha> — <one-line>
...
CI status: <all checks summary>.

Full diff at /tmp/diff.diff.

Review focus (DIFFERENT from pre-PR — DO NOT repeat detail-level findings):
1. Coherence: do all commits combine into a complete fix? Subtle inter-commit interactions?
2. Regression risk for prod (specifically: <project's prod risk vectors>)
3. Test coverage completeness (mock-passes-but-prod-fails?)
4. Project hygiene rules sweep (reference: <path to hygiene doc>)
5. Anything outright dangerous (state corruption / deadlock / secret leak)

Skip: stylistic / docstring / already-addressed pre-PR findings / unrelated audits.

Output format: same finding format as Phase 2, then a final line that is
EXACTLY one of (no other text):
- APPROVE
- REJECT

Treat APPROVE as binding. If any P0 doubt, REJECT.
```

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
| Codex 漏看一整个 module / alternative implementation | Codex 没自动扩 scope | Phase 2 prompt 显式列出**所有**相关文件，包括 alternative impls / 工厂切换的备选 |
| Phase 5 重复 Phase 2 的 finding | 两个 phase prompt 没区分清楚 | Phase 2 抓 detail；Phase 5 抓 coherence + regression + hygiene + 单 token 决议 |
| Codex 严重度判断和 Claude 不一致 | 两边 prior 不同 | Mode 2 challenge protocol 解决；Mode 1 中升级 Owner |

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

1. [ ] Codex CLI 装 + 登录验证 (`codex exec "hello"` 通)
2. [ ] `gh` CLI 已 auth + 能创 PR
3. [ ] CI workflow 配置好 PR 触发
4. [ ] 项目根放本文档（或新会话粘给 Claude）
5. [ ] 项目内定 prod-critical path 黑名单
6. [ ] Owner 显式说："Codex APPROVE → 自动 merge → CI 自动部署，我会监控告警"
7. [ ] 第一次跑用一个低风险 task 验证整链

---

## 第一次 ship 后 review 触发

- Claude 报告每个 phase 的进度（commit sha / CI run id / verdict）
- Owner 复盘：Codex 抓出来的东西真不真？拒绝率？哪些坑该加进本文档？
- 调整后再放更高风险的 task 进来
