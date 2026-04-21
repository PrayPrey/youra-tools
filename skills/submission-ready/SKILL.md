---
name: submission-ready
description: Pre-submission stress-test orchestrator that loops /youra-tools:paper-review → /youra-tools:revision-loop → /youra-tools:paper-review until the meta-review recommendation reaches the target grade, plateaus for N cycles, hits --max-cycles, or a sub-skill blocks. No silent success — the orchestrator cannot declare SUCCESS without actually reaching the target grade on disk.
triggers:
  - submission ready
  - submission-ready
  - stress test paper
  - paper stress test
  - pre-submission check
  - polish until submit
  - 제출 준비
  - 제출 직전 검증
  - 스트레스 테스트
  - 논문 마감 루프
argument-hint: "[--paper=<file.tex|file.md|dir>] [--target-grade=Accept|Weak Accept|Borderline] [--max-cycles=<N>] [--converge-window=<N>] [--venue=ACL|NeurIPS|EMNLP|ICML|ICLR|mixed] [--severity=strict|standard|lenient] [--reviewer=architect|critic|code-reviewer|verifier] [--lang=en|ko] [--out=<session-dir>]"
level: 4
---

# Submission Ready

Use this skill when the user has a **completed** paper draft (`.tex` / `.md`) and wants to iterate it to a submission-ready state by looping **peer review → revision → peer review** until the reviewer consensus is strong enough to submit. This is the orchestrator form of the `paper-review` ↔ `revision-loop` ↔ `paper-review` cycle — the tight stress-test that catches silent issues before a venue deadline.

Specialization of the OMC `ralph` persistence loop for the two-sub-skill convergence case: each cycle becomes one PRD story with `passes: false` until both the post-review grade and the revision-loop CLEAN status land on disk. The loop does not exit until a terminal state (`SUCCESS` / `CONVERGED` / `MAX_CYCLES` / `BLOCKED`) fires with concrete evidence.

This skill shares the youra-tools DNA: **evidence-first + loop-until-stable + no silent success.**

## Goal

Turn a completed draft into a submission-ready paper by driving the reviewer consensus up across cycles:

- Each cycle runs a **full** `/youra-tools:paper-review` (mixed venue, strict severity, with claim verification) and captures the meta-review recommendation.
- Each cycle then runs `/youra-tools:revision-loop` with the review bundle as feedback, and only proceeds when that revision-loop hits `CLEAN`.
- The next cycle re-reviews the revised paper and measures the grade delta.
- The orchestrator terminates on one of: **SUCCESS** (target grade reached), **CONVERGED** (plateau over the converge-window), **MAX_CYCLES** (cap hit without success), or **BLOCKED** (a sub-skill refused to complete).
- The orchestrator never reports `SUCCESS` unless the final meta-review recommendation is **actually** at or above `--target-grade`. Cycle-over-cycle improvement alone is not enough.

## When to invoke

- The user has a draft they plan to submit soon and wants an honest stress-test pass: what would real reviewers say, can it be fixed in-scope, and how close to acceptance is it today.
- The user wants to iterate until a target grade (e.g., at least `Weak Accept`) is reached, without babysitting the `paper-review` → `revision-loop` handoffs manually.
- The user wants a reproducible, on-disk audit trail: per-cycle reviewer bundles, per-cycle revision sessions, scored convergence history.

Do **not** invoke for:

- A first-pass draft with missing sections (use `methodology-writer` / `experiments-writer` first).
- A paper whose fundamental contribution is in question (use `gap-finder` to check novelty first — this skill polishes, it does not re-frame).
- A single one-off revision request (use `revision-loop` directly).
- A single one-off review (use `paper-review` directly).
- A from-scratch draft (use `autopilot` or a writer skill).

## Arguments

Parse the invocation string. All arguments are optional — ask the user once with concrete candidates when missing. Never silently default to an empty or wrong target.

| Flag | Meaning | Default resolution |
|------|---------|--------------------|
| `--paper=<path>` | Completed draft to stress-test. `.tex` or `.md` file. | Search for newest `*revised*.tex` / `*.tex` / `*.md` in the workspace root. If exactly one is an obvious winner, use it and echo the chosen path. If zero or multiple candidates exist, prompt via `AskUserQuestion` (candidates as options, open question if none). Never proceed without a confirmed paper. |
| `--target-grade=<grade>` | Minimum acceptable meta-review recommendation for SUCCESS. One of: `Accept`, `Weak Accept`, `Borderline`, `Weak Reject`, `Reject`. | `Weak Accept`. Echo the default so the user can override. |
| `--max-cycles=<N>` | Hard cap on review→revise→review cycles. Each cycle is expensive (claim verification + multi-reviewer + revision loop), so keep modest. | `3` |
| `--converge-window=<N>` | Number of consecutive cycles with no grade improvement that triggers `CONVERGED` exit. | `2` |
| `--venue=<v>` | Passed through to `/youra-tools:paper-review`. `ACL` / `NeurIPS` / `EMNLP` / `ICML` / `ICLR` / `mixed`. | `mixed` (ACL + NeurIPS + EMNLP, one reviewer each). |
| `--severity=<s>` | Passed through to `/youra-tools:paper-review`. `strict` / `standard` / `lenient`. | `strict` |
| `--reviewer=<agent>` | Passed through to `/youra-tools:revision-loop` as the per-issue reviewer gate. `architect` / `critic` / `code-reviewer` / `verifier`. | `architect` |
| `--lang=<en\|ko>` | Output language for the orchestrator's own report and prompts. Sub-skills inherit their own `--lang` defaults from the paper file. | Inspect the paper file; fallback `en`. |
| `--out=<path>` | Session directory for this orchestrator run. | `.omc/submission-ready/{timestamp}-{slug(paper)}/` |

## Session artefacts

```
.omc/submission-ready/{timestamp}-{slug(paper)}/
  state.json              # resolved arguments + current cycle index + terminal state (null until done)
  scores.json             # per-cycle { cycle, pre_review_grade, post_review_grade, ladder_delta, review_bundle, revision_session }
  cycle-{k}/
    review-bundle-path    # file containing absolute path to .omc/reviews/{timestamp}-{paper-stem}/ produced this cycle
    revision-session-path # file containing absolute path to .omc/revisions/{timestamp}-{slug}/ produced this cycle (absent for the final re-review-only cycle)
    verdict.json          # parsed meta-review recommendation + per-reviewer scores + derived ladder index
  report.md               # final human-readable bundle (terminal state, cycle table, residual blockers, next-step guidance)
```

## Workflow

### Phase 0 — Resolve arguments and set up the session

1. Parse flags. For each missing flag, apply the default resolution in the Arguments table above. When prompting the user via `AskUserQuestion`, do it **once per missing flag, serially** — do not fan out a multi-question turn.
2. Resolve `--paper` first (blocking). The paper file must exist and be non-empty. If user-supplied and missing, stop, list candidate `*.tex` / `*.md` via `Glob`, and re-prompt — do not guess.
3. Echo every resolved argument (resolved or defaulted) in the next message so the user can redirect before the first expensive cycle starts.
4. Create the session directory `--out`. Write `state.json` with the resolved args, `current_cycle: 0`, and `terminal_state: null`.
5. Initialize `scores.json` as an empty array.

### Phase 1 — Baseline peer review (cycle 1 pre)

1. Invoke `/youra-tools:paper-review` via the Skill tool with `--paper=<resolved>`, `--venue=<resolved>`, `--severity=<resolved>`, `--mode=full`, `--verify=true`, and `--lang=<resolved>`.
2. When the sub-skill completes, read its `report.md` and `meta-review.md` from the session directory it reports back.
3. Parse the meta-review recommendation string — expected values: `Accept` | `Weak Accept` | `Borderline` | `Weak Reject` | `Reject`. Map to ladder index: `Reject`=0, `Weak Reject`=1, `Borderline`=2, `Weak Accept`=3, `Accept`=4.
4. Persist to `cycle-1/`: the review bundle path, the parsed verdict (`verdict.json`), and append a row to `scores.json` with `pre_review_grade=<parsed>`, `post_review_grade=null` (not yet revised), `ladder_delta=null`.
5. If the recommendation is already `>= --target-grade`, set terminal state to `SUCCESS`, skip to Phase 6, and record that cycle 1 required no revision.

### Phase 2 — Apply feedback (cycle k revision)

1. Invoke `/youra-tools:revision-loop` via the Skill tool with `--feedback=<cycle-k review bundle absolute path>`, `--target=<the paper file>`, `--reviewer=<resolved>`, `--lang=<resolved>`.
2. Wait for `revision-loop` to terminate. It returns either `CLEAN` (all issues addressed with on-disk PASS verdicts) or `BLOCKED` (max-rounds hit or unresolved blocker).
3. If `BLOCKED`: set orchestrator terminal state to `BLOCKED`, record the revision-loop session path for the user, and jump to Phase 6. Do not re-review a paper whose revision pass stopped short.
4. If `CLEAN`: persist the revision-loop session path to `cycle-k/revision-session-path` and continue.
5. Mid-flow prompts: if revision-loop raises its own `AskUserQuestion` (conflicting reviewer items, missing bib entry, ambiguous target), surface that prompt to the user **verbatim** and forward the response to revision-loop — do not pre-answer or silently pick.

### Phase 3 — Re-review (cycle k post)

1. Invoke `/youra-tools:paper-review` again on the revised paper with the same `--venue` / `--severity` / `--verify=true` / `--mode=full` settings.
2. Parse the new meta-review recommendation into a ladder index exactly as in Phase 1.
3. Update `cycle-k` in `scores.json`: `post_review_grade=<parsed>`, `ladder_delta=<post-index minus the pre-index of this cycle>`.
4. The post-review bundle path for cycle k becomes the `pre_review_grade` of cycle k+1 (since it is the new starting point).

### Phase 4 — Convergence check

Evaluate exit conditions **in this order**, stopping at the first match:

1. **SUCCESS**: `post_review_grade >= --target-grade` (ladder index comparison). Terminal state = `SUCCESS`. Go to Phase 6.
2. **BLOCKED**: already handled in Phase 2. (Listed here for completeness — the orchestrator never reaches Phase 4 in the BLOCKED branch.)
3. **CONVERGED**: at least `--converge-window` cycles completed, and every ladder index in the last `--converge-window` cycles is `<=` the ladder index of the cycle immediately before that window. "No improvement" means **same or lower** grade — polite cosmetic edits that leave the recommendation unchanged count as no improvement. Terminal state = `CONVERGED`. Go to Phase 6.
4. **MAX_CYCLES**: current cycle index `== --max-cycles` and no SUCCESS has fired. Terminal state = `MAX_CYCLES`. Go to Phase 6.
5. Otherwise: increment cycle index, go back to Phase 2 with the revised paper as the new input.

### Phase 5 — (reserved)

Reserved for future hooks (e.g., a human-in-the-loop confirmation between cycles). Not used by default.

### Phase 6 — Report and terminate

1. Write `report.md` at the session root containing:
   - Resolved arguments (`--paper`, `--target-grade`, `--max-cycles`, `--converge-window`, `--venue`, `--severity`, `--reviewer`, `--lang`).
   - Terminal state (`SUCCESS` / `CONVERGED` / `MAX_CYCLES` / `BLOCKED`) with one-sentence rationale.
   - Per-cycle table: `cycle k | pre-grade | revision-loop status | post-grade | ladder Δ | review-bundle path | revision-session path`.
   - Outstanding blockers: copy the unresolved-blocker list from the **final** paper-review meta-review. Every item keeps its reviewer ID and line citation.
   - Next-step guidance:
     - On `SUCCESS`: recommend a final camera-ready pass, list non-blocking suggestions from the last review.
     - On `CONVERGED`: the grade plateaued — explain which weakness kept recurring and suggest an out-of-scope fix (new experiment, reframing, additional baseline) that only the user can do.
     - On `MAX_CYCLES`: similar to CONVERGED but note the orchestrator had not converged and suggest raising `--max-cycles` only if the grade was still trending up.
     - On `BLOCKED`: point at the sub-skill session that blocked, restate its blocking reason, and describe what user input unblocks it.
2. Print a terminal summary (`<= 25` lines): final terminal state, start→end grade delta (`Reject → Borderline`, etc.), cycles used, path to `report.md`.
3. On `SUCCESS`: run `/oh-my-claudecode:cancel` for clean state cleanup. On any other terminal state: **do not** cancel — leave the session on disk so the user can inspect and resume.

## Rules

- **The target grade is the bar.** The orchestrator cannot report `SUCCESS` unless the final post-review recommendation is `>= --target-grade`. Cycle-over-cycle improvement without hitting the target is `CONVERGED` or `MAX_CYCLES`, never `SUCCESS`.
- **No silent success.** If paper-review could not produce a parseable meta-review recommendation (MCP down, PDF extraction failed, bundle incomplete), the terminal state is `BLOCKED`, not `SUCCESS`. Same rule as the rest of youra-tools.
- **Sub-skill outputs are the evidence.** Every cycle's grade must trace to a specific `meta-review.md` on disk; every cycle's revision must trace to a specific revision-loop session with `CLEAN` status. Orchestrator-side summaries do not substitute for on-disk sub-skill bundles.
- **Serial cycles.** Phase 2 depends on Phase 1, Phase 3 depends on Phase 2 — no cycle-internal parallelism. Two different cycles of the same session also never run in parallel.
- **Interactive prompts flow through.** When a sub-skill raises an `AskUserQuestion`, forward it verbatim to the user; do not pre-answer, skip, or merge with other questions.
- **Revision-loop must hit CLEAN.** A `BLOCKED` from revision-loop halts the orchestrator; never re-review a paper whose revision pass stopped short — the partial state would inflate or deflate the grade misleadingly.
- **The boulder never stops.** If a hook injects `The boulder never stops`, continue iterating until a terminal state actually fires — do not declare done early on hook noise.
- **Do not tamper with sub-skill defaults silently.** If the user passes `--venue=ACL`, pass it through literally to each cycle's paper-review. Do not "improve" it to `mixed` even if `mixed` would be stronger.
- **Persist state every phase.** After each phase, update `state.json` and (if grades changed) `scores.json` so a crashed run can be resumed without losing cycle history.

## Failure modes to guard against

- **Silent-success-by-improvement.** Orchestrator reports `SUCCESS` because the grade went up from `Weak Reject` to `Borderline`, even though `--target-grade=Weak Accept`. Mitigation: Phase 4 exit ordering — SUCCESS requires `>= --target-grade`, not "grade improved".
- **Phantom-cycle.** Orchestrator writes `scores.json` rows without actually invoking the sub-skills on disk. Mitigation: each cycle row requires a real `review-bundle-path` and (except for the pre-revision baseline) a real `revision-session-path` pointing to on-disk session directories with valid contents.
- **Grade parsing drift.** Paper-review changes its recommendation vocabulary (new label, different capitalization). Orchestrator misparses and reports the wrong ladder index. Mitigation: whitelist the five allowed strings; any unrecognized string triggers `BLOCKED` with the raw string in the report.
- **Convergence false-positive.** The paper went up by one ladder step between cycle 1 and cycle 2, flat between cycle 2 and cycle 3 — with `--converge-window=2`, cycles 2 and 3 form a plateau even though cycle 3 was still better than the baseline. Mitigation: the converge window compares the last `N` cycles against the cycle **immediately before the window**, not against the baseline.
- **Sub-skill-unavailable drift.** `/youra-tools:paper-review` or `/youra-tools:revision-loop` is not installed. Orchestrator silently skips. Mitigation: Phase 1 probes for both skills on first invocation; if either is missing, exit immediately with a setup-error terminal state and instructions to install `youra-tools@>=0.4.0`.
- **Language drift.** The paper is Korean, orchestrator reports are English, but the final report.md mixes both. Mitigation: `--lang` controls only orchestrator-side prose; sub-skills inherit their own language rules from the paper file.

## Examples

### Good — SUCCESS in two cycles

User: `/youra-tools:submission-ready --paper=YouRA_0419_revised.tex --target-grade="Weak Accept"`

What happens: Cycle 1 paper-review returns `Borderline` (index 2). Revision-loop applies all reviewer blockers and exits CLEAN. Cycle 2 paper-review returns `Weak Accept` (index 3). Phase 4 detects SUCCESS (`3 >= 3`). Report states SUCCESS, cycles used = 2, delta = `Borderline → Weak Accept`. `/oh-my-claudecode:cancel` runs.

### Good — CONVERGED with actionable next step

User: `/youra-tools:submission-ready --paper=YouRA_0419_revised.tex --target-grade=Accept --max-cycles=4 --converge-window=2`

What happens: Cycles 1–4 return `Borderline`, `Weak Accept`, `Weak Accept`, `Weak Accept`. Target is `Accept` (index 4), so SUCCESS never fires. Cycles 2–4 form a plateau against cycle 1's baseline → `CONVERGED` on cycle 4. Report names the recurring blocker (e.g., "single-judge evaluation") that kept the grade below `Accept` and recommends a multi-judge ablation experiment the orchestrator cannot run on its own.

### Bad — silent-success-by-improvement

The orchestrator sees the grade went up from `Reject` to `Weak Reject` between cycles and reports `SUCCESS` because "the paper improved".

Why bad: Violates the core rule. `Weak Reject` is still below `--target-grade=Weak Accept`. The correct terminal state depends on whether cycles remain (keep going) or not (MAX_CYCLES). This is exactly the antipattern the skill exists to prevent.

### Bad — skipping BLOCKED revision

Cycle 2's revision-loop hits its own `--max-rounds` with 3 blockers remaining and returns `BLOCKED`. The orchestrator ignores this and invokes paper-review on the partially-revised paper anyway.

Why bad: Violates Phase 2 rule #3. A partially-revised paper distorts the next grade — some blockers are fixed, others remain — so the delta is meaningless. The correct behaviour is `BLOCKED` terminal state with the revision-loop session path surfaced for the user.

## Output

- Session bundle at `.omc/submission-ready/{timestamp}-{slug(paper)}/` with `state.json`, `scores.json`, per-cycle subdirectories linking to sub-skill bundles, and `report.md`.
- The paper file on disk is updated in place by each cycle's revision-loop (not by this orchestrator directly).
- Terminal summary (`<= 25` lines) with final terminal state, grade delta, cycle count, path to `report.md`.

## Related skills

- `/youra-tools:paper-review` — the sub-skill this orchestrator invokes each cycle for peer-review + claim verification.
- `/youra-tools:revision-loop` — the sub-skill this orchestrator invokes each cycle to apply the review feedback until CLEAN.
- `/youra-tools:methodology-writer` — upstream: use it first if the Methodology section is thin before stress-testing (this skill does not rewrite missing sections from scratch).
- `/youra-tools:experiments-writer` — upstream: use it first to reconcile numbers and interpretations with the results files.
- `/oh-my-claudecode:ralph` — the generic PRD-driven persistence loop this orchestrator specializes for the two-sub-skill cycle case.
- `/oh-my-claudecode:cancel` — called on `SUCCESS` for clean state cleanup; not called on `CONVERGED` / `MAX_CYCLES` / `BLOCKED` so the user can inspect and resume.
