# youra-tools

학술 논문 라이프사이클을 위한 6개의 skill을 제공하는 Claude Code 플러그인 —
**아이디어** 단계(gap-finder)부터 **구현 기술**(methodology-writer),
**실험 기술**(experiments-writer), **제출 리뷰**(paper-review),
**피드백 기반 수정**(revision-loop), 그리고
**제출 직전 스트레스 테스트 오케스트레이션**(submission-ready)까지.

모든 skill은 같은 DNA를 공유합니다: **evidence-first + loop-until-stable + no silent success.** (증거 우선 + 안정될 때까지 반복 + 무언의 성공 금지)

## Skills

### `/youra-tools:gap-finder`

아이디어 또는 주제를 받아, **문헌 조사 + 연구 갭 식별**을 환각 방지(hallucination-proof) 검증 루프와 함께 수행합니다. Semantic Scholar + Exa/Web 검색을 facet별로 병렬 실행하고, 결과를 클러스터링한 뒤 순위 매긴 갭을 제시하고, **인용된 모든 논문이 실제로 존재하는지 확인될 때까지** 재실행합니다. 유령 인용 없음.

- Invocation: `/youra-tools:gap-finder --idea="<topic or research idea>"`
- Defaults: `--depth=standard --max-papers=40 --years=7 --verify=strict --max-rounds=5`
- Output: `.omc/surveys/{timestamp}-{slug}/report.md` — TL;DR, 클러스터 맵, 순위화된 갭, 검증된 읽기 리스트, 검증 요약 포함.
- Triggers (Korean): `갭 찾기`, `연구 갭`, `논문 조사`, `서베이`, `문헌 조사`.

### `/youra-tools:methodology-writer`

구현 폴더(`src/`, `방법론 (구현한)/`, 서브모듈 등)를 읽고 **Methodology 섹션**을 작성하거나 갱신하여 코드가 실제로 하는 일을 충실히 기술합니다. reviewer agent가 통과시킬 때까지 **edit → verify** 루프를 반복합니다 — 유령 컴포넌트 없음, 얼버무림 없음.

- Invocation: `/youra-tools:methodology-writer --impl=<dir> --out=<methodology.md>`
- Defaults: `--max-rounds=5 --reviewer=architect`
- Output: 갱신된 `.md`/`.tex` 파일 + reviewer 서명이 포함된 짧은 리포트(전체 섹션 재출력 없음).
- Triggers (Korean): `방법론`, `methodology`.

### `/youra-tools:experiments-writer`

실험 결과 `.md` 파일들과 논문의 **Experiments 섹션**을 읽은 뒤, 섹션 안의 모든 숫자·표·해석이 실제 결과와 일치할 때까지 **analysis → edit → verify** 루프를 돕니다. 잘못된 숫자뿐 아니라 잘못된 해석도 재작성합니다.

- Invocation: `/youra-tools:experiments-writer --results=<dir> --paper=<file.tex>`
- Defaults: `--section=Experiments --max-rounds=5 --reviewer=architect`
- Output: 갱신된 `.tex`/`.md` 파일 + reviewer 서명이 포함된 diff 요약.
- Triggers (Korean): `실험 섹션`, `실험 파트`.

### `/youra-tools:paper-review`

ACL / NeurIPS / EMNLP reviewer 페르소나로 **엄격한 피어 리뷰 시뮬레이션** — 객관적 루브릭 점수, Semantic Scholar + Exa/web을 이용한 사실 검증, 그리고 저자의 응답을 증거로 반박하는 rebuttal 레인. 칭찬 인플레이션 없이, 근거에 기반한 냉정한 리뷰.

- Invocation: `/youra-tools:paper-review --paper=<file.pdf|tex|md>`
- Defaults: `--venue=mixed --severity=strict --mode=full --num-reviewers=3 --verify=true`
- Modes: `review` | `rebuttal` (`--rebuttal=<path>` 첨부) | `full` (리뷰 → 모의 저자 응답 → 메타리뷰).
- Output: `.omc/reviews/{timestamp}-{paper-stem}/report.md` — 학회별 리뷰, 검증 번들, 메타리뷰, 최종 권고(`Accept` / `Weak Accept` / `Borderline` / `Weak Reject` / `Reject`) 포함.
- Triggers (Korean): `논문 리뷰`, `피어 리뷰`, `리뷰어 시뮬레이션`, `리뷰탈`.

### `/youra-tools:revision-loop`

학술 문서(`.tex` 논문, `.md` 원고, 또는 `/youra-tools:paper-review`가 생성한 reviewer 번들)에 대한 **피드백**이나 **수정 항목 리스트**를 받아 개별 이슈로 분해한 뒤, reviewer agent가 모든 이슈에 대해 통과시킬 때까지 **edit → verify** 루프를 돕니다. OMC `ralph` 지속 루프를 피드백 기반 수정 케이스로 특화한 skill: 각 이슈는 PRD story 하나가 되며 reviewer PASS 판정이 디스크에 실제로 남을 때까지 `passes: false` 상태로 유지됩니다. 자체 승인 없음, 무언의 성공 없음 — `--max-rounds`에 도달했는데 blocker가 남아 있으면 `BLOCKED`로 보고하고 완료를 위조하지 않습니다.

- Invocation: `/youra-tools:revision-loop --feedback=<path|text|review-dir> --target=<file.tex|file.md>`
- Defaults: `--max-rounds=10 --reviewer=architect --lang=(inspect target)`
- Output: `.omc/revisions/{timestamp}-{slug}/report.md` — 해결된 인자, 사용한 라운드 수, 이슈별 판정, 블로킹이 아닌 잔여 제안 포함; 대상 파일은 in-place로 갱신.
- Triggers (Korean): `수정 루프`, `피드백 반영`, `반복 수정`, `리비전 루프`, `피드백 적용`.

### `/youra-tools:submission-ready`

**제출 직전 스트레스 테스트 오케스트레이터.** 완성된 초안을 받아 `/youra-tools:paper-review` → `/youra-tools:revision-loop` → `/youra-tools:paper-review` 사이클을, 메타리뷰 권고가 `--target-grade`(기본값 `Weak Accept`)에 도달하거나, `--converge-window` 사이클(기본값 `2`) 동안 정체되거나, `--max-cycles`(기본값 `3`)에 도달하거나, sub-skill이 `BLOCKED`를 반환할 때까지 반복합니다. 각 사이클은 reviewer 번들, revision 세션, 파싱된 래더 인덱스(`Reject` < `Weak Reject` < `Borderline` < `Weak Accept` < `Accept`)를 디스크에 남겨 성적 궤적을 감사 가능하게 합니다. 최종 post-review 권고가 실제로 `--target-grade`에 도달하지 않으면 오케스트레이터는 `SUCCESS`를 선언할 수 없습니다 — 사이클별 개선만으로는 충분하지 않습니다.

- Invocation: `/youra-tools:submission-ready --paper=<file.tex|file.md>`
- Defaults: `--target-grade="Weak Accept" --max-cycles=3 --converge-window=2 --venue=mixed --severity=strict --reviewer=architect`
- Output: `.omc/submission-ready/{timestamp}-{slug}/report.md` — 터미널 상태(`SUCCESS` / `CONVERGED` / `MAX_CYCLES` / `BLOCKED`), 사이클별 성적 표, 잔여 blocker, 다음 단계 가이드 포함; 사이클별 sub-skill 번들은 `cycle-{k}/`에서 링크.
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

`submission-ready`가 제출 직전 단일 진입점이며, 두 sub-skill은 handoff를 수동으로 제어하고 싶을 때 독립적으로도 사용할 수 있습니다.

## Installation

### GitHub에서 설치 (권장)

Claude Code 플러그인은 먼저 **marketplace 등록** 단계를 거친 뒤 `<plugin-name>@<marketplace-name>` 형식으로 설치합니다. `youra-tools`는 자체 `.claude-plugin/marketplace.json`을 제공하므로 전체 흐름은 아래 두 명령어입니다:

```
/plugin marketplace add PrayPrey/youra-tools
/plugin install youra-tools@youra-tools
```

그 다음 `/plugin`으로 플러그인 매니저 UI를 열어 활성화 상태를 확인하면, 6개 skill이 모두 `/youra-tools:<name>` 형태로 호출 가능합니다.

### 로컬 개발 (현재 머신에서)

GitHub에 push하지 않고 플러그인을 로컬에서 반복 개발할 때는, 로컬 marketplace 디렉토리를 먼저 등록합니다:

```
/plugin marketplace add C:/Users/OWNER/Desktop/Woo_Yoon_Kyu/Paper_review_workflow/youra-tools
/plugin install youra-tools@youra-tools
```

### 업데이트

```
/plugin marketplace update youra-tools
/plugin update youra-tools@youra-tools
```

## Requirements

6개 skill 모두 아래 MCP 서버의 도움을 받습니다 — 필수는 아닙니다:

- **Semantic Scholar MCP** (`mcp__hamid-vakilzadeh-mcpsemanticscholar__*`) — 학술 논문 조회, 인용 그래프, arXiv 접근.
- **Exa MCP** (`mcp__exa__*`) — 웹 기반 컨텍스트, GitHub 코드, 모델 카드.

둘 다 없는 경우 skill들은 내장 `WebSearch` / `WebFetch`로 폴백하며 confidence 등급이 한 단계 낮아집니다.

모든 skill은 [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)(OMC)와 매끄럽게 통합됩니다 — 사용 가능할 때 OMC 에이전트(`scientist`, `architect`, `critic`, `verifier`, `executor`)에 위임합니다. OMC는 강력히 권장되지만 필수는 아닙니다.

## Status

Pre-1.0. 1.0 이전에 API(플래그 이름, 기본 임계값)가 변경될 수 있습니다.

## License

MIT — `LICENSE` 파일 참조.
