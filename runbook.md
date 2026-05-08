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
Phase 5  Codex final review (codex exec, single-token APPROVE/REJECT)
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
- **Phase 2 and Phase 5 prompts must be different**. Phase 2 catches details; Phase 5 checks PR coherence + regression + project hygiene + a single-token verdict. Repeating the same prompt wastes tokens.

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

## Codex CLI invocation templates

```bash
# Pre-PR review (Phase 2) / Final review (Phase 5):
git diff <base-branch>...HEAD > /tmp/diff.diff   # write to a file; reference it in the prompt
cat > /tmp/codex-prompt.md <<'EOF'
... self-contained prompt (including file: paths + project hygiene rule references + output format spec)
EOF
codex exec --output-last-message /tmp/codex-finding.md - < /tmp/codex-prompt.md \
  > /tmp/codex.log 2>&1 &     # background; wait for task-notification

# Mode 2 parallel analyst: same pattern, while Claude also reads the code in the foreground.
```

**Pitfall**: `codex review --base <branch> "<prompt>"` errors because `--base and PROMPT are mutually exclusive`. **Always use `codex exec`** + an explicit `git diff`.

### Phase 2 prompt template (pre-PR review)

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
- trigger condition: 1-2 lines
- impact: 1-2 lines
- suggested fix: 1-2 lines

End with one of: PASS | NEEDS_FIXES | BLOCK.
```

### Phase 5 prompt template (final review / merge gate)

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
| Codex missed an entire module / alternative implementation | Codex did not expand scope automatically | Phase 2 prompt must explicitly list **all** relevant files, including alternative implementations / factory-switch candidates |
| Phase 5 repeats Phase 2 findings | The two phase prompts were not distinct enough | Phase 2 catches details; Phase 5 checks coherence + regression + hygiene + single-token decision |
| Codex and Claude disagree on severity | Different priors on the two sides | Mode 2 challenge protocol resolves this; in Mode 1, escalate to Owner |

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

**Exception (must run the full workflow)**: changes to this protocol's own README / runbook / assets. These are the entry points others use to adopt the workflow, so the "docs-only" exemption **does not apply** — an error here propagates to every adopter. There was one prior incident where this exception was forgotten — protocol's own README/runbook was treated as docs-only-exempt and skipped Phase 2/5. Owner caught the meta-error after the fact and we ran the full workflow retroactively. This rule is the lesson from that incident.

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
