---
name: gap-finder
description: Literature survey + research-gap identification with a hallucination-proof verification loop. Given an idea or topic, fans out Semantic Scholar + Exa/Web searches, clusters the hits, surfaces ranked gaps, and re-runs until every cited paper is confirmed to actually exist.
triggers:
  - gap finder
  - gap-finder
  - find the gap
  - research gap
  - literature survey
  - literature review
  - lit survey
  - lit review
  - 갭 찾기
  - 리서치 갭
  - 연구 갭
  - 논문 조사
  - 문헌 조사
  - 서베이
  - 갭 분석
argument-hint: "[--idea=<text or file>] [--depth=shallow|standard|deep] [--max-papers=<N>] [--years=<N>] [--verify=strict|standard|lenient] [--max-rounds=<N>] [--out=<path>] [--lang=en|ko]"
level: 4
---

# Gap Finder

Use this skill when the user hands over a research idea or topic and wants:

1. a **literature survey** of the surrounding field,
2. a ranked **gap analysis** pointing to what has not yet been done, and
3. a **hallucination-proof guarantee** that every paper cited in the survey actually exists.

The hard requirement is verification. The user explicitly asked that, after searching and organizing the results, the skill re-verifies every cited paper and keeps looping until each one is confirmed to exist via a real source. Never short-circuit this loop to save time.

## Default invocation

When called with no flags, resolve to:

```
--depth=standard --max-papers=40 --years=7 --verify=strict --max-rounds=5
```

If `--idea` is missing, prompt once with a single question: *"What idea or topic should I survey?"*. Accept a one-liner, a paragraph, or a path to a `.md`/`.txt`/`.tex` file. Every other flag falls back to the default without asking.

## Goal

Produce a reproducible survey bundle that:

1. **Maps** the field around the idea — key clusters, trajectories, competing approaches.
2. **Cites only real papers** — each with a Semantic Scholar `paperId` or an accessible authoritative URL.
3. **Names concrete gaps** — missing intersections, unexplored methodologies, unchallenged assumptions, stale baselines.
4. **Tells the user what to read next** — a ranked "first N to read" list, shortest path to the top gap.

## When to invoke

- The user has an idea for a new paper, thesis, or project and wants to know what has already been done.
- The user wants a gap list so they can pick the right research question.
- The user wants to stress-test a hypothesis against recent literature before drafting.
- The user wants to expand an existing related-work section with verified citations.

Do **not** invoke for: a single known paper (use `paper-review`), a quick web fact lookup (use `external-context`), or generic code research (use `sciomc`).

## Arguments

Parse the invocation string. All arguments are optional — ask once with concrete candidates when missing.

| Flag | Meaning | Default |
|------|---------|---------|
| `--idea=<text \| path>` | Research idea or topic. Accept raw text or a file path when longer than a line. | Prompt once. |
| `--depth=<d>` | `shallow` (~15 papers, no citation recursion), `standard` (~40 papers, 1-hop citation ring on seeds), `deep` (~80 papers, 2-hop recursion on the densest cluster). | `standard` |
| `--max-papers=<N>` | Hard cap on surveyed papers across all facets. | 40 |
| `--years=<N>` | Recency window: restrict searches to the last N years, with a small "classics" slot reserved for foundational work regardless of age. | 7 |
| `--verify=<s>` | `strict` (VERIFIED requires matching `paperId` + abstract), `standard` (paperId OR matching abstract), `lenient` (any authoritative URL). | `strict` |
| `--max-rounds=<N>` | Maximum verification iterations before stopping. | 5 |
| `--out=<path>` | Output directory for the survey bundle. | `.omc/surveys/{timestamp}-{slug(idea)}/` |
| `--lang=<en\|ko>` | Output language for the report. | Inspect the idea text; fallback `en`. |

## Session artefacts

```
.omc/surveys/{timestamp}-{slug(idea)}/
  state.json                 # session state + resolved arguments
  idea.md                    # captured idea verbatim + derived facets
  facets/
    facet-{N}.md             # per-facet search log and raw hits
  papers/
    paper-{paperId}.json     # normalized paper record
    index.json               # id -> metadata index
  verification/
    round-{k}/
      verdicts.json          # per-paper verification status this round
      summary.md             # round summary
    summary.md               # final verification summary
  gaps/
    cluster-map.md           # topic clusters with representative papers
    gap-list.md              # ranked gaps with evidence pointers
  report.md                  # top-level human-readable bundle
```

## Workflow

### Phase 0 — Resolve arguments and capture the idea

1. Resolve `--idea`. If it is a file path, read the file; otherwise take it as raw text.
2. Write `idea.md`:
   - A one-sentence distillation of the thesis.
   - Key entities (tasks, datasets, methods, domains, metrics).
   - What success would look like ("if the paper I want existed, it would do X").
3. Create the session directory and `state.json` with the resolved arguments and a timestamped run id.

### Phase 1 — Facet decomposition

Decompose the idea into **3-7 independent search facets**. Good facets are distinct enough not to duplicate results but together cover the relevant literature. Pick from (not all required):

- **Core** — the central method/task itself.
- **Adjacent** — competing or complementary approaches to the same problem.
- **Foundations** — canonical prior work the idea builds on.
- **Recent** — last 2-3 years only, to catch still-unindexed preprints.
- **Cross-domain** — the same idea in other subfields (NLP → vision, RL → robotics, etc.).
- **Critiques** — papers that challenge the assumptions behind the idea.
- **Evaluation** — benchmarks, datasets, metrics used to measure the idea.

Write each facet to `facets/facet-{N}.md` with the search query, target sources, year filter, and the target number of hits.

### Phase 2 — Parallel search

Fire one `oh-my-claudecode:scientist` agent per facet **in a single assistant turn**. Each agent follows this tool-preference order:

1. **Semantic Scholar MCP** (primary for academic papers):
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__paper-search-advanced` — ranked keyword search with year/venue filters.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-search-basic` — shallower keyword search when the advanced one returns nothing.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__search-arxiv` — preprints, often fresher than the indexed record.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__search-paper-title` — when a facet names a specific known paper.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-citations` / `papers-references` — 1-hop expansion on seed papers.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__analysis-citation-network` — structural view of clusters for `--depth=deep`.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-batch` — batch metadata fetch for 5+ paperIds.
   - `mcp__hamid-vakilzadeh-mcpsemanticscholar__get-paper-abstract` — one-off abstract lookup during ingest.
2. **Exa MCP** if tools appear as `mcp__exa__*` in the deferred-tool list. Use for web-grounded material (blog posts, threads, model cards) that peer review does not cover.
3. **WebSearch** + **WebFetch** as graceful fallback when neither MCP is available. Fallback hits inherit one confidence tier lower than MCP hits.

Each scientist writes its raw hits back to `facets/facet-{N}.md` with a consistent record per hit:

```json
{
  "title": "...",
  "authors": ["..."],
  "year": 2024,
  "venue": "ACL",
  "paperId": "...",
  "abstract": "...",
  "url": "https://...",
  "source_tool": "semantic-scholar | exa | websearch",
  "relevance_note": "one sentence on why this is relevant to the facet"
}
```

Maximum 5 parallel scientists at once. If a facet returns zero hits, widen the query once before giving up, and record the widened query in the facet log.

### Phase 3 — Normalize, deduplicate, cluster

1. Merge all facet hits into `papers/index.json`. Deduplicate first by `paperId`, then by fuzzy title+year match (Levenshtein ratio > 0.9 on normalized titles).
2. For any paper without an abstract, fetch it via `get-paper-abstract` or `download-full-paper-arxiv`. If the abstract still cannot be obtained, mark `abstract_missing: true` — that paper may only appear in supporting context, never as a headline citation.
3. Cluster papers by theme + method + era. Produce `gaps/cluster-map.md`:
   - Named clusters (e.g. "method-A family", "dataset-B era", "critique line").
   - Representative papers per cluster (2-5 each) with one-sentence summaries.
   - Chronological spine within each cluster.

### Phase 4 — Gap analysis

Spawn one `oh-my-claudecode:architect` (model=opus) agent. Give it `idea.md`, the cluster map, and the full paper index. Ask it to emit a ranked gap list with this structure per gap:

```markdown
### Gap G{N}: <one-line title>

- **What's missing:** <precise description>
- **Evidence it's missing:** <which clusters covered what, and what they did not>
- **Why it matters:** <payoff if filled>
- **Hardness:** LOW | MEDIUM | HIGH
- **Adjacent papers:** [paperId, paperId] (the nearest neighbours; the reader starts here)
- **Proposed angle:** <one paragraph on how the user's idea could plausibly address this gap>
```

Save to `gaps/gap-list.md`. Rank gaps by a rough `(payoff × novelty) / hardness` heuristic; the architect must explain the ranking in a final "ranking rationale" paragraph.

### Phase 5 — Verification loop (mandatory)

This is the user's hard requirement. Run until **every paper cited in `gaps/gap-list.md` and `gaps/cluster-map.md`** is VERIFIED, or `--max-rounds` is hit. "Cited once, checked once" is not enough — losing a paper forces a refill and another verify pass.

For round `k`:

1. **Per-paper verification** (parallel, up to 5 scientists, batched):
   - Look up `paperId` via `mcp__hamid-vakilzadeh-mcpsemanticscholar__papers-batch` — success = real paper, fields match the index.
   - If no `paperId`, resolve by title via `search-paper-title`; accept only when the top hit's title + first author + year match within tolerance (≤1-year mismatch allowed for preprint vs published).
   - If Semantic Scholar cannot confirm, fall back to Exa / WebSearch — accept only authoritative hosts (arxiv.org, aclanthology.org, openreview.net, proceedings.mlr.press, publisher domains, ACM/IEEE/Springer digital libraries, recognized university or lab domains). Blog posts, aggregators, and slide decks do **not** count under `--verify=strict`.
   - Emit per paper: `VERIFIED | WRONG_METADATA | HALLUCINATED | UNREACHABLE` with evidence (paperId, URL, abstract excerpt, or the negative lookup trace).

2. **Write `verification/round-{k}/verdicts.json`** and a human-readable `summary.md`.

3. **Act on non-VERIFIED papers:**
   - `WRONG_METADATA`: correct title / authors / year / venue in place from the verification evidence, re-verify next round.
   - `HALLUCINATED`: remove the paper from `papers/index.json`, from `cluster-map.md`, and from any gap that cites it. Record in `verification/summary.md` that it was dropped and why.
   - `UNREACHABLE`: try one alternative lookup (e.g., arXiv id if Semantic Scholar timed out, or vice versa); if still unreachable, downgrade to `UNVERIFIED CANDIDATE` in the final report — it may never be a headline citation.

4. **Refill.** For every paper dropped or downgraded, re-run a tiny targeted search on its facet to replace it with a verified neighbour. Mark refilled entries so the next round verifies them too.

5. **Stop condition.** Exit when every headline citation is VERIFIED, or when `k == --max-rounds`. If the max is hit with remaining HALLUCINATED / UNREACHABLE entries, the final report must clearly mark them and explain why they could not be verified. **Never silently claim success.**

### Phase 6 — Verifier pass (independent reviewer)

Spawn one `oh-my-claudecode:verifier` (or `critic`) agent that does **not** see Phase 5's self-report. Give it `papers/index.json`, the final `cluster-map.md`, and `gap-list.md`. It must:

- Pick 5 random citations, independently re-verify them via Semantic Scholar / Exa / Web, and report agreement / disagreement.
- Check that every gap cites at least two real clusters' papers.
- Return `PASS` (publication-ready) or `FAIL` (with the specific blocking citations).

If `FAIL`, go back to Phase 5 for one more round on just the blocking citations, then re-run the verifier. If it still `FAIL`s, surface the blockers in the final report rather than hiding them.

### Phase 7 — Report

Write `report.md` with:

- **TL;DR** (4-6 lines): the idea, the single most promising gap, the must-read papers.
- **Idea restatement** (from `idea.md`).
- **Cluster map** (embedded from `gaps/cluster-map.md`).
- **Ranked gaps** (embedded from `gaps/gap-list.md`).
- **Reading list**: 5-10 papers that shortest-path the user to the top gap, with one-sentence why-read notes and verified URLs / paperIds.
- **Verification summary**: total papers, VERIFIED / WRONG_METADATA-fixed / HALLUCINATED-dropped / UNREACHABLE-flagged counts, rounds used, verifier verdict.
- **Unverified candidates** (separate, clearly labelled section, only if any remain): title + last known metadata + why it could not be verified.
- **Limitations**: what the survey did not cover and why (non-English venues, paywalled journals, outage during a facet run, etc.).

Print a ≤20-line terminal summary: path to `report.md`, top gap title, reading-list size, verification counts, verifier verdict.

## Rules

- **No phantom papers.** A paper whose final verdict is not VERIFIED cannot appear as a headline citation in `report.md`. It may only appear, if at all, in the clearly separated "Unverified candidates" section.
- **Loop until stable.** The verification loop does not exit because every paper was "checked once" — it exits when every headline citation is VERIFIED, or `--max-rounds` is reached. Dropping a paper forces a refill and another pass.
- **Evidence over vibes.** Every claim in the gap list must point to specific cluster(s) or paperId(s). No gap may be asserted from general impression.
- **Real sources only under strict.** Under `--verify=strict`, aggregator blogs and secondhand summaries do not count as evidence of existence. Use the original venue or the arXiv id.
- **Fallback transparency.** If both Semantic Scholar and Exa fail for a facet, the report must say so in the Limitations section and drop the affected facet's confidence tier.
- **Language consistency.** If `--lang=ko`, the report is in Korean but keeps paper titles, author names, venue abbreviations, and equation-like technical terms in their original form.
- **Parallelism is mandatory.** Facet searches in Phase 2 and per-paper verification in Phase 5 fan out in a single assistant turn, up to the 5-agent cap.
- **Independent verifier.** Phase 6 is a separate lane — never self-approve. Even a perfect Phase 5 must still pass the verifier.
- **Stop early on blocked input.** If the idea is unintelligible (one vague word, empty, self-contradictory), do not survey — ask one clarifying question instead.

## Failure modes to guard against

- **Hallucinated paper titles.** LLM-generated "related work" often invents plausible-sounding papers. The verification loop exists specifically to catch these — never short-circuit it, even when time-pressured.
- **Wrong-year / wrong-venue metadata.** arXiv preprint years often differ from published years. Accept a ≤1-year mismatch *only* when the rest of the identity matches; any larger gap means the metadata is wrong and needs fixing.
- **Citation ring cascades.** `--depth=deep` can explode if a seed paper has thousands of citations. Cap 1-hop expansion at 50 papers per seed; only expand a 2nd hop on the top cluster.
- **Gap inflation.** Every "gap" must point at what the covered clusters did *not* do. An unsupported "no one has tried X" claim is a failure mode — drop it or find citation-ring evidence.
- **Single-cluster tunnel vision.** If more than 80% of surveyed papers land in one cluster, the facet decomposition was too narrow — re-decompose before gap analysis.
- **Silent MCP failure.** If Semantic Scholar returns empty for everything, assume a rate-limit or outage and surface it explicitly — do not silently fall back to WebSearch and present the result as a peer-reviewed survey.
- **Verifier rubber-stamp.** Phase 6 must actually re-lookup the sampled citations. A verifier report without concrete URLs/paperIds for the sampled checks is itself a failure.

## Output

- Session bundle under `.omc/surveys/{timestamp}-{slug(idea)}/`.
- `report.md` as the single human entry point.
- Short terminal summary: path, top gap, reading-list size, verification counts, rounds used, verifier verdict.

## Related skills

- `/oh-my-claudecode:paper-review` — once the user has drafted the paper that fills the gap, run a strict peer review.
- `/oh-my-claudecode:external-context` — lighter, one-shot web lookup when the user does not need a full survey or gap analysis.
- `/oh-my-claudecode:sciomc` — broader research orchestration when the user's question is not literature-bound.
- `/oh-my-claudecode:methodology-writer` — pairs well after a gap is chosen and the methodology section needs drafting.
- `/oh-my-claudecode:verify` — generic verification pattern; this skill uses a survey-tailored variant.
