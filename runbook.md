# Claude + Codex Collaboration Workflow

Two-LLM development collaboration protocol. Claude (primary dialog) + Codex (CLI) + Owner (oversight + final arbitration).

Put this document in your project's `docs/` directory, or paste it at the start of a fresh Claude session, and Claude can run the workflow from these rules.

---

## Prerequisites

- **Claude Code CLI** is ready (the primary dialog surface)
- **Codex CLI** is installed and logged in. A user-local install is recommended (lets an agent self-update later without sudo):

  ```bash
  # One-time bootstrap (per machine)
  npm config set prefix "$HOME/.local"
  npm install -g @openai/codex@latest
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.profile  # most Ubuntu setups do this already; verify
  codex login    # ChatGPT subscription or API key both work
  codex exec "hello"   # verify

  # Future upgrades — any cron / supervisor can run this, no sudo required
  npm install -g @openai/codex@latest
  echo hello | codex exec  # smoke
  ```

  **PATH note**: cron / systemd does not load `~/.profile`. Any script invoked by cron must `export PATH="$HOME/.local/bin:$PATH"` at the top, or it will fall back to a system-wide stale codex (if one exists).

  **Self-update recommendation**: have cron or a supervisor invoke a small wrapper (pseudocode):
  ```bash
  cur=$(codex --version | awk '{print $2}')
  latest=$(npm view @openai/codex version)
  [ "$cur" = "$latest" ] && exit 0
  npm install -g @openai/codex@"$latest"
  echo hello | codex exec >/dev/null 2>&1 \
    || npm install -g @openai/codex@"$cur"   # roll back if smoke fails
  ```

- A Git repository + GitHub repository (`gh` CLI installed and authenticated) + CI (any CI that runs tests on PRs is fine)
- Owner agrees to delegate the "merge gate" to Codex (revocable at any time)

---

## Roles

| Actor | Responsibilities |
|---|---|
| **Claude** | Primary dialog; implements code; handles review findings; opens PRs; after Codex APPROVE, squash-merges |
| **Codex (CLI)** | Independent second opinion. Appears only in Phase 2 (pre-PR review) and Phase 5 (final review / merge gate), or as a parallel analyst in Mode 2 |
| **Owner** | Final authority on task boundaries / mode selection; prod oversight; escalation target once the Phase 5 escalate threshold trips (threshold defined in the Mode 1 5-phase block below); arbiter for Mode 2 disagreements that cannot converge |

---

## Two task modes

Claude **explicitly announces** which mode the task will use at task entry. Owner can veto and re-route with one sentence.

| Trigger (user wording signal) | Mode |
|---|---|
| "implement / add / refactor / fix / write" | **Mode 1 — Linear** |
| "analyze / design / review / audit / is X OK / X or Y" | **Mode 2 — Parallel** |
| Ambiguous | Claude defaults to Mode 1 and notes that it can switch |

---

## Mode 1 — Linear (5-phase, dev / fix)

```
Phase 1    Claude implements + writes tests + runs the local test suite
Phase 2    Codex pre-PR review (codex exec + git diff against base branch)
           → finding table + PASS / NEEDS_FIXES / BLOCK
Phase 3    Claude handles findings (accept / reject / defer), edits code as needed
           → opens draft PR; PR body includes finding accept/reject table
Phase 3.5  Claude runs 4 self-checks before push (see §Phase 3 closeout self-check)
Phase 4    Claude pushes + waits for CI green (gh run watch <id> --exit-status)
Phase 5    Codex final review (codex exec, single-token APPROVE/REJECT
           on its own line — an optional second line can carry one reason
           but does not participate in gate parsing)
           APPROVE → Claude marks ready + squash-merges + deletes branch
                     CI auto-deploys (if an auto-deploy pipeline exists)
           REJECT  → return to Phase 3. Escalation is finding-cost-aware:
                     - Rounds 1-2 REJECT: continue on your own
                     - Round 3 REJECT: estimate fix size:
                       * < 10 min fix (usually hygiene / log-level / typo):
                         continue to round 4 on your own
                       * ≥ 10 min fix or design-tradeoff territory:
                         escalate to Owner
                     - Any 5th REJECT: force escalation (hard cap retained)
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
- **Phase 2 and Phase 5 prompts must be different**. Phase 2 catches detail; Phase 5 checks PR coherence + regression + project hygiene + emits a single-token verdict (on its own line for machine reading — the optional reason on a second line is for human readers only and does not participate in auto-merge gating). Repeating the same prompt wastes tokens.

### Phase 3 closeout self-check (run before push, ~10-15 min)

The window between "Phase 3 code change complete" and "Phase 4 push" is a rule gap. The four self-checks below tend to compress Phase 5 from a typical 3-4 rounds back to 1-2. High ROI. **These do not replace Codex** (the independent-judgment value stands); they reduce round count / wall time.

1. **Restate each finding and self-verify** — for every Phase 2 finding, list "the specific behavioral contracts the change satisfies," matched word-for-word against the finding text. "Accept" ≠ "fixed correctly."
   - **Typical pitfall**: a finding read "Append each confirmed id immediately after it is deemed canceled, **before optional logging/enrichment or more loop work**." The Phase 3 fix moved the append to **after** the logging/enrichment; "accept" never restated the "before optional logging/enrichment" qualifier; Phase 5 R1 rejected on the same point. Verbatim matching catches this.
2. **Hygiene grep** — run before commit (the command scans the worktree, including unstaged + staged changes):
   ```bash
   git diff "$(git merge-base origin/main HEAD)" -- <src dirs> | grep -nE "^\+.*\bpass\b|^\+.*logger\.debug.*exc"
   ```
   Fix any hit. **Note**: use `merge-base`, not `origin/main...HEAD` — the latter only covers what's already committed to HEAD, missing the freshly-edited but uncommitted code you most need to self-check. Tune the grep pattern to your project's hygiene rules — the two above match "swallow exception" and "logger.debug eats error", classic fail-silent anti-patterns.
   - **Typical pitfall**: a new helper used `except OSError: pass` three times; Phase 5 R2 rejected on hygiene. One grep catches it.
3. **Semantic uplift audit** — list every "new `if old_var: ...`" or "`state[k] = something_load_bearing`" in the diff and **re-read how `old_var` / `state[k]` was set before**. Are the related except branches / log levels still appropriate? A pre-existing `logger.debug` can become load-bearing under the new semantics and silently swallow.
   - **Typical pitfall**: added `refresh_ok_by_asset[asset] = False`, which made a pre-existing `logger.debug("refresh failed ...")` path an input to a new GC routine; never went back to inspect the log level; Phase 5 R3 rejected — if refresh failure gets swallowed at debug, the GC decision runs on wrong state.
4. **Mini Phase 5 self-review** — run the Phase 5 prompt mentally against your own diff. Pay attention to spots that "feel a bit off": does the variable naming look like production code? Can you explain why each new try/except needs to catch what it catches? Does the docstring reflect actual behavior?

**ROI estimate**: ~15 min self-check saves ~25 min of Codex round (each round ~10 min review + ~5 min push/CI/wait). Compressing 4 rounds to 2 is a net ~25 min/PR.

---

## Mode 2 — Parallel (analyst + multi-round challenge, design / audit)

```
Step 1  Claude and Codex analyze independently (Codex in background via codex exec;
        Claude reads the code in the foreground)
Step 2  Two-way cross-challenge: each side reviews the other's findings and tags
        each one AGREE / AGREE_BUT_RESEVERITY:Pn / DISAGREE + one-line reason
Step 3  Merge: keep shared conclusions + voluntary concessions + drop false
        positives both sides agree on; escalate unresolved disagreements to Owner.
        Track how many findings each side dropped — this is a real quality signal
        that the cross-check worked.
```

**Constraints**

- Round cap: **2 challenge rounds**. More than that = sharply diminishing returns + token waste.
- Prompts for both sides must be self-contained — Codex has no conversation history and must learn project background from the prompt (key document paths / rules / output format).
- Mode 2 does not open a PR; it is the analysis stage. To fix something, enter Mode 1.
- Do not make Codex the "judge" of its own findings. The challenge prompt you give it must be independent and must not attach its original findings.

---

## Mode switching

A task may move from Mode 2 → Mode 1 (after review, you decide to make changes). Claude must explicitly announce the switch and restart the full Mode 1 five-phase flow.

Do not switch in the opposite direction (Mode 1 → Mode 2 midstream). If code is already being written, the target is already known; Mode 2 is for exploration.

---

## Parallel multi-PR pipeline (speed up multiple tasks in one session)

When a session needs to push multiple independent PRs, stacking three parallelism levers lets total wall time approach "the longest one" instead of "all of them added up." Each lever is safe on its own; the table below shows when to stack.

| Lever | How to use | When not to |
|---|---|---|
| **A. Worktree-isolated agent implements PR B** | Main Claude writes PR A in its own worktree, and simultaneously spawns a sub-agent with worktree isolation to write PR B; the agent implements + runs tests + commits in its own git worktree (**local commit only, no push**) and reports the branch name back to main Claude, who handles push / open PR / run Codex centrally | PR A and PR B touch the same file / depend on each other / share a hard-to-split helper |
| **B. Parallel Codex Phase 2 / Phase 5** | Each `codex exec` creates a distinct session id; run 2-3 in the background with no interference; use different output files (`/tmp/codex-finding-A.md` / `-B.md`) + a single Monitor watching both | Codex CLI auth is known to be flaky (even smoke calls fail) — parallel won't be faster in that state |
| **C. Pipeline overlap** | While Codex is reviewing PR A, main Claude implements PR B; while Codex is doing final review on PR A, main Claude handles PR B's Phase 2 findings | Phase 2 / Phase 5 review on PR A surfaces a P0/BLOCK requiring an immediate fix — do not keep pushing PR B in that state |

**Agent worktree notes:**
- On agent start: `git fetch origin main` + explicitly branch off `origin/main`: `git checkout -b <branch> origin/main`. After the agent finishes, main Claude proactively rebases once (main has typically moved during agent work).
- **Do not let the agent push** — push / open PR / run Codex stays centralized in main Claude, to avoid two callers fighting over `codex exec` auth.
- Agent report must include: commit sha, worktree path, files changed + line count, one-line design decision, test result.
- **After rebase, re-run tests before opening Phase 2**. The agent's "tests pass" report is the state before rebase; resolving conflicts during rebase can introduce new regressions (typical case: a function the agent changed was renamed/refactored by another PR landed on main; the conflict resolution looks fine but behavior shifted). Main Claude must run the test suite once locally before generating the Phase 2 diff or pushing — treat the agent's test result as a hint, not a gate.

**Silent failure mode for parallel Phase 5 (important):** When multiple PRs run Phase 5 in parallel, after PR A's APPROVE is consumed + merged, origin/main advances; any not-yet-consumed APPROVE for PR B is now actually a verdict against the **old base** — the gate is invalidated. Rule: **before consuming each Phase 5 verdict, re-fetch origin/main and compare against the SHA used when Phase 5 diff was generated. If main moved, rebase the current branch → re-run tests → regenerate the diff → re-run Phase 5** to renew the verdict. Naming Phase 5 diff files with the base SHA (e.g. `/tmp/diff-B-base-04e3d91.diff`) makes the comparison easy.

**Real-world result:** one session estimated ~50 min sequential vs ~22 min parallel for two PRs — roughly 2× speedup. Codex caught 5 findings across the 2 PRs, **100% of them missed by both self-review and the agent's self-review** (including "two client implementation paths but only one fixed" + "orphan decider behavior diverged from design intent"). Both anti-patterns would have shipped under a single-serial + single-self-review workflow.

---

## Codex CLI invocation templates

```bash
# Pre-PR review (Phase 2) / Final review (Phase 5):
# 1) Always fetch upstream first + use origin/main as the diff base (see "diff must use origin/main" below)
git fetch origin main
git diff origin/main...HEAD > /tmp/diff.diff
# 2) Minimal prompt
cat > /tmp/codex-prompt.md <<'EOF'
... minimal prompt (see templates below — no more than ~10 lines)
EOF
codex exec --output-last-message /tmp/codex-finding.md - < /tmp/codex-prompt.md \
  > /tmp/codex.log 2>&1 &     # background; wait for task-notification

# Mode 2 parallel analyst: same pattern, while Claude also reads the code in the foreground.
# Non-main base (rare — long-lived feature branches etc.): replace origin/main above
# with origin/<your-base>. Same rules apply. The base must never be a stale local ref.
```

**Pitfall**: `codex review --base <branch> "<prompt>"` errors with `--base and PROMPT are mutually exclusive`. **Always use `codex exec`** + an explicit `git diff`.

**diff must use origin/main, not local main:** after `git fetch origin main`, the local `main` still points at the old commit. `git diff main...HEAD` computes its merge-base including the delta between base→origin/main — i.e., all the other PRs that merged into main while you were working — and the diff gets polluted with code that has nothing to do with your change. Codex then reviews a mix of your code and unrelated recent merges. **Always `git diff origin/main...HEAD`** (or `git rebase origin/main` first and then diff locally). One Phase 2 review had 1/3 of the findings be Codex critiquing a recently-merged unrelated PR — a full round wasted.

### Prompt minimalism (**the most important rule**)

The whole point of a second LLM is **independent judgment**. Every extra sentence in the prompt — "specific things to scrutinize," "I already checked hygiene rule #2," "I chose setdefault here because…" — reduces Codex to confirming your framing instead of finding what you missed.

Empirical: two versions of the same Phase 2 prompt:

| Prompt | Lines | Output |
|---|---|---|
| Editorialized: context + self-summary + "review focus #1-7" + hygiene self-check list | ~150 | Transport interrupted; zero findings (and even without the interruption, it would only have confirmed my framing) |
| **Minimal: diff path + hygiene doc pointer + output format** | **~10** | **2 real P1s** (missing label-set validation + xdist test isolation) |

Both were misses in the 7-item self-review. That's the ROI of independent judgment. So the prompt template **only allows the two minimal shapes below** — anyone adding a "review focus #N" goes back and rewrites.

### Phase 2 prompt template (pre-PR review)

```
Pre-PR review for /tmp/diff.diff.

Context: <one sentence on what audit/finding this PR closes>. Read the diff
and the actual files it touches.

Scope/relevant files (paths only, no analysis):
  <path/to/alternative_impl.py>
  <path/to/related_module.py>

Project hygiene rules: <path to hygiene doc>.

Output: finding table (severity / file:line / issue / suggested fix)
+ single-line PASS / NEEDS_FIXES / BLOCK at the end.
```

Only two blocks may be extended:
- **Context** — one sentence telling Codex the severity baseline (an audit P0 fix has a very different baseline from a minor refactor)
- **Scope/relevant files** — paths only, no explanation, no "review focus." This block prevents the "Codex missed an entire alternative implementation / factory-switch candidate" regression. **Especially when an interface has multiple client implementations** (e.g. a stdlib-based client + an SDK-based client), all alternative impls must be listed in Scope — otherwise Codex only reviews the one that appears in your diff, and the other silently fails open into production.

**Do not write your own analysis, do not list review focus, do not pre-declare what you've already checked.** "Scope" may only list filenames; the moment you add a sentence of explanation, you've regressed.

### Phase 5 prompt template (final review / merge gate)

```
Final review for the full PR diff against origin/main.

Run first (Claude prepares this before invoking codex):
  git fetch origin main
  git diff origin/main...HEAD > /tmp/diff.diff

Project hygiene rules: <path to hygiene doc>.

Output: a single line containing exactly one token (APPROVE or REJECT),
followed optionally by a separate line with one short reason.
The first line is the binding gate — auto-merge parses it as
`head -n 1 | grep -E '^(APPROVE|REJECT)$'`.
```

Phase 5 reviews **the full PR**, not just the latest commit — `git diff origin/main...HEAD` covers every commit since the divergence point. Reviewing only the latest commit silently misses earlier commits (a real pitfall on multi-commit PRs). Always `git fetch` first and base off `origin/main`, for the same reason as Phase 2: local `main` may be stale and will pull other merged PRs' deltas into the diff.

Phase 5 is shorter than Phase 2 — this is the binding decision; Codex reads the diff and judges on its own. Listing Phase 2 findings in the prompt anchors Codex to that set and increases the chance of missing cross-commit issues Phase 2 didn't catch.

**Why allow the optional second line:** in practice Codex's reason is useful for human audit (e.g. `APPROVE\nNo blocking regression found; focused tests pass`). But the reason does not participate in gating — the first line must be a bare token, or the parser fails. **Do not put the reason on the same line as the token** (formats like "APPROVE: <reason>" have been considered and break the grep). It is also fine for Codex to emit a bare token with no reason — that is Phase 5's minimal compliant shape.

---

## Codex progress diagnosis (what to do when you see `Auth(TokenRefreshFailed)`)

`Auth(TokenRefreshFailed("Failed to parse server response"))` appears on stderr fairly often, but **most of the time Codex is still running**. Killing it early wastes a full review round.

**Before killing, classify into two modes:**

| | **Mode A — still progressing (common)** | **Mode B — truly hung** |
|---|---|---|
| stdout (`codex.log`) | Continues to accumulate `exec ... succeeded in Nms` lines | 0 bytes / completely idle |
| `~/.codex/sessions/<today>/` | A new jsonl exists for today; mtime is updating | No file for today (session never started) |
| `ps -p <pid> -o stat,time` | CPU time is accumulating | `S` state + `0:00.x` CPU for minutes |
| Action | **Do nothing** — let the Monitor's `until [ -s file ]` wait it out. A real review typically takes 2-5 minutes before a verdict | `kill <pid>` → `codex logout && codex login` → retry or surface to Owner |

**Quick diagnostic one-liner:**

```bash
# Still emitting exec/succeeded lines = still running
tail -20 /tmp/codex.log | grep -E "exec|succeeded"
# Today's session dir has fresh files = still running (note: sessions are
# bucketed by local date, not UTC — using -u across a UTC/local date boundary
# can point at the wrong dir and falsely flag as hung)
ls -lt ~/.codex/sessions/$(date +%Y/%m/%d)/ 2>/dev/null | head -3
```

**Anti-patterns (each one paid for):**
- Treating the stderr `Auth(...)` line itself as the "kill signal" and immediately falling back to self-review
- Using `Read` / `cat` to poll the `--output-last-message` file — that file is the terminal output and is empty mid-run; only watch Monitor events
- Using `codex.log` (the stdout redirect) to look for the verdict — the verdict goes to `--output-last-message`, not stdout
- Pre-emptively writing "Codex unavailable" prose into the PR body / commit message, and then having Monitor fire the real verdict later and needing to rewrite the PR body

**Lesson: waiting on Monitor is far cheaper than killing and retrying.** Budget Phase 2 for 5 minutes and Phase 5 for 3 minutes of wait — Monitor's `until` predicate handles it.

### 503 outage ("wait, don't reauth")

Codex CLI occasionally returns `Proxy connection failed: HTTP CONNECT failed with status 503` together with intermittent `Auth(TokenRefreshFailed("invalid_grant: Grant not found"))`. **This is a transient outage on the chatgpt.com backend, not a real local-auth failure.**

- **Do not immediately `codex logout && codex login`** — reauth interrupts an active session and does nothing to fix the upstream outage
- Kill the stuck `codex exec`, **wait 10-15 min**, run a smoke (`echo hello | codex exec`), then continue Phase 2/5
- If the PR has to merge immediately, surface to Owner for manual review (using the hygiene self-checks as fallback)
- Only consider full reauth after the outage persists for 24h+

Empirical: one occurrence — 14:34 outage start → 14:55 smoke passes → next Phase 5 ran first-try APPROVE.

---

## Audit trail (every PR must have)

The PR body must include:
1. `## Summary` — 1-3 sentences on what changed and why
2. `## Phase 2 Codex review accept/reject table` — how each finding was handled, with rationale
3. `## Test plan` — checkbox list

After Phase 5 completes: Claude adds a `gh pr comment <pr>` review decision record (round count + verdict + prompt/finding file paths).

Keep locally (for the session):
- `/tmp/codex-*-prompt.md` — sent prompt
- `/tmp/codex-*-finding.md` — model output
- `/tmp/codex-*.log` — full transcript

---

## Real pitfalls (lessons paid for)

| Symptom | Root cause | Fix |
|---|---|---|
| `codex review --base <branch> "..."` errors | `--base` and `[PROMPT]` are mutually exclusive | Use `codex exec` + an explicit git diff |
| Local tests pass / CI fails | A test depended on something not installed locally → SkipTest skipped the bug | Install all CI requirements before push and run the suite once |
| `attempts["n"] == 0` in a mock test | The side_effect callable signature did not match the mocked method's args | Always use `def fn(*args, **kwargs):` |
| `gh pr merge` says "main is used by worktree" | Local `git checkout main` failed in a worktree environment | No impact — verify merge state with `gh pr view --json state` |
| Started polling without waiting for the background task notification | Misused ScheduleWakeup / sleep loop | After starting the background task, do nothing; wait for task-notification |
| Codex missed an entire module / alternative implementation | Codex did not expand scope automatically | Phase 2 prompt adds `Scope/relevant files: <paths only>` block (paths-only — no analysis, no review focus), listing alternative impls / factory-switch candidates by filename; see §"Phase 2 prompt template" |
| Phase 5 repeats Phase 2 findings | The two phase prompts were not distinct enough | Phase 2 catches detail; Phase 5 checks coherence + regression + hygiene + single-token decision |
| Codex and Claude disagree on severity | Different priors on the two sides | Mode 2 challenge protocol resolves this; in Mode 1, escalate to Owner |
| Codex output mirrors my own analysis and finds nothing new | The prompt fed it my framing / "review focus" / self-check list, killing independent judgment | Make the prompt minimal (see §"Prompt minimalism") — diff path + hygiene pointer + output format only |
| `Auth(TokenRefreshFailed)` appears and I assume Codex died | That line is transport-layer reconnection noise; the main loop is usually still running | Check whether stdout is still accumulating `exec ... succeeded` lines + whether `~/.codex/sessions/<today>/` has a fresh jsonl, then decide; see §"Codex progress diagnosis" |
| Both 503 + TokenRefreshFailed → immediately reauth | Backend transient outage misread as real local-auth failure | Wait 10-15 min, smoke-test, then continue; see §"503 outage" |
| Phase 5 verdict not yet out → pre-emptively wrote "Codex unavailable" in the PR body → Monitor later fired with the real verdict → PR body had to be rewritten | Wrote a verdict-state statement before the `--output-last-message` file landed | Do not write a "verdict-state" statement into any persistent artifact (PR body / commit message / code comment) before the Monitor event fires |
| Codex Phase 2 findings include ~1/3 from an already-merged different PR | `git diff main...HEAD` used a stale local main; the merge-base computed against an old commit pulled in deltas from other merged PRs | After `git fetch origin main`, use `git diff origin/main...HEAD`; or `git rebase origin/main` first and then diff |
| Sub-agent worktree used `git checkout -b <branch> main` with a stale local main; main has moved during the agent's work; the agent's push hits a wall of conflicts | Agent started early; main Claude later merged other PRs; the agent never fetched | Require `git fetch origin main && git checkout -b <branch> origin/main` in the agent prompt; after the agent finishes, main Claude rebases proactively + force-with-lease pushes |
| An interface has multiple client implementations (stdlib + SDK), but only the one in the diff got fixed | Phase 2 prompt did not list all implementation paths in Scope; Codex was confined to the diff | Phase 2 Scope must list all alternative impls; keep a project-level "interface → multi-impl path" list to fill into Scope when relevant |
| Phase 2 finding wording not parsed strictly; the fix only half-met the contract; Phase 5 rejected on the same point | "Accept finding" treated as "fixed" without verbatim restatement of the finding's qualifiers | At Phase 3 closeout, list the specific behavioral contracts the change satisfies, matched word-for-word against the finding text; see §"Phase 3 closeout self-check #1" |
| New code has `except OSError: pass`; Phase 5 catches the hygiene violation | No hygiene self-check before push; prompt-engineering lessons have transfer limits | Phase 3 closeout hygiene grep (`git diff $(git merge-base origin/main HEAD)` — use merge-base, not `origin/main...HEAD`, to also scan uncommitted worktree); see §"Phase 3 closeout self-check #2" |
| Changed pre-existing code semantics (made an except branch load-bearing) but did not adjust log level | Semantic uplift blind spot: when adding `state[k] = X`, only looked at new code, not the existing setters | Phase 3 closeout semantic-uplift audit — list all "new if/state assigns" in the diff, re-check input-variable origins + related except branches / log levels; see §"Phase 3 closeout self-check #3" |
| Phase 5 took 4 rounds to APPROVE, but R2/R3 were 1-line hygiene fixes; under a fixed 3-round cap this would have escalated to Owner | Escalation threshold did not consider fix cost; a fixed round cap blocks a 99%-done PR for Owner attention unnecessarily | Make the escalation threshold finding-cost-aware: at round 3 REJECT, estimate fix size; < 10 min you continue on your own; see §"Mode 1 5-phase" Phase 5 block |

---

## Where this fits

Decision rule: run the workflow whenever **"another independent model taking one more look" reduces risk or saves Owner attention**. Representative cases: solo dev / small teams / open-source maintainers without a second reviewer; money / compliance / security-related code; large refactors or DB migrations; pre-launch audits or architecture choices (Mode 2); reviewing code written by another LLM; and changes to this protocol's own README / runbook (the "docs-only" exemption does not apply).

For the full list of use cases and concrete examples, see README: [en](README.md#where-this-fits) / [zh](README.zh.md#适合的场景).

---

## When not to use this

- A 1-line typo / log message / comment — just commit
- Time-critical incident hotfix — use the manual fast path, then audit afterward
- Owner has 100% confidence + the diff is tiny — the workflow has token cost and is not worth it
- Docs-only changes — Phase 2/5 usually add no value

**Exception (must run the full workflow)**: changes to this protocol's own README / runbook / assets. These are the entry points others use to adopt the workflow, so the "docs-only" exemption **does not apply** — a mistake here propagates to every adopter. There was one prior incident where this exception was forgotten — the protocol's own README/runbook was treated as docs-only-exempt and skipped Phase 2/5. Owner caught the meta-error after the fact and we ran the full workflow retroactively. This rule is the lesson from that incident.

---

## Evolution

- **REJECT rate > 30%** on Phase 5 → Phase 2 review is not deep enough, or Phase 5 prompt scope is too broad; tune it
- **Same pitfall hits twice** → add it to this document + memory
- Codex CLI upgrade changes behavior → retest the prompt template

---

## Adoption checklist

1. [ ] Codex CLI installed user-local + login verified (`npm config get prefix` = `$HOME/.local`, `codex exec "hello"` works)
2. [ ] `~/.local/bin` is in PATH (login shells via `~/.profile` get this for free; cron scripts must export explicitly)
3. [ ] Project has a CLI self-update mechanism (cron or supervisor invoking a wrapper script — see the "Self-update recommendation" block above) — auto-rollback on failure
4. [ ] `gh` CLI authenticated + able to create PRs
5. [ ] CI workflow configured to run on PRs
6. [ ] This document placed at the project root (or pasted into a fresh Claude session)
7. [ ] Project-specific prod-critical path blacklist defined
8. [ ] Owner explicitly says: "Codex APPROVE → auto-merge → CI auto-deploys, and I will monitor alerts"
9. [ ] First run uses a low-risk task to validate the whole chain

---

## Post-first-ship review trigger

- Claude reports progress for each phase (commit sha / CI run id / verdict)
- Owner reviews: Were Codex findings real? What was the reject rate? Which pitfalls should be added to this document?
- After adjustments, allow higher-risk tasks into the workflow
