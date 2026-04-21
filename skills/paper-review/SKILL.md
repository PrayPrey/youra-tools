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
--venue=mixed --severity=strict --mode=full --num-reviewers=3 --verify=true
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
    authors-response.md          # either user-provided or simulated
    reviewer-ACL-final.md        # post-rebuttal verdict per reviewer
    ...
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

### Phase 5 — Aggregate and report

1. Write `scores.json` with machine-readable scores per reviewer per rubric dimension, before and after rebuttal.
2. Write `report.md`:
   - Executive summary (2 paragraphs: one on contribution, one on remaining blockers).
   - Per-reviewer section: rubric table, key weaknesses, remaining concerns after rebuttal.
   - Verification summary: how many claims VERIFIED / CONTRADICTED / UNVERIFIED.
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
- **Language consistency.** If `--lang=ko`, reviews are in Korean but keep equation labels, table numbers, citations, and English-origin technical terms in their original form.
- **No silent success.** If verification could not run (MCP down, no network), say so in `verification/summary.md` and downgrade every affected reviewer's confidence by one tier.
- **Parallelism is mandatory.** All reviewers in Phase 3 spawn in a single assistant turn; all claim batches in Phase 2 run in parallel up to the concurrency cap.
- **Stop early on blockers.** If Phase 1 cannot extract structured text from the paper (corrupted PDF, unsupported encoding), stop and report. Do not fabricate a review of the title and abstract.

## Failure modes to guard against

- **Praise inflation**: reviewer writes glowing summary but gives low Overall. Enforce: Overall ≤ min(rubric sub-scores) + 1.
- **Hallucinated prior work**: reviewer claims "X et al. 2021 already did this." Every such claim must carry a Semantic Scholar paperId in the verification bundle.
- **Rebuttal rubber-stamp**: reviewer raises scores after a rebuttal that added nothing. The meta-reviewer must audit each raised score against `authors-response.md`.
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
