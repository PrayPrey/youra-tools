---
name: revision-loop
description: Take feedback or a list of fix-items for an academic document (LaTeX/Markdown), decompose into discrete issues, then loop edit → verify until a reviewer agent signs off that every issue is resolved — no silent success, no self-approval.
triggers:
  - revision loop
  - revision-loop
  - revise until done
  - feedback loop
  - fix feedback
  - apply feedback
  - 수정 루프
  - 피드백 반영
  - 반복 수정
  - 리비전 루프
  - 피드백 적용
argument-hint: "[--feedback=<path|text|review-dir>] [--target=<file.tex|file.md|dir>] [--lang=en|ko] [--max-rounds=<N>] [--reviewer=architect|critic|code-reviewer|verifier] [--out=<session-dir>] [--allow-bib=<custom.bib>] [--scope=text-only|all]"
level: 3
---

# Revision Loop

Use this skill when the user has **feedback** or a **list of things to fix** in an academic document (a `.tex` paper, a `.md` writeup, a reviewer report, or inline prose) and wants the document revised iteratively until every issue is verifiably resolved. The skill is a specialization of the OMC `ralph` loop for the feedback-driven revision case: each fix-item becomes one PRD story with `passes: false` until a reviewer agent approves it, and the loop does not exit until every story flips to `passes: true` (or `--max-rounds` is hit and the remaining blockers are surfaced honestly).

This skill shares the youra-tools DNA: **evidence-first + loop-until-stable + no silent success.**

## Goal

Turn concrete feedback into a verified, issue-free revision of the target document:

- Every discrete issue in the feedback becomes one tracked story.
- Every story is fixed with a minimal, surgical edit (`Edit` with exact `old_string`/`new_string`, not whole-file `Write`).
- Every fixed story is verified by an independent reviewer agent against a specific, per-issue acceptance criterion — the skill never self-approves.
- If the reviewer rejects, the skill fixes the blocking items and re-verifies; it does not silently mark "done".
- If `--max-rounds` is hit with unresolved issues, the skill stops and reports the remaining blockers — **it never fakes success**.

## When to invoke

- The user has a reviewer report (e.g., from `/youra-tools:paper-review` under `.omc/reviews/.../reviewer-*.md`) and wants each weakness systematically addressed in the paper.
- The user pastes a bullet list of fixes ("rephrase the abstract, fix the typo on page 3, the Table 2 caption contradicts Section 4.2, add a limitation about the 10-task subset") and wants the document revised until each one is confirmed done.
- The user received real peer-review feedback in prose form and wants a hallucination-proof iterative revision.
- The user wants a loop they can walk away from — one that only stops when it is *actually* done or when a human has to intervene.

Do **not** invoke for: a single one-line change (delegate directly to an executor), a full-section rewrite (use `oh-my-claudecode:autopilot`), a from-scratch draft (use `methodology-writer` or `experiments-writer`), or a build-error debug (use `oh-my-claudecode:debugger`).

## Arguments

Parse the invocation string. All arguments are optional — ask the user once with concrete candidates when missing. Never silently default to an empty feedback set.

| Flag | Meaning | Default resolution |
|------|---------|--------------------|
| `--feedback=<path\|text\|review-dir>` | The feedback to apply. Accepts: (a) raw inline text, (b) a `.md`/`.txt`/`.json` file path, (c) a reviewer bundle directory such as `.omc/reviews/{timestamp}-{stem}/` — in which case read every `reviewer-*.md` and the `meta-review.md` inside. | Prompt once via `AskUserQuestion` if missing. Do not proceed with an empty feedback set. |
| `--target=<file\|dir>` | The document to revise. A single `.tex` / `.md` / `.txt` file, or a directory (for multi-file revisions). | Resolve in order: user-named file (ALWAYS wins), then most-recently-modified `*revised*.tex` in the workspace root, then most recent `*.tex`, then most recent `*.md`. If multiple plausible candidates exist, list them and ask. |
| `--lang=en\|ko` | Output language for the progress log and reviewer prompts. Edits preserve the target file's existing language. | Inspect the target file; fallback `en`. |
| `--max-rounds=<N>` | Maximum edit → verify iterations across the whole PRD (not per issue). | `10` |
| `--reviewer=<agent>` | Reviewer agent for the verification gate: `architect` (Opus, default, academic rigour), `critic` (Opus, harsher), `code-reviewer` (Sonnet, faster for markup-level checks), `verifier` (evidence-focused). | `architect` |
| `--out=<path>` | Session directory for the revision bundle. | `.omc/revisions/{timestamp}-{slug(target)}/` |
| `--allow-bib=<custom.bib>` | Bib file that new `\cite{...}` keys must exist in, for `.tex` targets. | Auto-detect the nearest `*.bib` next to the target; if none, disable citation additions. |
| `--scope=text-only\|all` | Which feedback items to act on. `text-only` (default): only items that can be fixed by editing the document — text rewrites, table/caption fixes, citation swaps, limitations additions, reinterpretation of existing results. Items requiring **new experiments, new training/inference runs, new datasets, new ablations, or new baselines** are deferred to `report.md` and `deferred.json`, not applied. `all`: legacy behaviour — every issue is in scope, including those requiring new runs. | `text-only` |

## Session artefacts

```
.omc/revisions/{timestamp}-{slug(target)}/
  state.json              # resolved arguments + target file snapshot (pre-edit hash)
  feedback.md             # normalized feedback verbatim (copied or concatenated)
  issues.json             # in-scope issue list — this IS the PRD (filtered by --scope)
  deferred.json           # issues excluded by --scope=text-only (requires new experiments)
  progress.txt            # append-only log: per-issue edits, reviewer verdicts, learnings
  verification/
    round-{k}/
      verdicts.json       # per-issue PASS/FAIL from the reviewer this round
      summary.md          # round summary
  report.md               # final human-readable bundle (rounds used, verdict, residual items)
```

## Workflow

### Phase 0 — Resolve arguments and set up the session

1. Parse flags. If `--feedback` is missing, ask the user once via `AskUserQuestion`: *"Paste the feedback, or give me a file / reviewer-bundle path."* Accept any of: inline text, a file path, a reviewer directory.
2. Resolve `--target`:
   - If the user named a file (anywhere in the feedback or in the flag): use it EXACTLY. Do not auto-resolve to a "newer" file.
   - If the target does not exist, stop and list available `*.tex` / `*.md` via `Glob`.
   - Otherwise fall back to the priority order in the Arguments table and **echo the chosen path** in the next message so the user can redirect.
3. Create the session directory `--out`. Snapshot the target file's current SHA to `state.json` (so later rounds can diff against the starting point).
4. Copy/concatenate the feedback into `feedback.md`.

### Phase 1 — Ingest feedback and decompose into issues

1. Load `feedback.md`. If `--feedback` pointed to a reviewer bundle directory, read every `reviewer-*.md` plus `meta-review.md`.
2. Decompose into **discrete, numbered issues**. For each issue emit:
   ```json
   {
     "id": "RL-I-001",
     "description": "Abstract leads with motivation instead of benchmark result",
     "location_hint": "\\begin{abstract} in YouRA_0419_revised.tex",
     "severity": "major | minor | blocker",
     "acceptance_criterion": "Abstract's first sentence names the top-line benchmark number; motivation is moved to sentence 2 or later",
     "source": "reviewer-ACL.md W1  |  user-inline-bullet-3  |  meta-review recommendation #2",
     "requires_new_experiment": false,
     "experiment_reason": null,
     "passes": false
   }
   ```
3. **Classify each issue for `--scope` filtering.** For every decomposed issue, set `requires_new_experiment: true` if fixing it would require any of:
   - a new training run, fine-tuning pass, or inference sweep
   - a new dataset, new evaluation split, or new benchmark
   - a new ablation, new baseline comparison, or new hyperparameter sweep
   - any new numerical result that is not already present in the existing experiment artefacts on disk

   Otherwise set `requires_new_experiment: false` (text rewrites, caption/table fixes, citation swaps, reinterpretation of existing numbers, limitations additions, structural reorganisation, typos). When `true`, also set `experiment_reason` to a one-line justification (e.g., `"reviewer asks for accuracy on a held-out CIFAR-100 split — no such run exists"`).

4. **Apply `--scope` filter.**
   - If `--scope=text-only` (default): split the decomposed list into two files. Items with `requires_new_experiment: true` go to `deferred.json` (not the PRD); items with `requires_new_experiment: false` go to `issues.json`.
   - If `--scope=all`: write everything to `issues.json` (legacy behaviour).
   - In either case, **echo the split in the next message** so the user can redirect: `"Decomposed 12 issues — 9 in-scope, 3 deferred (need new experiments): [ids]. Continue, or rerun with --scope=all?"`.
5. Write the in-scope list to `issues.json`. **This is the PRD.** Each issue corresponds to exactly one story; the loop does not exit until all stories have `passes: true`. Deferred issues do **not** block CLEAN exit, but they MUST be reported in `report.md`.
6. If two issues conflict (reviewer A wants X removed, reviewer B wants X expanded), surface the conflict to the user via `AskUserQuestion` and let them pick — do **not** silently choose one side.
7. If the decomposition produces zero in-scope issues (everything was deferred under `text-only`), stop, write `report.md` with the deferred list, and exit `BLOCKED-NO-INSCOPE` — do not invent text-only issues to justify running.

### Phase 2 — Pick the next pending issue

1. Read `issues.json`. Select the highest-severity issue with `passes: false` (ties broken by `id` order).
2. Snapshot the current target file state for later diffing.

### Phase 3 — Apply the fix (delegate to executor)

1. Delegate the edit to an executor agent:
   - `Task(subagent_type="oh-my-claudecode:executor", model="sonnet", ...)` for standard rewrites, typo fixes, citation swaps, caption edits.
   - `Task(subagent_type="oh-my-claudecode:executor", model="opus", ...)` for argument-level rewrites, additions that must stay consistent with other sections, cross-section changes.
   - If OMC is not installed, use direct `Edit` calls from the parent context; the model tiering rule still applies to the level of care taken.
2. The executor prompt MUST include: (a) the issue verbatim, (b) the `location_hint`, (c) the `acceptance_criterion`, (d) the hard constraints below.
3. **Hard constraints for every edit:**
   - Use `Edit` with exact `old_string`/`new_string`. Never rewrite whole files with `Write`.
   - Do not touch passages outside the declared `location_hint`. Incidental changes, if unavoidable, must be appended to `progress.txt` under the current issue.
   - Do not introduce `% TODO`, `<!-- TODO -->`, or placeholder markers.
   - **For `.tex` targets:** every `\begin{env}` must retain its matching `\end{env}`; braces must balance; math delimiters (`$...$`, `\[...\]`) must be preserved; any new `\cite{key}` must reference a key that already exists in `--allow-bib`. If the feedback asks for a citation whose key is not in the .bib and no BibTeX entry was provided, stop and ask the user for the entry — do not fabricate one.
   - **For `.md` targets:** preserve YAML front-matter, heading levels, existing link anchors, and code-fence language tags.
   - Preserve bilingual comments or Korean/English interleaving if present in the source.
4. Append an entry to `progress.txt`: the issue id, the file/line range edited, and a one-line description of the change.

### Phase 4 — Verify (looped part)

1. Invoke the reviewer agent with a concrete per-issue checklist:
   - `--reviewer=architect` → `Task(subagent_type="oh-my-claudecode:architect", model="opus", ...)`
   - `--reviewer=critic` → `Task(subagent_type="oh-my-claudecode:critic", model="opus", ...)`
   - `--reviewer=code-reviewer` → `Task(subagent_type="oh-my-claudecode:code-reviewer", model="sonnet", ...)`
   - `--reviewer=verifier` → `Task(subagent_type="oh-my-claudecode:verifier", ...)`
2. The reviewer prompt MUST pass:
   - The issue `description` and `acceptance_criterion` verbatim.
   - The diff (changed `old_string` → `new_string`) plus ±20 lines of surrounding context.
   - The target document's current state on disk (not an in-memory draft).
   - The following checklist, to be answered per item with PASS / FAIL + evidence:
     1. Does the change actually satisfy the issue's `acceptance_criterion`? (No phantom fixes.)
     2. Is the edit confined to the declared `location_hint`? (No scope creep.)
     3. Is document syntax still valid? (For `.tex`: braces, environments, math delimiters. For `.md`: front-matter, headings, links.)
     4. Do any new `\cite{...}` keys resolve inside `--allow-bib`? (N/A if target is `.md`.)
     5. Were any new issues introduced elsewhere? (Regressions are a FAIL.)
3. The reviewer returns `PASS` or `FAIL` with a bulleted list of **blocking** items and a separate list of **non-blocking** suggestions. Persist the verdict to `verification/round-{k}/verdicts.json`.
4. **No self-approval.** A reviewer call that was never actually made, a missing response, or a verdict without evidence counts as FAIL. The skill MUST NOT mark an issue complete without an explicit PASS verdict on disk.

### Phase 5 — Branch on verdict

- **PASS** → Set `passes: true` for this issue in `issues.json`. Append a success line to `progress.txt`. Go to Phase 6.
- **FAIL** → Leave `passes: false`. Append the blocking reasons to `progress.txt`. Return to Phase 3 with a refined executor prompt that addresses only the blocking items — do not rewrite passages the reviewer did not flag.
- If the same issue is rejected 3 times in a row with related reasons, stop and escalate to the user with the rejection history — this signals the issue's `acceptance_criterion` may be infeasible as stated.

### Phase 6 — Check PRD completion

1. Read `issues.json`. Are ALL stories `passes: true`?
2. **Not yet** → loop back to Phase 2 for the next pending issue.
3. **All complete** → go to Phase 7.
4. If `--max-rounds` has been hit and stories remain incomplete, go to Phase 7 with the **remaining-blockers** branch — do not mark the run successful.

### Phase 7 — Report

1. Write `report.md`:
   - Resolved arguments (`--feedback`, `--target`, `--lang`, `--reviewer`, `--max-rounds`, `--allow-bib`, `--scope`).
   - Rounds used, total issues, counts of passed / remaining / escalated / deferred.
   - Per-issue table (in-scope): id, description, final verdict, file/line range of the final edit.
   - **Deferred — requires new experiment** (REQUIRED section, even if empty): each entry from `deferred.json` with id, description, `experiment_reason`, source, and a suggested follow-up (e.g., "run with `/youra-tools:experiments-writer` after the new sweep, or rerun with `--scope=all`"). When `--scope=all` was used this section is empty by design.
   - Outstanding non-blocking reviewer suggestions (separate list).
   - For the remaining-blockers branch: each unresolved issue with the last rejection reason and a suggested next step.
2. Print a ≤20-line terminal summary: session path, rounds used, passed/remaining counts, final verdict (`CLEAN` if all PASS, `BLOCKED` if any remain at max-rounds).
3. On `CLEAN`: run `/oh-my-claudecode:cancel` for clean state cleanup. On `BLOCKED`: do **not** cancel — leave the session on disk so the user can resume.

## Rules

- **Feedback is the source of truth.** Every story in `issues.json` must trace to a specific line/paragraph in `feedback.md` (or a specific reviewer report). No invented issues, no scope expansion beyond what the feedback actually asked for.
- **No phantom fixes.** A reviewer PASS requires concrete evidence that the edit satisfies the `acceptance_criterion` — not a vibe check.
- **No silent success.** If verification could not run (agent unavailable, network down), say so in `report.md` and exit with `BLOCKED`. Never declare `CLEAN` without an actual reviewer PASS.
- **User choice ALWAYS wins on target.** If the user names a file, use it exactly. Do not apply the default priority and do not silently fall back to a different file.
- **Minimal surgical edits only.** `Edit` with exact `old_string`/`new_string`. No whole-file `Write`. No touch-ups to passages the feedback did not mention.
- **Syntax integrity is a hard gate.** Broken `.tex` (missing `\end`, unbalanced braces, orphan math delimiters) or broken `.md` (corrupted front-matter, broken link anchors) is an automatic reviewer FAIL.
- **Bib integrity for LaTeX.** Every new `\cite{key}` must resolve in `--allow-bib`. Unknown keys stop the loop until the user supplies the BibTeX entry.
- **Language preservation.** Keep the target file's language; only `--lang` controls the reviewer prompt and progress log language. For bilingual documents, keep Korean/English pairing intact.
- **Apply only blocking reviewer issues between rounds.** Defer non-blocking suggestions to `report.md` rather than ballooning the edit scope.
- **Default scope is text-only.** Without `--scope=all`, never run a fix that requires a new experiment, training run, dataset, ablation, or baseline. Such issues go to `deferred.json` and `report.md`'s "Deferred — requires new experiment" section. The skill never silently fabricates numbers to satisfy an experiment-class issue.
- **The boulder never stops.** If a hook injects `The boulder never stops`, continue iterating — do not declare done unless every story is `passes: true` with an on-disk reviewer PASS.

## Failure modes to guard against

- **Phantom-fix.** The edit looks plausible but does not actually satisfy the issue's `acceptance_criterion`. Mitigation: reviewer must quote the edited text and compare it against the criterion verbatim.
- **Scope creep beyond the feedback.** The executor "improves" neighbouring paragraphs the feedback never mentioned. Mitigation: hard constraint in Phase 3; reviewer checklist item #2; progress.txt records incidental changes.
- **Reviewer bypass / self-approval.** The skill marks a story complete without an actual reviewer call, or accepts a hand-waved response. Mitigation: Phase 4 requires verdicts.json on disk with evidence; Phase 5 treats missing/empty verdicts as FAIL.
- **Document syntax corruption.** LaTeX braces/environments break, or Markdown front-matter/links get mangled. Mitigation: Phase 3 hard constraints + Phase 4 checklist item #3.
- **Citation fabrication.** A new `\cite{key}` points at a nonexistent bib entry. Mitigation: Phase 3 checks `--allow-bib` before accepting the edit; unknown keys stop the loop for user input.
- **Reviewer rubber-stamp.** The reviewer returns PASS without quoting evidence. Mitigation: Phase 4 requires per-item PASS/FAIL with evidence; missing evidence treated as FAIL.
- **Max-rounds masquerade.** `--max-rounds` is hit with blockers remaining and the skill reports `CLEAN`. Mitigation: Phase 6 and Phase 7 explicitly branch on the remaining-blocker case; `report.md` has a required `BLOCKED` section.
- **Silent language drift.** The revision loop accidentally converts Korean prose to English (or vice versa) to match the reviewer prompt language. Mitigation: edits preserve the target file's language; only the progress log and reviewer prompt follow `--lang`.
- **Fabricated experiment results.** Under `--scope=text-only` the executor invents numbers for an experiment-class issue to make a reviewer pass. Mitigation: experiment-class issues never enter `issues.json`; they live in `deferred.json` only. The reviewer checklist for in-scope issues includes "did this edit add or modify any numerical result not present elsewhere in the document or experiment artefacts?" — yes counts as FAIL.
- **Misclassification.** A genuine experiment-class issue (e.g., "report variance over 5 seeds") is mistakenly marked `requires_new_experiment: false` and the executor invents the variance. Mitigation: when in doubt, classify as `true`; the user can override by rerunning with `--scope=all` or by editing `deferred.json` → `issues.json`.

## Examples

### Good — reviewer-bundle feedback

User: `/youra-tools:revision-loop --feedback=.omc/reviews/2026-04-15-YouRA_0419_revised/ --target=YouRA_0419_revised.tex --reviewer=architect`

What happens: The skill reads every `reviewer-*.md` and `meta-review.md` from the review bundle, decomposes each weakness (W1, W2, ...) into one issue with a precise `acceptance_criterion`, loops per issue through executor → architect, and exits `CLEAN` only when every issue has a PASS verdict on disk.

### Good — inline Korean bullet feedback

User: `/youra-tools:revision-loop --feedback="초록을 벤치마크 수치로 시작; 3.2절의 Table 2 캡션이 4.2절과 모순됨을 수정; 10-task subset 한계를 Limitations에 추가" --target=YouRA_0416_korean.tex`

What happens: The skill decomposes into three issues (abstract lead, Table 2 / Section 4.2 consistency, limitations addition), each with a Korean-rooted `acceptance_criterion`. Edits stay in Korean; the progress log follows `--lang` (defaulting to `ko` because the target is Korean).

### Good — text-only default with deferred experiment items

User: `/youra-tools:revision-loop --feedback=.omc/reviews/2026-04-15-YouRA_0419_revised/ --target=YouRA_0419_revised.tex`

What happens: The skill decomposes 12 issues. 9 are text-class (abstract rewrite, caption fix, citation swap, limitations addition, ...) → `issues.json`. 3 require new experiments ("add variance over 5 seeds", "compare against LLaMA-3 baseline", "report results on held-out CIFAR-100 split") → `deferred.json`. The skill echoes the split, runs the loop on the 9 in-scope items, and exits `CLEAN` with the 3 deferred items listed in `report.md`'s "Deferred — requires new experiment" section. The user can rerun with `--scope=all` or kick the deferred items to `/youra-tools:experiments-writer` separately.

### Bad — silent-success antipattern

The skill applies an edit, declares "fix looks reasonable", marks the issue `passes: true`, and moves on — all without an actual reviewer call.

Why bad: Violates the no-self-approval rule. Without an on-disk reviewer PASS verdict, the issue stays `passes: false` and the loop continues. This is the failure mode the whole skill exists to prevent.

### Bad — max-rounds masquerade

`--max-rounds=10` is hit with 3 blockers remaining. The skill writes `report.md` claiming "CLEAN, all done".

Why bad: Violates Phase 6 / Phase 7 branching. The correct behaviour is `BLOCKED` status, remaining-blockers section, and leaving the session on disk for the user to resume.

## Output

- A session bundle at `.omc/revisions/{timestamp}-{slug(target)}/` containing `state.json`, `feedback.md`, `issues.json`, `progress.txt`, `verification/round-{k}/`, and `report.md`.
- The target file on disk is updated in place (unless the user explicitly passes `--out` to redirect output — then the revised copy goes under the session directory).
- Terminal summary (≤20 lines): resolved arguments, rounds used, passed/remaining counts, final status (`CLEAN` or `BLOCKED`), path to `report.md`.

## Related skills

- `/youra-tools:paper-review` — upstream: produces the reviewer bundle this skill consumes. The revision-loop → paper-review → revision-loop cycle is the intended tight iteration for pre-submission stress-testing.
- `/youra-tools:experiments-writer` — sister skill for the Experiments section specifically, with a narrower "results are ground truth" verification checklist.
- `/youra-tools:methodology-writer` — sister skill for the Methodology section; use it when the feedback says "methodology no longer matches the code" and the underlying fix is a re-read of the implementation folder.
- `/oh-my-claudecode:ralph` — the generic PRD-driven persistence loop this skill specializes. Use ralph directly when the work is not feedback-on-a-document (e.g., code changes, infra edits).
- `/oh-my-claudecode:verify` — generic verification pattern; this skill uses a feedback-tailored variant with per-issue acceptance criteria.
- `/oh-my-claudecode:cancel` — called on `CLEAN` exit to clean up state artefacts.
