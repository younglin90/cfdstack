# cfdstack development

## Commands

```bash
./setup          # register skills with Claude Code (~/.claude/skills/)
./setup --local  # register skills into .claude/skills/ of the current project
```

## Project structure

```
cfdstack/
├── setup                   # install script: symlinks skills into ~/.claude/skills/
├── SKILL.md                # meta-skill: /cfdstack overview and navigation
├── CLAUDE.md               # this file — development guide
├── README.md               # public-facing documentation
│
│ ── Simulation skills (기존 솔버 사용) ──
├── cfd-new/SKILL.md        # /cfd-new — scaffold a new CFD case
├── cfd-check/SKILL.md      # /cfd-check — validate mesh and case setup
├── cfd-run/SKILL.md        # /cfd-run — run solver and monitor convergence
├── cfd-post/SKILL.md       # /cfd-post — post-process results
├── cfd-debug/SKILL.md      # /cfd-debug — debug convergence issues
├── cfd-validate/SKILL.md   # /cfd-validate — validate against benchmarks
├── cfd-report/SKILL.md     # /cfd-report — generate simulation report
│
│ ── Solver dev skills (솔버 개발) ──
├── cfd-paper-to-code/SKILL.md   # /cfd-paper-to-code — 논문→코드 변환
├── cfd-vv/SKILL.md              # /cfd-vv — 검증 및 확인 (V&V)
│   └── benchmarks.md            # 표준 벤치마크 DB
├── cfd-debug-adv/SKILL.md       # /cfd-debug-adv — 솔버 디버깅 심화
├── cfd-flux/SKILL.md            # /cfd-flux — flux 함수 구현
├── cfd-reconstruction/SKILL.md  # /cfd-reconstruction — 고차 재구성
├── cfd-time-integration/SKILL.md # /cfd-time-integration — 시간 적분
├── cfd-implicit/SKILL.md        # /cfd-implicit — implicit 솔버
├── cfd-euler/SKILL.md           # /cfd-euler — Euler 방정식
├── cfd-ns/SKILL.md              # /cfd-ns — Navier-Stokes
└── cfd-turbulence/SKILL.md      # /cfd-turbulence — 난류 모델
```

## Adding a new skill

1. Create a new directory: `mkdir my-skill`
2. Add `my-skill/SKILL.md` with YAML frontmatter (`name`, `version`, `description`, `allowed-tools`)
3. Add the skill name to the `SKILLS` array in `setup`
4. Run `./setup` to register it
5. Add the skill to the list in `SKILL.md` and `README.md`

## SKILL.md format

Each skill is a Markdown prompt template with YAML frontmatter:

```markdown
---
name: skill-name
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /skill-name.
  What this skill does and when to use it.
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

## Step 1: ...

Instructions for Claude to follow...
```

The `description` field is what Claude Code reads to decide when to invoke the skill.
Always start with `MANUAL TRIGGER ONLY: invoke only when user types /skill-name.`

## Platform notes

- Skills work on macOS, Linux, and Windows (Git Bash / WSL)
- No compiled binaries required — pure Markdown prompt templates
- Skills adapt to the user's installed CFD software by checking PATH and reading CLAUDE.md

## cfdstack

Simulation: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report
Solver dev: /cfd-paper-to-code, /cfd-vv, /cfd-debug-adv, /cfd-flux, /cfd-reconstruction, /cfd-time-integration, /cfd-implicit, /cfd-euler, /cfd-ns, /cfd-turbulence

## 작업 이력

### v2.0.0 — 솔버 개발 스킬 추가
- 10개 솔버 개발 스킬 추가 (논문→코드, V&V, 디버깅, flux, 재구성, 시간적분, implicit, Euler, NS, 난류)
- 메타 스킬 `/cfdstack` 업데이트: 시뮬레이션 + 솔버 개발 이중 트랙
- `cfd-skill-forge` 제거 (개발 완료 후 불필요)
- `cfd-solver-dev` 제거 (전문 스킬 10개로 대체)

### v1.0.0 — 초기 구성
- 7개 CFD 시뮬레이션 스킬 작성
- 메타 스킬 `/cfdstack`, `setup` 스크립트
- 모든 SKILL.md 본문을 한글로 작성
