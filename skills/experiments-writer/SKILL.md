---
name: experiments-writer
description: Read experiment result .md files and a paper's Experiments section, then loop analysis → edit → verify until every number, table, and interpretation in the section is faithful to the actual results — including rewriting flawed analyses.
triggers:
  - experiments-writer
  - experiments writer
  - fix experiments
  - 실험 섹션
  - 실험 파트
  - experiments section
argument-hint: "[--results=<dir or glob>] [--paper=<file.tex>] [--section=Experiments] [--out=<patched.tex>] [--lang=en|ko] [--max-rounds=<N>] [--reviewer=architect|critic|code-reviewer]"
level: 3
---

# Experiments Writer

Use this skill when the user has a folder of experiment result documents (`.md` summaries, score tables, analysis reports, fabrication/honesty audits) and wants the Experiments section of a paper (`.tex`, `.md`) to be corrected so that every number, table, claim, and interpretation faithfully reflects what the results actually show. The loop keeps revising until a reviewer agent finds no blocking mismatches.

## Goal

Turn concrete experimental results into a faithful, complete, verifiable Experiments section:

- Every table cell and in-prose number matches the result files.
- Every interpretive paragraph (causal claims, trend explanations, implications) is actually *supported* by the results — not merely consistent with them.
- **If the paper's existing analysis of a result is wrong, rewrite it.** Do not preserve flawed reasoning just because it was already in the manuscript. This is a hard requirement: the user explicitly asked for incorrect analyses to be corrected, not only incorrect numbers.

## Arguments

Parse the invocation string. All arguments are optional; if missing, prompt the user once with concrete candidates you derived from the working directory.

| Flag | Meaning | Default resolution |
|------|---------|--------------------|
| `--results=<path>` | Folder (or glob) of experiment result `.md` files | Search in this order: `Analysis_all/`, `Analysis_summary/`, `results/`, `experiments/`, `exp/`. If exactly one matches, use it and echo the chosen path. If zero match, prompt the user interactively via `AskUserQuestion` for the folder path (open question). If multiple match, prompt via `AskUserQuestion` with the matches as options (the user may select one to treat as authoritative, or choose "merge all" in which case the skill keeps per-claim provenance). Never proceed without a confirmed results folder. |
| `--paper=<path>` | Paper file to patch | Search for newest `*revised*.tex` / `*.tex` / `*.md` in project root. If exactly one is an obvious winner, use it and echo the chosen path. If zero or multiple candidates exist, prompt via `AskUserQuestion` (list the candidates as options, or ask openly if none). Never proceed without a confirmed paper file. |
| `--section=<name>` | Section heading to focus on | `Experiments`. Match `\section{...Experiments...}` case-insensitively, including `\section*{...}`. Cover every `\subsection`/`\paragraph` under it up to the next `\section`. |
| `--out=<path>` | Output file | Edit `--paper` in place unless a different file is requested. |
| `--lang=en\|ko` | Output language | Inspect the existing section; fall back to `en`. Preserve the bilingual comment style if the source has Korean `%` comments alongside English prose. |
| `--max-rounds=<N>` | Maximum edit → verify iterations | `5`. |
| `--reviewer=...` | Verifier agent | `architect` (Opus) by default. Also allow `critic`, `code-reviewer`. |

## Workflow

1. **Resolve arguments and inputs.**
   - If `--results` was not provided, apply the default search. If exactly one default matches, use it and echo the chosen path in the next message so the user can redirect. If zero match, prompt via `AskUserQuestion` for the results folder (open question). If multiple match, prompt via `AskUserQuestion` with those paths as options. Do not silently fall back and do not proceed without a confirmed folder.
   - Confirm the resolved `--results` path exists and is non-empty; list the result files you will treat as ground truth. If a user-supplied path does not exist, stop, list the candidate subdirectories that look like results folders, and re-prompt — do not guess.
   - If `--paper` was not provided, apply the default search. If exactly one candidate is an obvious winner (newest `*revised*.tex` and no near-ties), use it and echo the chosen path. Otherwise prompt via `AskUserQuestion` with the candidates as options (or open question if none). Confirm the resolved `--paper` exists and contains the target section; if it does not contain the section, stop and report — do not silently rewrite the wrong section.
   - This skill revises the Experiments section **in place** — there is no "draft from scratch" mode. If the paper has no Experiments section at all, surface that to the user and exit; use `methodology-writer` or a plain executor for greenfield drafts.
   - Snapshot the resolved config so later iterations use the same paths.

2. **Build the "ground truth" model from results.**
   - List every `.md` under `--results` (including subfolders: e.g. `Kappa_analysis/`, `Rewrite_*analysis*/`, `Vanilla_*analysis*/`).
   - Batch the reads — prefer a single `Agent(subagent_type="Explore", thoroughness="medium", ...)` when there are more than ~10 result files; otherwise parallel `Read`.
   - For each result file, extract:
     - **Numeric tables** (row/column labels + values, units, what task/benchmark/system they belong to).
     - **Key findings / statements** (summary claims the result file itself makes).
     - **Metric definitions** (how Data Type / Honesty / Score / Kappa / etc. are defined — never silently redefine them in the paper).
     - **Scope** (which subset, how many tasks, which backbone).
   - Build a per-claim provenance index: `(value, label) → source file + section`. This is what the reviewer will check against.

3. **Extract the paper's Experiments section.**
   - Locate the section heading in `--paper`; capture everything until the next `\section` (or end of file).
   - Decompose into:
     - Setup subsection(s): benchmark, systems, evaluation protocol, rewrite-style evaluation, etc.
     - Main Results tables + surrounding prose.
     - Analysis paragraphs (causal attribution, trend commentary, implications for LLM-as-Judge, scaling, etc.).
     - Supplementary / appendix pointers that live inside the section.
   - Enumerate every claim in this section into two buckets:
     - **Quantitative**: numbers, percentages, counts, rankings, deltas, trends.
     - **Interpretive**: why-claims, because-claims, implications, attributions, comparisons.

4. **Cross-check paper vs ground truth.**
   - For each **quantitative** claim, find the matching entry in the ground-truth index. Record exact-match / near-match / missing / contradicted.
   - For each **interpretive** claim, judge whether the results actually support the stated mechanism. Flag:
     - Unsupported (no evidence in results).
     - Contradicted (results point the other way).
     - Overclaimed (stronger than evidence permits; single-judge exploratory findings stated as definitive).
     - Conflated (mixes scopes — e.g. applies a 10-task-subset finding to all 201 tasks without saying so).
   - Flag **material omissions**: findings that are present in results but missing from the section.
   - Produce a change-list with, for each item: location in paper → problem → supporting source → proposed fix.

5. **Revise.**
   - Apply the change-list to `--out` using `Edit` (small, local edits — do not rewrite untouched paragraphs).
   - When a number is wrong, fix the number and any rounded/derived mention of it elsewhere in the section.
   - When the analysis is wrong, **rewrite the interpretive paragraph** so the reasoning matches the data. Do not keep flawed prose just because it is fluent.
   - Preserve LaTeX structure: labels, `\ref{}`, `\cite{}`, `\texttt{}`, tabular alignment, `\rowcolor`, figure/table numbering.
   - Preserve the bilingual comment style if present (Korean `%` comments next to English sentences).
   - Do not silently drop existing citations or footnotes.
   - Preserve scope language ("on the ten-task subset", "under our protocol", "single-judge diagnostic") — if the source flags a finding as exploratory, the paper must also flag it.

6. **Verify (looped part).**
   - Invoke the reviewer agent with a concrete checklist pointing at `--results` and `--out`:
     - [ ] Every number in Experiments has a matching entry in `--results`.
     - [ ] Every interpretive paragraph is *supported* by the cited result, not merely consistent with it.
     - [ ] No interpretive claim is contradicted by any other file in `--results`.
     - [ ] No material result from `--results` is silently omitted.
     - [ ] Metric definitions in the paper match the definitions in `--results` (or its `definitions.md` if present).
     - [ ] Scope qualifiers (subset size, single-judge, exploratory) are preserved everywhere the claim appears.
     - [ ] Tables/figures are internally consistent with surrounding prose.
     - [ ] LaTeX integrity (labels, citations, refs, environments) is intact.
   - Reviewer agent mapping:
     - `--reviewer=architect` → `Agent(subagent_type="oh-my-claudecode:architect", model="opus", ...)`
     - `--reviewer=critic` → `Agent(subagent_type="oh-my-claudecode:critic", model="opus", ...)`
     - `--reviewer=code-reviewer` → `Agent(subagent_type="oh-my-claudecode:code-reviewer", ...)`
   - The reviewer must return a verdict of `PASS` or `FAIL`, a bulleted list of **blocking** issues, and a separate list of **non-blocking** suggestions.

7. **Loop.**
   - If verdict is `FAIL`: apply only the blocking fixes, re-run step 6. Do not touch unrelated paragraphs.
   - If verdict is `PASS`: stop and report.
   - If `--max-rounds` is hit without `PASS`: stop, report the remaining blockers, and hand control back to the user. Do not silently claim success.

8. **Report.**
   - Summarise: rounds used, reviewer verdict, number of quantitative fixes, number of interpretive rewrites, sections touched, residual non-blocking suggestions, exact output path.
   - Do NOT re-paste the whole section. Point the user to `--out`.

## Rules

- **Results are ground truth.** If prose and result file disagree, the prose must change, not the result file. If a result file itself looks wrong, surface it to the user — do not edit result files.
- **Correct wrong analyses.** The user explicitly asked that flawed interpretations be rewritten, not just wrong numbers. An interpretive paragraph that contradicts the data is a blocking issue on every round.
- **Preserve scope.** Findings derived from a 10-task subset or a single LLM judge must stay flagged as such in the paper, exactly as in the source.
- **No phantom results.** The Experiments section must not mention a table, number, or finding that is not in `--results`.
- **No silent drops.** Do not remove existing citations, figure/table references, or appendix pointers unless they are factually wrong — in which case state the removal in the final report.
- **Preserve bilingual comments.** If the source keeps Korean `%` comments alongside English prose, keep them in sync when you edit the English.
- **Run the reviewer on disk.** Verification must read the actual updated file, not an in-memory draft.
- **Apply only blocking reviewer issues between rounds.** Defer non-blocking suggestions to the final report.
- **Never mark a run complete if the reviewer returned FAIL or was never actually run.**

## Failure modes to guard against

- **Silent number drift**: table value updated in one place, left stale in surrounding prose.
- **Plausible-but-unsupported interpretation**: prose sounds right but no result file supports it.
- **Scope creep**: a finding from a 10-task subset reported as if it held across the full benchmark.
- **Metric redefinition**: paper uses a metric name with a slightly different meaning than `--results` defines it.
- **Reviewer bypass**: declaring PASS without a reviewer run. Always emit the reviewer's actual verdict.
- **Over-editing**: rewriting untouched paragraphs that the reviewer did not flag.

## Output

- Updated file at `--out` (or in-place edits to `--paper`).
- Short report block containing:
  - Resolved `--results` / `--paper` / `--section` / `--out` / `--lang` / `--reviewer`.
  - Iteration count and final verdict.
  - Counts: quantitative fixes, interpretive rewrites, omissions added, overclaims softened.
  - Outstanding non-blocking suggestions (if any).

## Related skills

- `/oh-my-claudecode:methodology-writer` — sister skill for the Methodology section.
- `/oh-my-claudecode:verify` — generic verification pattern (this skill uses a narrower reviewer checklist tailored to experiments).
- `/oh-my-claudecode:ralph` — wrap the loop in PRD-driven persistence if the manuscript is still churning.
- `/oh-my-claudecode:paper-review` — run a full peer-review simulation once this skill has passed.
