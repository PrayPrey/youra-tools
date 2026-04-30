---
name: paper-review
description: Strict peer-review simulation with ACL/NeurIPS/EMNLP reviewer personas — objective scoring, Semantic Scholar + Exa/web fact verification, and a rebuttal lane that challenges author responses with evidence.
triggers:
  - paper review
  - paper-review
  - peer review
  - ACL review
  - NeurIPS review
  - EMNLP review
  - 논문 리뷰
  - 피어 리뷰
  - 리뷰어 시뮬레이션
  - rebuttal
  - 리뷰탈
argument-hint: "[--paper=<path>] [--venue=ACL|NeurIPS|EMNLP|mixed] [--num-reviewers=N] [--severity=strict|standard|lenient] [--mode=review|rebuttal|full] [--rebuttal=<path>] [--verify=true|false] [--lang=en|ko] [--out=<path>]"
level: 4
---

# Paper Review

Use this skill when the user wants a **strict, objective peer-review simulation** of an academic paper, styled after reviewers at top NLP/ML venues (ACL, NeurIPS, EMNLP, ICML, ICLR). The skill scores the paper using each venue's official rubric, cross-checks factual claims against Semantic Scholar and the open web (Exa MCP if available, WebSearch/WebFetch otherwise), and then runs a **rebuttal lane** that either simulates an author rebuttal or evaluates the rebuttal the user provides.

The goal is brutal honesty with evidence — not praise. A reviewer agent must cite lines, tables, equations, or prior work whenever they assert a weakness.

## Default invocation

If the user calls the skill with no flags, resolve to:

```
--venue=mixed --severity=strict --mode=full --num-reviewers=3 --verify=true --rebuttal-max-revisions=2
```

Still prompt once for `--paper=<path>` when the paper cannot be auto-detected; every other flag falls back to the default without asking.

## Goal

Produce a reproducible review bundle that:

1. **Scores** the paper with each venue's actual scales (e.g., ACL ARR `Soundness/Excitement/Overall`, NeurIPS `Quality/Clarity/Originality/Significance`).
2. **Verifies** every verifiable claim — cited prior work, reported SOTA numbers, dataset sizes, model sizes — against external sources.
3. **Attacks weaknesses** with concrete evidence, the way a real reviewer would.
4. **Rebuts** (or re-rebuts) with a meta-reviewer pass so authors can see which responses actually close the gap.

## When to invoke

- The user has a paper draft (PDF, `.tex`, `.md`) and wants an ACL/NeurIPS/EMNLP-style peer review.
- The user wants to stress-test their paper before submission.
- The user received real reviews and wants help writing the rebuttal (use `--mode=rebuttal`).
- The user wants a meta-review that compares their actual rebuttal against remaining concerns.

## Arguments

Parse the invocation string. All arguments are optional — ask the user once with concrete candidates when missing.

| Flag | Meaning | Default |
|------|---------|---------|
| `--paper=<path>` | Paper file to review (PDF / TeX / MD). | Newest `*.pdf` or `*.tex` in CWD; else prompt. |
| `--venue=<v>` | Target venue rubric: `ACL`, `NeurIPS`, `EMNLP`, `ICML`, `ICLR`, or `mixed`. | `mixed` (spawns one reviewer per big-three venue). |
| `--num-reviewers=N` | Number of reviewer personas (1-5). | 3 |
| `--severity=<s>` | `lenient`, `standard`, `strict` — controls score priors and attack surface. | `strict` |
| `--mode=<m>` | `review`, `rebuttal`, or `full`. `full` = review → auto-rebuttal by simulated authors → meta-review. | `full` |
| `--rebuttal=<path>` | User-provided rebuttal text/file. When set, `--mode` defaults to `rebuttal`. | (empty) |
| `--verify=true|false` | Enable Semantic Scholar + web claim verification. | `true` |
| `--rebuttal-max-revisions=N` | Max Phase 4↔4.5 revision iterations after the first rebuttal pass. `0` disables the loop. | `2` |
| `--rebuttal-revise-on=<verdicts>` | Comma list of audit verdicts that trigger a revision. | `UNSUPPORTED,MISREPRESENTED,NEW_CLAIM_OUT_OF_BLOCK` |
| `--lang=en|ko` | Output language for reviews and meta-review. | Inspect paper; fallback `en`. |
| `--out=<path>` | Output directory for the review bundle. | `.omc/reviews/{timestamp}-{paper-stem}/` |

## Session artefacts

```
.omc/reviews/{timestamp}-{paper-stem}/
  state.json                     # session state
  paper/
    paper.txt                    # extracted plain text
    claims.json                  # extracted verifiable claims
    figures/                     # captured figures / tables if any
  verification/
    claim-{id}.json              # per-claim verification verdict
    summary.md                   # verification overview
  reviews/
    reviewer-ACL.md              # one per persona
    reviewer-NeurIPS.md
    reviewer-EMNLP.md
  rebuttal/
    authors-response.md          # final / converged rebuttal (symlink or copy of latest iter)
    reviewer-ACL-final.md        # post-rebuttal verdict per reviewer (final iter)
    ...
    verification/                # Phase 4.5 — rebuttal vs paper audit (final iter)
      rebuttal-claims.json       # per-claim verdict (SUPPORTED/UNSUPPORTED/MISREPRESENTED/NEW_CLAIM)
      audit.md                   # human-readable record incl. mismatch_reason citing paper evidence
      summary.md                 # counts + unresolved mismatches with paper line refs
    iterations/                  # Phase 4.6 — rebuttal revision loop history
      iter-0/                    # initial Phase 4 + Phase 4.5
        authors-response.md
        reviewer-{venue}-final.md
        verification/{rebuttal-claims.json,audit.md,summary.md}
      iter-1/ ...                # one folder per revision attempt
      convergence.md             # per-iter counts, what changed, stop reason
  meta-review.md                 # senior area chair synthesis
  scores.json                    # machine-readable scores
  report.md                      # top-level human-readable bundle
```

## Workflow

### Phase 0 — Resolve arguments

1. Confirm `--paper` exists. For PDF, prepare to extract via Read (PDF pages parameter) or shell tools.
2. Create session folder `.omc/reviews/{timestamp}-{paper-stem}/`.
3. Write `state.json` with resolved arguments and reviewer roster.

### Phase 1 — Ingest the paper

1. Read the paper. For PDFs > 20 pages, chunk by `pages=` ranges; for TeX/MD read in full.
2. Extract structured elements and persist to `paper/paper.txt`:
   - Title, authors, affiliations, venue-target statement.
   - Abstract, contributions list, problem statement.
   - Section outline with page/line ranges.
   - All tables and figures (captions + captured numbers).
   - Equations with surrounding text.
   - Bibliography entries (authors, year, title, venue).
3. Build `paper/claims.json`: a list of **verifiable factual claims**. Each claim entry:
   ```json
   {
     "id": "C1",
     "section": "Introduction",
     "line": 42,
     "claim": "BERT-base achieves 84.5 F1 on SQuAD v1.1 dev",
     "type": "sota_number | prior_work_attribution | dataset_stat | reproduced_baseline | methodological",
     "requires_verification": true,
     "source_cited": "Devlin et al., 2019"
   }
   ```
   Prioritize: (a) SOTA/benchmark numbers, (b) attributions to prior work, (c) dataset stats, (d) architectural claims ("first to...", "unlike prior work..."), (e) reproduced baselines.

### Phase 2 — Claim verification (parallel)

Skip if `--verify=false`.

Fan out verification **in parallel** across claims. Batch claims into groups of ≤20 per MCP call where the API supports batching.

**Tool priority** for each claim:

1. **Semantic Scholar MCP** (always first choice for academic claims):
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__search-paper-title` — resolve cited title → paperId.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__get-paper-abstract` — fetch abstract for cross-reference.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-search-basic` / `paper-search-advanced` — find the original paper when only author+year is cited.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__search-arxiv` — find arXiv preprints (often has the raw numbers).
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__download-full-paper-arxiv` — when the number is in a table deep in the PDF.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-batch` — for 5+ paperIds at once.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-citations` / `papers-references` — for "no one has done X" claims (check citations against the field).
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__authors-papers` — when the claim attributes an idea to a specific author's body of work.
2. **Exa MCP** if present in the environment (tools appear under `mcp__exa__*` — see the deferred-tool list). Use for web-grounded claims that Semantic Scholar cannot confirm (blog posts, model cards, dataset websites).
3. **WebSearch / WebFetch** as graceful fallback when neither Semantic Scholar nor Exa has the claim — mark the verdict confidence one tier lower than MCP-verified.

**Verification verdict** (write to `verification/claim-{id}.json`):

```json
{
  "id": "C1",
  "verdict": "VERIFIED | CONTRADICTED | UNVERIFIED | PARTIAL",
  "confidence": "HIGH | MEDIUM | LOW",
  "evidence": [
    {"source": "Devlin et al. 2019 Table 2", "url": "...", "quote": "BERT_BASE: 84.8 F1 on SQuAD dev", "discrepancy": "Paper reports 84.5, source says 84.8 — within rounding but should be footnoted"}
  ],
  "reviewer_note": "Short summary a reviewer can paste."
}
```

- `CONTRADICTED` and `PARTIAL` verdicts are surfaced to every reviewer as attack ammunition.
- Never mark `VERIFIED` without quoting the source.
- If a number differs within 1 rounding unit, mark `PARTIAL` with a note on decimal precision.

**Parallel agent pattern** for verification:

```
Agent(
  subagent_type="oh-my-claudecode:scientist",
  model="sonnet",
  description="Verify claim batch {N}",
  prompt="""
    [CLAIM_VERIFICATION_BATCH]
    For each claim below, call Semantic Scholar MCP first, then Exa/Web as fallback.
    Emit one JSON verdict per claim with fields: id, verdict, confidence, evidence, reviewer_note.
    Do NOT fabricate sources. If no source is found, emit UNVERIFIED.

    Claims:
    { batched claim entries }
  """
)
```

Maximum 5 parallel verification scientists to respect rate limits.

### Phase 3 — Reviewer personas (parallel)

Spawn one agent per reviewer slot. Use `oh-my-claudecode:critic` (Opus) for `--severity=strict`, `oh-my-claudecode:code-reviewer` (Sonnet) for `standard`, `oh-my-claudecode:scientist` for `lenient`.

Pass each reviewer:
- The full paper text.
- The verification bundle (all `CONTRADICTED / PARTIAL` verdicts **highlighted**).
- Their venue-specific rubric (see **Rubrics** below).
- The severity floor (strict reviewers never give the top score without extraordinary evidence).
- A persona prompt that reflects the venue's reviewing culture.

**Invocation pattern** (fire all reviewers in one assistant turn for true parallelism):

```
Agent(subagent_type="oh-my-claudecode:critic", model="opus",
      description="ACL reviewer",
      prompt=<ACL persona + rubric + paper + verification bundle>)
Agent(subagent_type="oh-my-claudecode:critic", model="opus",
      description="NeurIPS reviewer",
      prompt=<NeurIPS persona + rubric + paper + verification bundle>)
Agent(subagent_type="oh-my-claudecode:critic", model="opus",
      description="EMNLP reviewer",
      prompt=<EMNLP persona + rubric + paper + verification bundle>)
```

Each reviewer writes `reviews/reviewer-{venue}.md` following the **Review template** below. They MUST:
- Quote line numbers, table numbers, or equation numbers for every weakness.
- Reference the verification bundle by claim ID when they attack a reproduced number or a prior-work attribution.
- Give every rubric score and justify it in at least 2 sentences.
- List **specific questions the authors must answer in the rebuttal**.

### Phase 4 — Rebuttal lane

Branch on `--mode`:

#### Mode `full` (default)

1. **Simulate authors** via a single agent instance using `oh-my-claudecode:executor` (model=opus). Give it: the paper + all three reviewer reports. Ask it to write the strongest possible rebuttal the real authors could write with what is already in the paper (no new experiments). Save to `rebuttal/authors-response.md`.
2. **Re-evaluate** per reviewer: spawn the same reviewer agents again with the rebuttal attached. Each emits a post-rebuttal score and a list of concerns that remain. Save to `rebuttal/reviewer-{venue}-final.md`.
3. **Meta-review** via `oh-my-claudecode:architect` (opus): synthesise final recommendation (`Accept / Weak Accept / Borderline / Weak Reject / Reject`) and write `meta-review.md`.

#### Mode `rebuttal`

1. Read `--rebuttal=<path>` (or ask user to paste).
2. For each reviewer, reload their original review and attach the user's rebuttal. Reviewer re-scores and writes `reviewer-{venue}-final.md`. They MUST explicitly say which concerns are resolved, which are partially addressed, and which remain unaddressed. A resolved concern should raise the sub-score only if the rebuttal introduced new evidence; polite restatement does not move the score.
3. Meta-review as above.
4. Optionally emit a **"rebuttal counter-critique"** block: what the authors should still add to the camera-ready if accepted.

#### Mode `review`

Stop after Phase 3. Produce `report.md` without rebuttal.

### Phase 4.5 — Rebuttal verification (isolated subagent)

Skip when `--mode=review` (no rebuttal exists). Otherwise this phase runs *after* Phase 4 and *before* Phase 5, regardless of whether the rebuttal was simulated (`full`) or user-provided (`rebuttal`).

**Purpose.** A rebuttal can claim "as shown in Table 3", "we report this in Section 4.2", "our experiment uses N=10000 samples", or "we already cite X et al. 2022". The reviewer's score recovery in Phase 4 is only legitimate if those rebuttal assertions are actually true with respect to the paper. This phase audits each rebuttal claim against the paper and the Phase 2 verification bundle, then records *why* anything that does not match is wrong.

**Isolation.** This pass MUST run inside a subagent so the main context is not polluted by the per-claim back-and-forth. The main agent only reads `rebuttal/verification/summary.md` back into context — never the full JSON.

**Subagent invocation** (single call):

```
Agent(
  subagent_type="oh-my-claudecode:scientist",
  model="opus",
  description="Verify rebuttal against paper",
  prompt="""
    [REBUTTAL_VERIFICATION] — paper-internal audit, no external tool calls.
    Inputs (read from disk only):
      - paper/paper.txt
      - rebuttal/authors-response.md      (or user-provided rebuttal copy)
      - verification/summary.md           (Phase 2 verdicts)
      - reviews/reviewer-*.md             (so you know which weaknesses each
                                           rebuttal claim is trying to close)

    For every *factual* assertion the rebuttal makes — numbers, table/figure
    references, section references, "we already showed / already cite"
    statements, prior-work attributions, dataset stats — emit one JSON object:

      {
        "rebuttal_claim_id":   "R{n}",
        "addresses_weakness":  "W{k} from reviewer-{venue}",
        "quote_from_rebuttal": "<exact substring>",
        "claimed_paper_anchor":"<table/figure/section/line as cited in rebuttal>",
        "verdict":             "SUPPORTED | UNSUPPORTED | MISREPRESENTED | NEW_CLAIM",
        "paper_evidence":      "<quoted paper passage with line/table number, or 'not found'>",
        "mismatch_reason":     "<only when verdict != SUPPORTED — explain *why*
                                 the rebuttal diverges from the paper, citing
                                 specific paper evidence (line/table/equation)>",
        "severity":            "blocker | major | minor"
      }

    Verdict definitions:
      - SUPPORTED:      paper actually contains what the rebuttal claims, at
                        the cited anchor, with the cited number/scope.
      - UNSUPPORTED:    paper does not contain the cited result, and the
                        rebuttal does NOT promise it as new material.
      - MISREPRESENTED: paper contains a related result, but the rebuttal
                        mischaracterises the number, scope, or conclusion
                        (e.g., paper reports 84.5 but rebuttal says 88.2;
                        paper reports on English only but rebuttal claims
                        multilingual coverage).
      - NEW_CLAIM:      rebuttal introduces material not present in the paper.
                        Allowed ONLY inside an explicit "New material promised
                        for camera-ready" block — otherwise treat as a
                        rebuttal-rule violation and mark severity=blocker.

    Cross-check rules:
      - If the rebuttal repeats a prior-work claim that Phase 2 marked
        CONTRADICTED, set verdict=MISREPRESENTED and reuse the Phase 2
        evidence in mismatch_reason.
      - Do NOT call WebSearch, WebFetch, Semantic Scholar, or Exa. This pass
        is paper-internal. Prior-work claims are checked against
        verification/summary.md only.
      - When in doubt between MISREPRESENTED and UNSUPPORTED, prefer the more
        specific verdict (MISREPRESENTED) and quote the related paper passage.

    Persist to disk yourself:
      - rebuttal/verification/rebuttal-claims.json  (array of all claim objects)
      - rebuttal/verification/audit.md              (human-readable per-claim
                                                     entries with quote, paper
                                                     evidence, and mismatch_reason)
      - rebuttal/verification/summary.md            (counts by verdict, list of
                                                     blockers/majors with paper
                                                     line refs, and a roll-back
                                                     list naming each
                                                     reviewer-{venue}-final.md
                                                     score that depends on a
                                                     non-SUPPORTED claim)

    Return only a 10–20 line markdown summary to the caller. Do not echo the
    full JSON back.
  """
)
```

**Effect on scoring and meta-review**:

- Any `UNSUPPORTED` or `MISREPRESENTED` rebuttal claim **invalidates** the corresponding score recovery in `rebuttal/reviewer-{venue}-final.md`. The meta-review MUST roll those scores back toward the pre-rebuttal value and cite the audit entry by `rebuttal_claim_id`.
- `NEW_CLAIM` outside an explicit "New material promised for camera-ready" block is a rebuttal-rule violation and is surfaced as a blocker in the meta-review.
- A clean audit (all SUPPORTED) is recorded in the meta-review as a positive signal but does not, by itself, raise scores beyond what Phase 4 produced.
- `audit.md` is the durable record of *why* a rebuttal claim was rejected — it must always quote the paper passage that contradicts the claim, not just say "not found".

### Phase 4.6 — Rebuttal revision loop

Skip when `--mode=review`, when `--rebuttal-max-revisions=0`, or when Phase 4.5's audit has zero entries matching `--rebuttal-revise-on`. Otherwise loop the rebuttal lane using the audit findings as feedback.

**Mode behaviour**:
- `--mode=full` — the simulated rebuttal is rewritten by the executor and re-audited up to `--rebuttal-max-revisions` times.
- `--mode=rebuttal` (user-provided) — the user's rebuttal text is *not* rewritten on their behalf. Run the loop in **report-only** form: produce a `required-fixes.md` per iteration listing what the user must change, and stop after iteration 0. The convergence record still gets written so the user can apply fixes and re-invoke the skill.

**Snapshot before iterating.** Move the current `rebuttal/authors-response.md`, `reviewer-{venue}-final.md`, and `verification/` outputs into `rebuttal/iterations/iter-0/` so the original artefacts are preserved.

**Loop invariants** — for `i = 1 .. --rebuttal-max-revisions`:

1. **Build feedback packet.** Read `rebuttal/iterations/iter-{i-1}/verification/audit.md` and pull every entry whose `verdict` is in `--rebuttal-revise-on`. For each, capture: `rebuttal_claim_id`, `quote_from_rebuttal`, `claimed_paper_anchor`, `paper_evidence`, `mismatch_reason`, `severity`. This packet is the only Phase 4.5 output that crosses into the next rebuttal — do NOT replay the full audit JSON.
2. **Regenerate rebuttal** via `oh-my-claudecode:executor` (`model=opus`) with prompt:
   ```
   [REBUTTAL_REVISION iter={i}]
   You wrote rebuttal/iterations/iter-{i-1}/authors-response.md.
   Audit found these problems — fix every one:
     {feedback packet}
   Hard rules:
     - Do NOT re-cite a table/figure/section that the audit marked
       UNSUPPORTED. Either remove the citation or replace it with a
       SUPPORTED anchor from the paper.
     - For MISREPRESENTED entries, restate the claim using the exact
       number/scope quoted in paper_evidence, or concede the weakness.
     - NEW_CLAIM entries must move under an explicit
       "## New material promised for camera-ready" block, or be deleted.
     - Do NOT introduce new factual claims that are not already in the
       paper or in the Phase 2 verification bundle.
   Write the revised rebuttal to rebuttal/iterations/iter-{i}/authors-response.md.
   ```
3. **Re-run reviewer post-rebuttal pass** for the revised rebuttal — same parallel reviewer agents as Phase 4 — and write `rebuttal/iterations/iter-{i}/reviewer-{venue}-final.md`.
4. **Re-audit** by re-invoking Phase 4.5's subagent on `iter-{i}/authors-response.md`. Persist outputs under `iter-{i}/verification/`.
5. **Stop conditions** — break the loop and record the reason in `rebuttal/iterations/convergence.md`:
   - `CONVERGED` — zero entries in iter-{i} match `--rebuttal-revise-on`.
   - `EXHAUSTED` — `i == --rebuttal-max-revisions`.
   - `NO_PROGRESS` — iter-{i} produced ≥ the same number of revise-on entries as iter-{i-1} *and* the set of `rebuttal_claim_id` values overlaps by ≥ 80%. Do not waste another iteration on a stuck rebuttal.
   - `BLOCKED` — executor refused to revise (e.g., every remaining claim would require new experiments). Record the executor's reason verbatim.

**Promote the winning iteration.** After the loop, copy the best iteration's artefacts up to `rebuttal/` (overwriting the non-iter-prefixed files):
- Best = lowest count of revise-on verdicts, ties broken by lowest `severity=blocker` count, then by highest iteration index.

**Write `rebuttal/iterations/convergence.md`**:

```markdown
# Rebuttal revision loop

| Iter | UNSUPPORTED | MISREPRESENTED | NEW_CLAIM_OOB | SUPPORTED | Notes |
|------|-------------|----------------|---------------|-----------|-------|
| 0    | ...         | ...            | ...           | ...       | initial |
| 1    | ...         | ...            | ...           | ...       | fixed Rk, Rm |
| ...  |             |                |               |           |       |

Stop reason: CONVERGED | EXHAUSTED | NO_PROGRESS | BLOCKED
Promoted iteration: iter-{j}
Carry-over blockers (still present in promoted iter): R{...}
```

**Effect on meta-review and report**:
- The meta-review reads only `convergence.md` + the promoted iteration's `verification/summary.md`. It must explicitly state the stop reason and any carry-over blockers.
- If stop reason is `EXHAUSTED` or `NO_PROGRESS`, the meta-review may not raise the recommendation above the pre-rebuttal level for any reviewer whose post-rebuttal score depended on a still-failing claim.
- `report.md` shows the convergence table and links to each `iter-{i}/`.

**Cost guard.** The loop is bounded by `--rebuttal-max-revisions`. Do not introduce auto-extension. If the user wants more iterations, they re-invoke the skill with a higher value.

### Phase 5 — Aggregate and report

1. Write `scores.json` with machine-readable scores per reviewer per rubric dimension, before and after rebuttal. When Phase 4.5 rolled back a score, record both the unrolled and rolled-back values plus the offending `rebuttal_claim_id`.
2. Write `report.md`:
   - Executive summary (2 paragraphs: one on contribution, one on remaining blockers).
   - Per-reviewer section: rubric table, key weaknesses, remaining concerns after rebuttal.
   - Verification summary: how many claims VERIFIED / CONTRADICTED / UNVERIFIED.
   - **Rebuttal verification summary** (when Phase 4.5 ran): counts by verdict and the list of UNSUPPORTED / MISREPRESENTED / out-of-block NEW_CLAIM entries with their paper line refs and mismatch_reason — pulled verbatim from `rebuttal/verification/summary.md`.
   - **Rebuttal revision loop** (when Phase 4.6 ran): the convergence table, stop reason, promoted iteration, and any carry-over blockers — pulled from `rebuttal/iterations/convergence.md`.
   - Meta-review verdict with the venue-appropriate decision label.
   - Actionable revision list ranked by severity.
3. Print a short summary to the user (≤30 lines):
   - Final recommendation + average score.
   - Top 3 weaknesses.
   - Path to `report.md` and session directory.

## Rubrics

### ACL / ARR (Action Editor–style)

| Dimension | Scale | Anchor guidance |
|-----------|-------|-----------------|
| Soundness | 1-5 | 5=bulletproof methodology; 3=reasonable with gaps; 1=fundamentally flawed |
| Excitement | 1-5 | 5=advances the subfield; 3=solid increment; 1=incremental or niche |
| Reproducibility | 1-5 | 5=code+data+hyperparams released; 3=partial; 1=unreplicable |
| Overall | 1-5 | Not an average — reviewer's holistic recommendation |
| Confidence | 1-5 | 5=expert in exact subarea; 3=familiar; 1=quick read |
| Ethical issues | Y/N | Must flag dual-use, annotator welfare, PII |

**Strict calibration:** Overall=5 requires soundness≥4 AND excitement≥4 AND no CONTRADICTED claims. Overall=4 requires soundness≥4 OR excitement≥4.

### NeurIPS

| Dimension | Scale | Anchor guidance |
|-----------|-------|-----------------|
| Quality | 1-10 | Technical correctness, experimental rigour |
| Clarity | 1-10 | Writing, figures, notation |
| Originality | 1-10 | Novelty relative to cited and uncited prior work |
| Significance | 1-10 | Impact on field; transferability |
| Soundness | 1-4 | 4=excellent; 3=good; 2=fair; 1=poor |
| Presentation | 1-4 | same scale |
| Contribution | 1-4 | same scale |
| Overall | 1-10 | 10=seminal; 7=clear accept; 5=borderline; 3=reject; 1=trivially wrong |
| Confidence | 1-5 | 5=expert; 3=familiar; 1=outsider |

**Strict calibration:** Overall≥8 requires Quality≥8 AND Originality≥7 AND zero CONTRADICTED claims AND full reproducibility statement.

### EMNLP

Same rubric as ACL/ARR plus:
- **Reproducibility checklist** compliance (hyperparams, data splits, seeds, environment).
- **Resource claims**: compute, carbon estimate, licensing of datasets.

### ICML / ICLR

Quality / Clarity / Originality / Significance on 1-10 scales. ICLR additionally requires reviewers to flag double-blind violations.

## Review template

Each reviewer writes in this format (language follows `--lang`):

```markdown
# Reviewer: {venue} ({persona handle})

## Summary of paper
<2–4 sentences, in the reviewer's own words. No praise inflation.>

## Strengths
- S1: <strength with evidence — cite line/table>
- S2: ...

## Weaknesses
- W1 [severity: major|minor]: <claim>
  - Evidence: <line / table / equation / verification claim id>
  - Why it matters: <impact on the paper's thesis>
- W2 ...

## Questions for authors (rebuttal)
Q1. <specific question that can be answered with existing experiments or clarification>
Q2. ...

## Scores
| Dimension | Score | Justification |
|-----------|-------|---------------|
| ... | ... | ... |

## Recommendation
<Accept / Weak Accept / Borderline / Weak Reject / Reject>

## Confidence
<scale + one-line rationale>

## Ethics flags
<Y/N + note>
```

## Rebuttal template (simulated authors)

```markdown
# Authors' Response

## Response to Reviewer {venue}

### On W1 (<short label>)
<direct response. Cite existing table/equation. Concede if the reviewer is right.>

### On W2 ...

## New material promised for camera-ready
- ...

## Respectful disagreement
- <Only when the reviewer misread something. Quote the passage that contradicts the reviewer's claim.>
```

## Persona prompts (hint, not verbatim)

- **ACL persona**: Senior ACL reviewer. Cares about linguistic justification, error analysis, and clear ablations. Penalises papers that chase benchmark numbers without analysis. Allergic to overclaiming.
- **NeurIPS persona**: ML theorist + empiricist. Cares about algorithmic novelty, theoretical grounding, compute-matched baselines, and significance testing. Expects error bars.
- **EMNLP persona**: Applied NLP reviewer. Cares about data quality, annotator agreement, multilingual coverage, and reproducibility. Hostile to English-only evaluations claiming generality.
- **ICML persona**: Optimisation/learning theorist. Requires formal assumptions, tight proofs, and honest comparisons.
- **ICLR persona**: Representation-learning reviewer. Focuses on scaling behaviour, open-source readiness, and thorough baseline coverage.

Each reviewer MUST also:
- Not repeat the abstract back as "strengths".
- Not accept claims at face value when the verification bundle contradicts them.
- Never give a top Overall score with any CONTRADICTED claim unresolved.

## Meta-review template

```markdown
# Meta-Review

## Synthesis
<3–5 sentences synthesising reviewer consensus and disagreements.>

## Average scores
- Pre-rebuttal: overall X.Y (min A, max B)
- Post-rebuttal: overall X.Y (min A, max B)

## Verification summary
- Claims total: N
- Verified: ... / Contradicted: ... / Unverified: ...
- Unresolved contradictions: list

## Unresolved concerns after rebuttal
- ...

## Recommendation
<Accept / Weak Accept / Borderline / Weak Reject / Reject> with 1-sentence rationale.

## Required revisions for camera-ready
1. ...
2. ...
```

## Rules

- **Evidence or silence.** A reviewer comment without a line/table/equation/claim-id reference must be removed before the report is written.
- **No fabricated citations.** Every external reference in a review must come from the Semantic Scholar / Exa / WebSearch verification bundle.
- **Strict severity = strict.** Never soften scores to be polite. A 3/5 soundness is a 3/5 soundness.
- **Rebuttal must actually change something.** If the authors' response contains no new evidence, the reviewer score does not move up.
- **Rebuttals are verified, not trusted.** Every factual assertion in `rebuttal/authors-response.md` (or a user-provided rebuttal) is audited against the paper in Phase 4.5 inside an isolated subagent. UNSUPPORTED / MISREPRESENTED claims roll back any score recovery they were used to justify, and `rebuttal/verification/audit.md` must quote the paper passage that contradicts each rejected claim along with a `mismatch_reason` — silent disagreement is not allowed.
- **Rebuttal revision loop is bounded.** Phase 4.6 runs at most `--rebuttal-max-revisions` iterations. Stop early on `CONVERGED`, `NO_PROGRESS`, or `BLOCKED`. Never auto-extend the loop, never silently retry — the convergence record must always state the stop reason. For `--mode=rebuttal`, do not rewrite the user's rebuttal; emit `required-fixes.md` and stop after iter 0.
- **Language consistency.** If `--lang=ko`, reviews are in Korean but keep equation labels, table numbers, citations, and English-origin technical terms in their original form.
- **No silent success.** If verification could not run (MCP down, no network), say so in `verification/summary.md` and downgrade every affected reviewer's confidence by one tier.
- **Parallelism is mandatory.** All reviewers in Phase 3 spawn in a single assistant turn; all claim batches in Phase 2 run in parallel up to the concurrency cap.
- **Stop early on blockers.** If Phase 1 cannot extract structured text from the paper (corrupted PDF, unsupported encoding), stop and report. Do not fabricate a review of the title and abstract.

## Failure modes to guard against

- **Praise inflation**: reviewer writes glowing summary but gives low Overall. Enforce: Overall ≤ min(rubric sub-scores) + 1.
- **Hallucinated prior work**: reviewer claims "X et al. 2021 already did this." Every such claim must carry a Semantic Scholar paperId in the verification bundle.
- **Rebuttal rubber-stamp**: reviewer raises scores after a rebuttal that added nothing. The meta-reviewer must audit each raised score against `authors-response.md`.
- **Rebuttal hallucination**: the rebuttal cites a table, figure, section, or number that does not actually exist in the paper, or misstates an existing one. Caught by Phase 4.5; the meta-review must explicitly call this out, reverse any score recovery built on the hallucinated reference, and link to the offending entry in `rebuttal/verification/audit.md`.
- **Language drift**: reviewer flips between English and Korean within a paragraph. Enforce `--lang`.
- **Skipping verification**: if `--verify=true` and the bundle is empty, re-run Phase 2 before Phase 3.
- **PDF extraction glitches**: if more than 10% of extracted lines look like garbled OCR, stop and ask the user for the TeX source.

## Output

- Full session bundle under `.omc/reviews/{timestamp}-{paper-stem}/`.
- `report.md` as the single entry point.
- Short terminal summary: path to bundle, final recommendation, average score, top-3 blockers.

## Related skills

- `/oh-my-claudecode:verify` — reuse for double-checking that the review bundle on disk is complete before emitting the summary.
- `/oh-my-claudecode:external-context` — for standalone literature lookups outside a review session.
- `/oh-my-claudecode:sciomc` — if the user wants a broader research audit rather than a single-paper review.
- `/oh-my-claudecode:methodology-writer` — pair after acceptance to update the methodology section.
