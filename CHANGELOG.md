# Changelog

## 0.4.0 — 2026-04-21

Added a pre-submission stress-test orchestrator that wraps the peer-review ↔ revision cycle.

- `submission-ready` — takes a completed `.tex` / `.md` draft and loops `/youra-tools:paper-review` → `/youra-tools:revision-loop` → `/youra-tools:paper-review` until the meta-review recommendation reaches `--target-grade` (default `Weak Accept`), plateaus for `--converge-window` cycles (default `2`), hits `--max-cycles` (default `3`), or a sub-skill returns `BLOCKED`. Per-cycle reviewer bundles and revision sessions are persisted under `.omc/submission-ready/{timestamp}-{slug}/cycle-{k}/` with a parsed ladder index (`Reject` < `Weak Reject` < `Borderline` < `Weak Accept` < `Accept`). The orchestrator cannot declare `SUCCESS` unless the final post-review recommendation is actually at or above `--target-grade` — cycle-over-cycle improvement alone is not enough. Interactive argument resolution upfront; sub-skill prompts forwarded verbatim mid-flow. Specialization of OMC `ralph` for the two-sub-skill convergence case.

## 0.3.0 — 2026-04-21

Added a feedback-driven revision skill that closes the loop from peer-review output back into the manuscript.

- `revision-loop` — takes feedback (inline text, a `.md`/`.txt`/`.json` file, or a `/youra-tools:paper-review` bundle directory) for a `.tex` or `.md` document, decomposes it into discrete numbered issues with per-issue acceptance criteria, and loops executor-edit → reviewer-verify until every issue has an on-disk PASS verdict. Specialization of the OMC `ralph` persistence loop for the feedback-driven revision case. No self-approval: a missing or evidence-free reviewer response counts as FAIL. If `--max-rounds` is hit with blockers remaining, the skill reports `BLOCKED` and leaves the session on disk for the user to resume — it never fakes `CLEAN`.

## 0.2.0 — 2026-04-21

Added two more skills to cover the full paper lifecycle.

- `methodology-writer` — reads an implementation folder and writes/updates a Methodology section faithful to the code; edit → verify loop until a reviewer agent signs off.
- `paper-review` — strict ACL / NeurIPS / EMNLP peer-review simulation with rubric scoring, Semantic Scholar + Exa/Web claim verification, and a rebuttal lane (author-response simulation or user-provided rebuttal).

## 0.1.0 — 2026-04-21

Initial release.

- `gap-finder` — literature survey + research-gap identification with a hallucination-proof verification loop (Semantic Scholar + Exa/Web), parallel facet search, independent verifier pass.
- `experiments-writer` — reads experiment result `.md` files and a paper's Experiments section, loops analysis → edit → verify until every number, table, and interpretation is faithful; rewrites flawed analyses as well as wrong numbers.
