# Claude + Codex Collaboration Workflow

Two-LLM development collaboration protocol. Claude (primary dialog) + Codex (CLI) + Owner (oversight + final arbitration).

Put this document in your project's `docs/` directory, or paste it at the start of a fresh Claude session, and Claude can run the workflow from these rules.

---

## Prerequisites

- **Claude Code CLI** is ready (the primary dialog surface)
- **Codex CLI** is installed and logged in: `npm install -g @openai/codex` (>= 0.129), `codex login`
- A Git repository + GitHub repository (`gh` CLI installed and authenticated) + CI (any CI that runs tests on PRs is fine)
- Owner agrees to delegate the "merge gate" to Codex (revocable at any time)

---

## Roles

| Actor | Responsibilities |
|---|---|
| **Claude** | Primary dialog; implements code; handles review findings; opens PRs; after Codex APPROVE, squash-merges |
| **Codex (CLI)** | Independent second opinion. Appears only in Phase 2 (pre-PR review) and Phase 5 (final review/merge gate), or as a parallel analyst in Mode 2 |
| **Owner** | Final authority on task boundaries / mode selection; prod oversight; escalation target after the 3rd Phase 5 REJECT; arbiter for Mode 2 disagreements that cannot converge |

---

## Two task modes

Claude explicitly announces which mode the task will use at task entry. Owner can veto and re-route with one sentence.

| Trigger (user wording signal) | Mode |
|---|---|
| "implement / add / refactor / fix / write" | **Mode 1 — Linear** |
| "analyze / design / review / audit / is X OK / X or Y" | **Mode 2 — Parallel** |
| Ambiguous | Claude defaults to Mode 1 and notes that it can switch |

---

## Mode 1 — Linear (5-phase, dev/fix)

```
Phase 1  Claude implements + writes tests + runs the local test suite
Phase 2  Codex pre-PR review (codex exec + git diff against base branch)
         → outputs finding table + PASS / NEEDS_FIXES / BLOCK
Phase 3  Claude handles findings (accept / reject / defer), edits code as needed
         → opens draft PR; PR body includes finding accept/reject table
Phase 4  Claude pushes + waits for CI green (gh run watch <id> --exit-status)
Phase 5  Codex final review (codex exec, single-token APPROVE/REJECT
         on its own line — an optional one-line reason may follow but
         is not parsed by the gating script)
         APPROVE → Claude marks ready + squash-merges + deletes branch
                   CI auto-deploys (if an auto-deploy pipeline exists)
         REJECT  → return to Phase 3, up to 3 rounds; a 3rd REJECT escalates to Owner
```

**Hard rules**

- Phase 5 auto-merge is enabled only because Owner explicitly authorized it; Owner can revoke that authorization at any time.
- **Mandatory post-merge observation window**: Owner watches monitoring (Sentry / Datadog / self-hosted alerts) and rolls back immediately if anything breaks. Recommended: 30 minutes.
- **Prod-critical path blacklist** (even with Codex APPROVE, Claude must surface these to Owner for a second confirmation). Each project defines its own list; common categories:
  - Risk control / limits / security-related code
  - Secret / config / environment variable loading logic
  - Core calls involving money / asset movement
  - Database schema migration
  - CI / deployment scripts / rollback scripts
- **Phase 2 and Phase 5 prompts must be different**. Phase 2 catches details; Phase 5 checks PR coherence + regression + project hygiene + a single-token verdict (on its own line so it is machine-parseable; the optional one-line reason on the next line is for human readers and is not part of the auto-merge gate). Repeating the same prompt wastes tokens.

---

## Mode 2 — Parallel (analyst + multi-round challenge, design/audit)

```
Step 1  Claude and Codex analyze independently (Codex runs in the background
        with codex exec; Claude reads the code in the foreground)
Step 2  Two-way cross-challenge: each side reviews the other's findings and tags
        each one AGREE / AGREE_BUT_RESEVERITY:Pn / DISAGREE + one-line reason
Step 3  Merge: keep shared conclusions + voluntary concessions + drop false
        positives both sides agree on; escalate disagreements that cannot converge
        to Owner for arbitration
        Record how many findings each side dropped — this is a real quality signal
        that the cross-check worked
```

**Constraints**

- Round cap: **2 challenge rounds**. More than that means sharply diminishing returns + token waste.
- Prompts for both sides must be self-contained — Codex has no conversation history and must learn the project background from the prompt (key document paths / rules / output format).
- Mode 2 does not open a PR; it is the analysis stage. To fix something, enter Mode 1.
- Do not make Codex the "judge" of its own findings. The challenge prompt you give it must be independent and must not attach its original findings.

---

## Mode switching

A task may move from Mode 2 → Mode 1 (after review, you decide to make changes). Claude must explicitly announce the switch and restart the full five Phase 1-5 flow.

Do not switch in the opposite direction (Mode 1 → Mode 2 midstream). If code is already being written, the target is already known; Mode 2 is for exploration.

---

## Parallel multi-PR pipeline (when one session ships multiple PRs)

When one session needs to ship two or three independent PRs, three orthogonal parallelism techniques can be stacked so the total wall time is closer to "the longest one" than "the sum". Each technique is safe alone; combine them per the table below.

| Technique | How | When NOT to use |
|---|---|---|
| **A. Worktree-isolated agent for PR B** | The primary Claude implements PR A in its own worktree; in parallel it spawns a sub-agent with `isolation: "worktree"` to implement PR B. The sub-agent works in its own git worktree (implements + runs tests + commits **locally only — does not push**) and reports its branch name back. The primary Claude pushes / opens PRs / runs Codex centrally | PR A and PR B touch the same file, depend on each other, or share a hard-to-split helper |
| **B. Concurrent Codex Phase 2 / Phase 5** | Each `codex exec` creates its own session id; running 2-3 in the background concurrently is safe. Use distinct output files (`/tmp/codex-finding-A.md` / `-B.md`) and a single Monitor watching for both | Codex CLI auth is already flaky (even smoke tests fail) — concurrency does not help if the CLI is broken |
| **C. Pipeline overlap** | While Codex reviews PR A in Phase 2, the primary Claude implements PR B. While Codex final-reviews PR A, the primary Claude addresses PR B's Phase 2 findings | Phase 2 / Phase 5 returns P0 / BLOCK on PR A and you must return to PR A immediately — pause work on PR B |

**Agent worktree notes:**
- The agent must `git fetch origin main` and explicitly branch from `origin/main`: `git checkout -b <branch> origin/main`. Once the agent finishes, the primary Claude rebases once (main usually advanced during the agent's work).
- The agent prompt **must not let the agent push** — push / open PR / run Codex are centralized in the primary Claude to avoid two parallel `codex exec` calls colliding on auth.
- The agent's report must include: commit sha, worktree path, files changed + line counts, one-sentence design choice, test results.

**Empirical reference:** A 2-PR session (independent fixes + independent tests) that would have taken ~50 min serial took 22 min parallel — close to a 2× speedup. Codex caught 5 findings across the 2 PRs, **100% of them issues self-review and the agent's self-review missed** (including "two client implementation paths but only one was fixed" and "design intent and implementation drifted apart"). Both anti-patterns would have shipped in a single-serial + single-self-review workflow.

---

## Codex CLI invocation templates

```bash
# Pre-PR review (Phase 2) / Final review (Phase 5):
git diff <base-branch>...HEAD > /tmp/diff.diff
cat > /tmp/codex-prompt.md <<'EOF'
... minimal prompt (see template below — under 10 lines)
EOF
codex exec --output-last-message /tmp/codex-finding.md - < /tmp/codex-prompt.md \
  > /tmp/codex.log 2>&1 &     # background; wait for task-notification

# Mode 2 parallel analyst: same pattern, while Claude also reads the code in the foreground.
```

**Pitfall**: `codex review --base <branch> "<prompt>"` errors because `--base and PROMPT are mutually exclusive`. **Always use `codex exec`** + an explicit `git diff`.

**Diff must use `origin/main`, not local `main`:** After `git fetch origin main`, your local `main` still points at an old commit. `git diff main...HEAD` then computes the merge-base from that stale local main, so the diff includes everything between local main and origin/main (i.e. other PRs that were merged into main while you were working) on top of your actual changes. Codex then reviews a polluted diff with code that is not yours. **Always use `git diff origin/main...HEAD`** (or `git rebase origin/main` first and then diff locally). This has bitten us: one Phase 2 review had ~1/3 of its findings about other already-merged PRs — a wasted round.

### Prompt minimalism (**the most important rule**)

The value of the second LLM is **independent judgment**. Every extra line in the prompt — "specific things to scrutinize", "I have already checked hygiene #2", "I chose setdefault here because..." — pushes Codex toward confirming your framing instead of catching what your self-review missed.

Empirical: the same state-persistence PR was sent with two prompts:

| Prompt | Lines | Output |
|---|---|---|
| editorialized: context + self-summary + "review focus #1-7" + hygiene checklist | ~150 | transport interruption; zero findings (and even with a clean run it would only have confirmed the framing) |
| **minimal: diff path + hygiene doc pointer + output format** | **~10** | **2 real P1s** (one missed validation set + xdist test isolation regression) |

Both were issues my 7-item self-review missed. That is the ROI of independent judgment. The prompt templates below are the **only forms allowed** — anyone adding a "review focus #N" goes back and rewrites it.

### Phase 2 prompt template (pre-PR review)

```
Pre-PR review for /tmp/diff.diff.

Context: <one sentence on the finding/audit this PR closes>. Read the diff and the
actual files it touches.

Scope/relevant files (paths only, no analysis):
  <path/to/alternative_impl.py>
  <path/to/related_module.py>

Project hygiene rules: <hygiene doc path>.

Output: finding table (severity / file:line / issue / suggested fix)
+ single-line PASS / NEEDS_FIXES / BLOCK at the end.
```

Only two things may be expanded:
- **Context** — one sentence to give Codex the severity baseline (a P0 audit fix vs. a small refactor differ a lot).
- **Scope/relevant files** — paths only. No commentary, no "review focus". This blocks the "Codex missed an alternative implementation / factory-switch candidate" regression (see the pitfall table). **In particular, when one interface has multiple client implementations (e.g. a stdlib path and an SDK path)** every alternative implementation must be listed in Scope, otherwise Codex only reviews whichever path appears in your diff.

**Do not write your own analysis, do not list a "review focus", and do not pre-mention what you have already checked**. "Scope" must be file paths only — anything more than a single line and you are sliding back to editorial mode.

### Phase 5 prompt template (final review / merge gate)

```
Final review for the full PR diff against origin/main.

Run first (Claude prepares this before invoking codex):
  git fetch origin main
  git diff origin/main...HEAD > /tmp/diff.diff

Project hygiene rules: <hygiene doc path>.

Output: a single line containing exactly one token (APPROVE or REJECT),
followed optionally by a separate line with one short reason.
The first line is the binding gate — auto-merge parses it as
`head -n 1 | grep -E '^(APPROVE|REJECT)$'`.
```

Phase 5 reviews the **whole PR**, not just the latest commit — `git diff origin/main...HEAD` covers every commit on the branch since fork from main. Reviewing only the latest commit silently ignores earlier commits in the PR (a real foot-gun on multi-commit PRs). Always pin against `origin/main` after `git fetch` for the same reason as Phase 2 — local `main` may be stale and would inflate the diff with unrelated upstream merges.

Phase 5 is even shorter than Phase 2 — this stage is the binding decision; Codex reads the diff and decides on its own. Listing Phase 2 findings in the Phase 5 prompt only anchors it on those and lets it miss cross-commit issues that Phase 2 didn't see.

**Why a second-line reason is allowed:** Codex's reason is empirically useful for human auditing (a typical output is `APPROVE\nNo blocking regression found; focused tests pass`). But the reason is not part of the auto-merge gate — the first line must be a bare token, otherwise the parsing script fails. **Do not put the reason and the token on the same line** (a format like `APPROVE: <reason>` would break grep). Codex emitting just the bare token without a reason is also fine — that is the simplest valid Phase 5 form.

---

## Codex progress diagnosis (what to do when you see `Auth(TokenRefreshFailed)`)

`Auth(TokenRefreshFailed("Failed to parse server response"))` shows up in stderr quite often, but **most of the time Codex is still running**. Killing too early wastes a whole review round.

**Before killing, classify the failure mode:**

| | **Mode A — still working (common)** | **Mode B — actually hung** |
|---|---|---|
| stdout (`codex.log`) | Continues to accrue `exec ... succeeded in Nms` lines | 0 bytes / completely static |
| `~/.codex/sessions/<today>/` | Has a session jsonl from today, mtime is updating | No file from today (the session never started) |
| `ps -p <pid> -o stat,time` | CPU time is accumulating | `S` state + `0:00.x` CPU for several minutes |
| Action | **Do nothing.** Let the Monitor's `until [ -s file ]` loop finish — a real review takes ~2-5 minutes after the first `Auth` warning before the verdict file appears | `kill <pid>` → `codex logout && codex login` → retry or surface to Owner |

**Quick diagnosis one-liner:**

```bash
# stdout has new exec/succeeded lines = still running
tail -20 /tmp/codex.log | grep -E "exec|succeeded"
# session dir has a today-stamped file = still running
# (sessions are bucketed by LOCAL date, not UTC — using -u
#  around the local/UTC day boundary points at the wrong dir
#  and falsely reports the session as hung)
ls -lt ~/.codex/sessions/$(date +%Y/%m/%d)/ 2>/dev/null | head -3
```

**Anti-patterns (each one a real lesson):**
- Seeing the stderr `Auth(...)` line and concluding "Codex is unavailable", then immediately falling back to self-review.
- Polling `--output-last-message` with `Read` / `cat` — that file is the terminal artifact and is empty during the run; only watch the Monitor event.
- Reading `codex.log` (the stdout redirect) for the verdict — the verdict goes to `--output-last-message`, not stdout.
- Pre-writing "Codex unavailable" prose into the PR body or commit message — if the Monitor later fires with the real verdict, the PR body has to be rewritten.

**Lesson: waiting for the Monitor is much cheaper than killing and re-running.** Budget Phase 2 for 5 minutes and Phase 5 for 3 minutes — the Monitor's `until` predicate handles it.

---

## Audit trail

Every PR body must include:
1. `## Summary` — 1-3 sentences on what changed and why
2. `## Phase 2 Codex review accept/reject table` — how each finding was handled, with rationale
3. `## Test plan` — checkbox list

After Phase 5 completes: Claude adds a `gh pr comment <pr>` review decision record (including round count + verdict + prompt/finding file paths).

Keep locally (for the session):
- `/tmp/codex-*-prompt.md` — sent prompt
- `/tmp/codex-*-finding.md` — model output
- `/tmp/codex-*.log` — full transcript

---

## Real pitfalls (lessons paid for)

| Symptom | Root cause | Fix |
|---|---|---|
| `codex review --base <branch> "..."` errors | `--base` and `[PROMPT]` are mutually exclusive | Use `codex exec` + an explicit git diff |
| Local tests pass / CI fails | A test depended on something not installed locally, so SkipTest skipped the bug | Install all CI requirements before push and run the suite once |
| `attempts["n"] == 0` in a mock test | The side_effect callable signature did not match the mocked method's args | Always use `def fn(*args, **kwargs):` |
| `gh pr merge` says "main is used by worktree" | Local `git checkout main` failed in a worktree environment | No impact — verify merge state with `gh pr view --json state` |
| Started polling without waiting for the background task notification | Misused ScheduleWakeup / sleep loop | After starting the background task, do nothing; wait for task-notification |
| Codex missed an entire module / alternative implementation | Codex did not expand scope automatically | Add a `Scope/relevant files: <paths only>` block to the Phase 2 prompt (paths only — no analysis, no review focus). List alternative implementations / factory-switch candidates by file name. See "Phase 2 prompt template". |
| Phase 5 repeats Phase 2 findings | The two phase prompts were not distinct enough | Phase 2 catches details; Phase 5 checks coherence + regression + hygiene + single-token decision |
| Codex and Claude disagree on severity | Different priors on the two sides | Mode 2 challenge protocol resolves this; in Mode 1, escalate to Owner |
| Codex output mirrors my own analysis and catches nothing new | The prompt fed Codex my framing / "review focus" / self-review checklist, killing independent judgment | Use the minimal prompt (see "Prompt minimalism") — diff path + hygiene pointer + output format only |
| `Auth(TokenRefreshFailed)` appears and I assume Codex is dead | That stderr line is transport-layer noise; the main session usually keeps running | Check whether stdout is still accumulating `exec ... succeeded` lines + whether `~/.codex/sessions/<today>/` has a fresh jsonl before killing. See "Codex progress diagnosis". |
| Phase 5 verdict is slow → I prematurely write "Codex unavailable" into the PR body → Monitor later fires with the real verdict → PR body has to be rewritten | I rewrote PR body / commit messages before the `--output-last-message` file landed | Do not write any "verdict status" sentence into a persistent artifact (PR body / commit message / code comment) until the Monitor event has fired |
| ~1/3 of Codex Phase 2 findings refer to code from another already-merged PR | `git diff main...HEAD` used a stale local `main`; the merge-base was computed from an old commit, pulling in unrelated upstream PRs | After `git fetch origin main`, use `git diff origin/main...HEAD`; or `git rebase origin/main` first and then diff |
| Sub-agent worktree was branched from `main` (stale local) and main moved during the agent's work; on push it has a wall of conflicts | The agent started early, the parent Claude later merged other PRs into main, the agent never re-fetched | The agent prompt must explicitly run `git fetch origin main && git checkout -b <branch> origin/main`; after the agent finishes, the parent Claude proactively rebases + force-with-lease pushes |
| Multiple client implementations of one interface (e.g. stdlib + SDK) — only the one in the diff was fixed | The Phase 2 prompt did not list all implementation paths in Scope, so Codex's review was constrained to the diff | The Phase 2 Scope block must list alternative implementations like `bfxapi_client.py / bitfinex_client.py`. Keep a project-root list of "interface → multiple implementations" so this is easy to fill in. |

---

## Where this fits

Decision rule: run the workflow whenever **"another independent model taking one more look" reduces risk or saves Owner attention**. Representative cases include solo dev / small teams / open-source maintainers without a second reviewer; money / compliance / security-related code; large refactors or DB migrations; pre-launch audits or architecture choices (Mode 2); reviewing code written by another LLM; and changes to this protocol's own README / runbook (the "docs-only" exemption does not apply).

For the full list of use cases and concrete examples, see README: [en](README.md#where-this-fits) / [zh](README.zh.md#适合的场景).

---

## When not to use this

- A 1-line typo / log message / comment — just commit
- Time-critical incident hotfix — use the manual fast path, then audit afterward
- Owner has 100% confidence + the diff is tiny — the workflow has token cost and is not worth it
- Docs-only changes — Phase 2/5 usually add no value

**Exception (must run the full workflow)**: changes to this protocol's own README / runbook / assets. These are the entry points others use to adopt the workflow, so the "docs-only" exemption **does not apply** — an error here propagates to every adopter. There was one prior incident where this exception was forgotten — the protocol's own README/runbook was treated as docs-only-exempt and skipped Phase 2/5. Owner caught the meta-error after the fact and we ran the full workflow retroactively. This rule is the lesson from that incident.

---

## Evolution

- **REJECT rate > 30%** on Phase 5 → Phase 2 review is not deep enough, or Phase 5 prompt scope is too broad; tune it
- **Same pitfall hits twice** → add it to this document + memory
- Codex CLI upgrade changes behavior → retest the prompt template

---

## Adoption checklist

1. [ ] Codex CLI installed + login verified (`codex exec "hello"` works)
2. [ ] `gh` CLI authenticated + able to create PRs
3. [ ] CI workflow configured to run on PRs
4. [ ] This document placed at the project root (or pasted into a fresh Claude session)
5. [ ] Project-specific prod-critical path blacklist defined
6. [ ] Owner explicitly says: "Codex APPROVE → auto-merge → CI auto-deploys, and I will monitor alerts"
7. [ ] First run uses a low-risk task to validate the whole chain

---

## Current Owner authorization

- Claude reports progress for each phase (commit sha / CI run id / verdict)
- Owner reviews after the first ship: Were Codex findings real? What was the reject rate? Which pitfalls should be added to this document?
- After adjustments, allow higher-risk tasks into the workflow
