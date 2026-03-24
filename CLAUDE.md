# cfdstack development

## Commands

```bash
./setup          # register skills with Claude Code (~/.claude/skills/)
./setup --local  # register skills into .claude/skills/ of the current project
```

## Project structure

```
cfdstack/
├── setup             # install script: symlinks skills into ~/.claude/skills/
├── SKILL.md          # meta-skill: /cfdstack overview and navigation
├── CLAUDE.md         # this file — development guide
├── README.md         # public-facing documentation
├── cfd-new/
│   └── SKILL.md      # /cfd-new — scaffold a new CFD case
├── cfd-check/
│   └── SKILL.md      # /cfd-check — validate mesh and case setup
├── cfd-run/
│   └── SKILL.md      # /cfd-run — run solver and monitor convergence
├── cfd-post/
│   └── SKILL.md      # /cfd-post — post-process results
├── cfd-debug/
│   └── SKILL.md      # /cfd-debug — debug convergence issues
├── cfd-validate/
│   └── SKILL.md      # /cfd-validate — validate against benchmarks
└── cfd-report/
    └── SKILL.md      # /cfd-report — generate simulation report
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

Available skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report

## 작업 이력

### v1.0.0 — 초기 구성
- 7개 CFD 스킬 작성: `/cfd-new`, `/cfd-check`, `/cfd-run`, `/cfd-post`, `/cfd-debug`, `/cfd-validate`, `/cfd-report`
- 메타 스킬 `/cfdstack`: 전체 워크플로 안내 및 스킬 네비게이션
- `setup` 스크립트: `~/.claude/skills/`에 심링크 등록 (`--local` 옵션으로 프로젝트 로컬 등록 가능)
- 모든 SKILL.md 본문을 한글로 작성 (YAML frontmatter의 `name`/`description` 트리거 키워드는 영문 유지)
