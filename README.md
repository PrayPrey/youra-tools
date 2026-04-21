# youra-tools

Claude Code plugin providing six skills for the academic paper lifecycle —
from **idea** (gap-finder) through **implementation writeup** (methodology-writer),
**empirical writeup** (experiments-writer), **submission** (paper-review),
**feedback-driven revision** (revision-loop), and
**pre-submission stress-test orchestration** (submission-ready).

Every skill shares the same DNA: **evidence-first + loop-until-stable + no silent success.**

## Skills

### `/youra-tools:gap-finder`

Given an idea or topic, runs a **literature survey + research-gap identification** with a hallucination-proof verification loop. Fans out parallel Semantic Scholar + Exa/Web searches, clusters the hits, surfaces ranked gaps, and re-runs until **every cited paper is confirmed to actually exist**. No phantom citations.

- Invocation: `/youra-tools:gap-finder --idea="<topic or research idea>"`
- Defaults: `--depth=standard --max-papers=40 --years=7 --verify=strict --max-rounds=5`
- Output: `.omc/surveys/{timestamp}-{slug}/report.md` with TL;DR, cluster map, ranked gaps, verified reading list, verification summary.
- Triggers (Korean): `갭 찾기`, `연구 갭`, `논문 조사`, `서베이`, `문헌 조사`.

### `/youra-tools:methodology-writer`

Reads an implementation folder (`src/`, `방법론 (구현한)/`, a submodule) and writes or updates a **Methodology section** so that it faithfully describes what the code actually does. Loops **edit → verify** until a reviewer agent signs off — no phantom components, no hand-waving.

- Invocation: `/youra-tools:methodology-writer --impl=<dir> --out=<methodology.md>`
- Defaults: `--max-rounds=5 --reviewer=architect`
- Output: updated `.md`/`.tex` file + short reviewer-signed report (no full-section re-paste).
- Triggers (Korean): `방법론`, `methodology`.

### `/youra-tools:experiments-writer`

Reads experiment result `.md` files plus a paper's **Experiments section**, then loops **analysis → edit → verify** until every number, table, and interpretation in the section is faithful to the actual results. Rewrites flawed analyses too — not only wrong numbers.

- Invocation: `/youra-tools:experiments-writer --results=<dir> --paper=<file.tex>`
- Defaults: `--section=Experiments --max-rounds=5 --reviewer=architect`
- Output: updated `.tex`/`.md` file with a reviewer-signed diff summary.
- Triggers (Korean): `실험 섹션`, `실험 파트`.

### `/youra-tools:paper-review`

**Strict peer-review simulation** with ACL / NeurIPS / EMNLP reviewer personas — objective rubric scoring, Semantic Scholar + Exa/web fact verification, and a rebuttal lane that challenges author responses with evidence. Brutal honesty with evidence, no praise inflation.

- Invocation: `/youra-tools:paper-review --paper=<file.pdf|tex|md>`
- Defaults: `--venue=mixed --severity=strict --mode=full --num-reviewers=3 --verify=true`
- Modes: `review` | `rebuttal` (attach `--rebuttal=<path>`) | `full` (review → simulated authors' response → meta-review).
- Output: `.omc/reviews/{timestamp}-{paper-stem}/report.md` with per-venue reviews, verification bundle, meta-review, and a final recommendation (`Accept` / `Weak Accept` / `Borderline` / `Weak Reject` / `Reject`).
- Triggers (Korean): `논문 리뷰`, `피어 리뷰`, `리뷰어 시뮬레이션`, `리뷰탈`.

### `/youra-tools:revision-loop`

Takes **feedback** or a **list of fix-items** for an academic document (a `.tex` paper, a `.md` writeup, or a reviewer bundle produced by `/youra-tools:paper-review`), decomposes it into discrete issues, and loops **edit → verify** until a reviewer agent signs off on every issue. Specialization of the OMC `ralph` persistence loop for the feedback-driven revision case: each issue becomes one PRD story with `passes: false` until the reviewer PASS verdict lands on disk. No self-approval, no silent success — if `--max-rounds` is hit with blockers remaining, the skill reports `BLOCKED` rather than faking completion.

- Invocation: `/youra-tools:revision-loop --feedback=<path|text|review-dir> --target=<file.tex|file.md>`
- Defaults: `--max-rounds=10 --reviewer=architect --lang=(inspect target)`
- Output: `.omc/revisions/{timestamp}-{slug}/report.md` with resolved arguments, rounds used, per-issue verdicts, and residual non-blocking suggestions; target file updated in place.
- Triggers (Korean): `수정 루프`, `피드백 반영`, `반복 수정`, `리비전 루프`, `피드백 적용`.

### `/youra-tools:submission-ready`

**Pre-submission stress-test orchestrator.** Takes a completed draft and loops `/youra-tools:paper-review` → `/youra-tools:revision-loop` → `/youra-tools:paper-review` until the meta-review recommendation reaches `--target-grade` (default `Weak Accept`), plateaus for `--converge-window` cycles (default `2`), hits `--max-cycles` (default `3`), or a sub-skill returns `BLOCKED`. Each cycle persists the reviewer bundle, the revision session, and the parsed ladder index (`Reject` < `Weak Reject` < `Borderline` < `Weak Accept` < `Accept`) so the grade trajectory is auditable. The orchestrator cannot declare `SUCCESS` without the final post-review recommendation actually hitting `--target-grade` — cycle-over-cycle improvement alone is not enough.

- Invocation: `/youra-tools:submission-ready --paper=<file.tex|file.md>`
- Defaults: `--target-grade="Weak Accept" --max-cycles=3 --converge-window=2 --venue=mixed --severity=strict --reviewer=architect`
- Output: `.omc/submission-ready/{timestamp}-{slug}/report.md` with terminal state (`SUCCESS` / `CONVERGED` / `MAX_CYCLES` / `BLOCKED`), per-cycle grade table, residual blockers, and next-step guidance; per-cycle sub-skill bundles linked from `cycle-{k}/`.
- Triggers (Korean): `제출 준비`, `제출 직전 검증`, `스트레스 테스트`, `논문 마감 루프`.

## Typical workflow

```
     idea                        implementation               results
       │                              │                          │
       ▼                              ▼                          ▼
/youra-tools:gap-finder   /youra-tools:methodology-writer   /youra-tools:experiments-writer
       │                              │                          │
       └──────────────────────────────┴──────────────────────────┘
                                      │
                                      ▼
                         ┌─── /youra-tools:submission-ready ───┐
                         │                                     │
                         │  per cycle:                         │
                         │   1. /youra-tools:paper-review ────▶ reviewer bundle
                         │   2. /youra-tools:revision-loop ──── apply feedback
                         │   3. /youra-tools:paper-review ──── re-grade
                         │                                     │
                         │  exits on:                          │
                         │   SUCCESS  (grade ≥ target)         │
                         │   CONVERGED (plateau)               │
                         │   MAX_CYCLES | BLOCKED              │
                         └─────────────────────────────────────┘
```

`submission-ready` is the pre-submission entry point; the two sub-skills can also be used standalone when you want manual control over the handoff.

## Installation

### From GitHub (recommended)

Claude Code plugin installs go through a **marketplace registration** step first, then install the plugin by `<plugin-name>@<marketplace-name>`. Since `youra-tools` ships its own `.claude-plugin/marketplace.json`, the full flow is two commands:

```
/plugin marketplace add PrayPrey/youra-tools
/plugin install youra-tools@youra-tools
```

After that, confirm it's enabled (`/plugin` opens the manager UI) and all six skills are callable as `/youra-tools:<name>`.

### Local development (this machine)

For iterating on the plugin locally without pushing to GitHub, register the local marketplace directory first:

```
/plugin marketplace add C:/Users/OWNER/Desktop/Woo_Yoon_Kyu/Paper_review_workflow/youra-tools
/plugin install youra-tools@youra-tools
```

### Updating

```
/plugin marketplace update youra-tools
/plugin update youra-tools@youra-tools
```

## Requirements

All six skills benefit from — but do not require — these MCP servers:

- **Semantic Scholar MCP** (`mcp__hamid-vakilzadeh-mcpsemanticscholar__*`) — academic paper lookup, citation graph, arXiv access.
- **Exa MCP** (`mcp__exa__*`) — web-grounded context, GitHub code, model cards.

If neither is available, the skills fall back to built-in `WebSearch` / `WebFetch` with lowered confidence tiers.

The skills integrate cleanly with [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) (OMC) — they delegate to OMC agents (`scientist`, `architect`, `critic`, `verifier`, `executor`) when available. OMC is strongly recommended but not required.

## Status

Pre-1.0. API (flag names, default thresholds) may change before 1.0.

## License

MIT — see `LICENSE`.
