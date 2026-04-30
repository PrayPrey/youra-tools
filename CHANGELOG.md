# Changelog

## 0.5.0 — 2026-04-30

Added rebuttal-side fact checking and a bounded revision loop to `paper-review` so simulated/user rebuttals cannot smuggle in claims the paper does not actually support.

- `paper-review` — new **Phase 4.5 (Rebuttal verification)**: an isolated `scientist` (opus) subagent audits every factual assertion in the rebuttal against the paper text and the Phase 2 verification bundle, emitting per-claim verdicts (`SUPPORTED` / `UNSUPPORTED` / `MISREPRESENTED` / `NEW_CLAIM`) and a `mismatch_reason` that quotes the paper passage which contradicts the claim. Persisted to `rebuttal/verification/{rebuttal-claims.json, audit.md, summary.md}`. Main context only re-reads the summary so the loop stays clean. UNSUPPORTED / MISREPRESENTED entries roll back the corresponding post-rebuttal score recovery; out-of-block `NEW_CLAIM` is treated as a rebuttal-rule violation.
- `paper-review` — new **Phase 4.6 (Rebuttal revision loop)**: feeds the audit findings back into the rebuttal lane and reruns Phase 4 → 4.5 up to `--rebuttal-max-revisions` (default `2`) times. Stops on `CONVERGED`, `EXHAUSTED`, `NO_PROGRESS`, or `BLOCKED`; the best iteration is promoted to `rebuttal/` and per-iteration history is preserved under `rebuttal/iterations/iter-{i}/` with a `convergence.md` recording the stop reason and carry-over blockers. For `--mode=rebuttal` the user's text is never rewritten — the loop emits a `required-fixes.md` and stops after iter 0.
- `paper-review` — new flags `--rebuttal-max-revisions=N` (default `2`, `0` disables the loop) and `--rebuttal-revise-on=<verdicts>` (default `UNSUPPORTED,MISREPRESENTED,NEW_CLAIM_OUT_OF_BLOCK`). New rules ("Rebuttals are verified, not trusted", "Rebuttal revision loop is bounded") and a new failure mode ("Rebuttal hallucination") added to the skill spec.
- Catches up `marketplace.json` to match the previously published `0.4.1` `revision-loop` text-only default — installs that pinned by version were not picking that up.

## 0.4.1 — 2026-04-28

Made `revision-loop` default to text-only fixes so reviewer items requiring new experiments cannot trigger fabricated numbers.

- `revision-loop` — added `--scope=text-only|all` (default `text-only`). Each decomposed issue is now classified with `requires_new_experiment`; experiment-class items (new training runs, datasets, ablations, baselines, or any new numerical result not already on disk) are routed to a new `deferred.json` and reported under a required "Deferred — requires new experiment" section in `report.md` instead of entering the PRD. `--scope=all` restores the legacy behaviour. Adds a fabricated-results failure mode to the reviewer checklist and a misclassification mitigation (when in doubt, defer; user can rerun with `--scope=all`).

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
