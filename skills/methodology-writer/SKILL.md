---
name: methodology-writer
description: Read an implementation folder, analyze the code, and write or update a Methodology section — then loop edit → verify until the reviewer signs off
triggers:
  - methodology
  - 방법론
  - write methodology
  - methodology-writer
argument-hint: "[--impl=<path>] [--out=<methodology.md>] [--lang=en|ko] [--max-rounds=<N>] [--reviewer=architect|critic|code-reviewer]"
level: 3
---

# Methodology Writer

Use this skill when the user has an implementation folder (e.g. `방법론 (구현한)/`, `src/`, a submodule) and wants a Methodology section of a paper or report to be written or updated so that it faithfully describes what the code actually does — with a verification loop that keeps revising until a reviewer agent finds no blocking issues.

## Goal
Turn a concrete implementation into a faithful, complete, verifiable Methodology document. No hand-waving, no invented components, no missed phases.

## Arguments

Parse the invocation string. All arguments are optional; if missing, prompt the user once with concrete candidates.

| Flag | Meaning | Default resolution |
|------|---------|--------------------|
| `--impl=<path>` | Implementation folder to analyze | Search in this order: `방법론 (구현한)/`, `구현한 방법론/`, `implementation/`, `src/`. If exactly one matches, use it and echo the chosen path. If zero match, or multiple match, prompt the user interactively via `AskUserQuestion` with the candidate paths (or an open "please give the folder path" question when zero match). Never proceed without a confirmed implementation folder. |
| `--out=<path>` | Methodology document to write/update | Search for the newest `*methodology*.md` / `*methodology*.tex` in the project root. If none, propose `methodology.md`. |
| `--lang=en\|ko` | Output language | Inspect existing methodology target; fall back to `en`. |
| `--max-rounds=<N>` | Maximum edit→verify iterations | `5`. |
| `--reviewer=...` | Verifier agent | `architect` (Opus) for academic rigour; allow `critic`, `code-reviewer`. |

## Workflow

1. **Resolve arguments and inputs.**
   - If `--impl` was not provided, apply the default search. If exactly one default path matches, use it and echo the chosen path in the next message so the user can redirect. If zero paths match, prompt the user via `AskUserQuestion` for the implementation folder (open question). If multiple default paths match, prompt the user via `AskUserQuestion` with those paths as options. Do not silently fall back and do not proceed without a confirmed folder.
   - Confirm the resolved `--impl` path exists and is non-empty. If it was user-supplied and does not exist, stop, list workspace subdirectories that look like implementation folders, and re-prompt — do not guess.
   - Confirm `--out` path. If the file exists, read it; treat it as the baseline to revise (this skill verifies and revises existing methodology text in place). If it does not exist, plan a fresh draft with a standard Methodology skeleton — the verify-revise loop then runs on the new draft identically.
   - Snapshot the resolved config so later iterations use the same paths.

2. **Inventory the implementation.**
   - List files under `--impl` (non-recursive first, then one level deep). Note language, module boundaries, entry points, configs, tests.
   - Batch the reads — prefer a single `Agent(subagent_type="Explore", thoroughness="medium", ...)` when the tree has more than ~15 files; otherwise read the key files directly with parallel `Read` calls.
   - Extract for each module: purpose, inputs/outputs, key algorithms, external dependencies (MCP tools, APIs, datasets), invariants, and phase ordering if there is a pipeline.

3. **Plan the Methodology structure.**
   - If the baseline doc exists, keep its section numbering and heading style; only add/rewrite sections that the code has changed or that were missing.
   - If starting fresh, use an appropriate academic skeleton: Overview → System Architecture → Components → Workflow/Phases → Failure Handling → Implementation Details → Limitations.
   - For each section, list the evidence (file paths + symbols) that will back the claims.

4. **Draft or revise.**
   - Write or edit `--out` in the target language. Every non-trivial claim must trace to a file/symbol in `--impl`.
   - Preserve existing tables, figure references, citations — do not silently renumber or drop them.
   - Keep prose academic: no marketing tone, no speculative claims not grounded in the code.

5. **Verify (this is the looped part).**
   - Invoke the reviewer agent with a concrete checklist, pointing it at both `--impl` and `--out`:
     - Every component named in the Methodology exists in `--impl` (no fabricated modules).
     - Every non-trivial algorithm/phase in `--impl` is either described, or explicitly marked out-of-scope.
     - No section claims capabilities the code does not implement.
     - Terminology is consistent between code identifiers and prose.
     - Language/tense is consistent; headings match the existing style.
     - Figures/tables/citations are intact and still correct.
   - `--reviewer=architect` → `Agent(subagent_type="oh-my-claudecode:architect", model="opus", ...)`.
   - `--reviewer=critic` → `Agent(subagent_type="oh-my-claudecode:critic", model="opus", ...)`.
   - `--reviewer=code-reviewer` → `Agent(subagent_type="oh-my-claudecode:code-reviewer", ...)`.
   - The reviewer must return a verdict of `PASS` or `FAIL` plus a bulleted list of blocking issues and, separately, non-blocking suggestions.

6. **Loop.**
   - If verdict is `FAIL`: apply only the blocking fixes, re-run step 5. Do not touch unrelated sections.
   - If verdict is `PASS`: stop and report.
   - If `--max-rounds` is hit without `PASS`: stop, report the remaining blockers, and hand control back to the user. Do not silently claim success.

7. **Report.**
   - Summarise: rounds used, sections changed, reviewer verdict, remaining non-blocking suggestions, exact output path.
   - Do NOT re-paste the whole Methodology. Point the user to `--out`.

## Rules

- Ground every claim in the implementation. If the code does not do X, the Methodology must not say it does.
- Do not invent phase names, agents, or tools that are not in the implementation folder.
- Respect the existing document's style and numbering unless the user asks for a rewrite.
- Run reviewer verification on the actual updated file on disk (not on an in-memory draft).
- Apply only blocking reviewer issues between rounds — defer "nice-to-have" suggestions to the final report.
- When `--lang=ko`, keep code identifiers, table headers, and citations in their original form.
- Never mark a run complete if the reviewer returned `FAIL` or was never actually run.

## Failure modes to guard against

- **Phantom components**: Methodology mentions a module that no longer exists in `--impl`.
- **Silent drift**: Implementation changed (new phase, renamed agent) but prose still reflects the old version.
- **Over-summarisation**: A complex algorithm collapsed into one vague sentence.
- **Reviewer bypass**: Declaring PASS without a reviewer run. Always emit the reviewer's actual verdict in the final report.

## Output

- Updated/created file at `--out`.
- Short report block containing:
  - Resolved `--impl` / `--out` / `--lang` / `--reviewer`.
  - Iteration count and final verdict.
  - Sections added/modified.
  - Outstanding non-blocking suggestions (if any).

## Related skills

- `/oh-my-claudecode:verify` — generic verification pattern (this skill uses a narrower reviewer checklist).
- `/oh-my-claudecode:ralph` — if the user wants the verification loop wrapped in PRD-driven persistence.
- `/oh-my-claudecode:writer-memory` — for capturing durable writing preferences once this skill has been used a few times.
