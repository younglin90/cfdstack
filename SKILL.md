---
name: cfdstack
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfdstack.
  CFD simulation assistant overview. Shows available skills and guides users
  to the right workflow. Use when asked about "cfdstack", "CFD skills",
  "what CFD commands are available", or "help with CFD simulation".
  Also suggest adjacent skills by stage: new case /cfd-new; mesh check /cfd-check;
  run solver /cfd-run; post-process /cfd-post; debug issues /cfd-debug;
  validate results /cfd-validate; generate report /cfd-report.
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion

---

## cfdstack — CFD 시뮬레이션 스킬 모음

cfdstack은 Claude Code를 CFD 시뮬레이션 어시스턴트로 변환합니다. 각 스킬은 케이스 설정부터 검증 및 보고서 작성까지 CFD 워크플로의 한 단계를 담당합니다.

### 워크플로: 구상 → 설정 → 검사 → 실행 → 후처리 → 검증 → 보고

| 스킬 | 단계 | 설명 |
|------|------|------|
| `/cfd-new` | **설정** | 새 CFD 시뮬레이션 케이스를 스캐폴딩합니다. 솔버, 물리 모델, 형상을 질문한 후 경계 조건과 솔버 설정이 포함된 전체 디렉토리 구조를 생성합니다. |
| `/cfd-check` | **검사** | 실행 전 메시 품질과 케이스 설정을 확인합니다. `checkMesh` 실행, 경계 조건 검증, 누락된 필드 탐지, 일반적인 설정 오류를 잡아냅니다. |
| `/cfd-run` | **계산** | CFD 솔버를 실행하고 수렴을 모니터링합니다. 적절한 솔버를 선택하고 잔차를 관찰하며 수렴이 정체되거나 발산이 시작될 때 경고합니다. |
| `/cfd-post` | **후처리** | 결과를 추출하고 시각화를 생성합니다. 적분량을 계산하고, 잔차 플롯을 생성하며, ParaView/matplotlib 스크립트를 만듭니다. |
| `/cfd-debug` | **디버그** | 수렴 및 안정성 문제를 진단합니다. 발산 원인을 추적하고, CFL을 확인하며, 불량 메시 영역을 식별하고, 구체적인 수정 방안을 제안합니다. |
| `/cfd-validate` | **검증** | 시뮬레이션 결과를 참조 데이터 또는 해석해와 비교합니다. 오차 노름을 계산하고 불일치를 표시합니다. |
| `/cfd-report` | **보고** | 케이스 요약, 메시 통계, 수렴 이력, 주요 결과가 포함된 전문적인 시뮬레이션 보고서를 생성합니다. |

### 지원 솔버

- **OpenFOAM** (주요) — icoFoam, simpleFoam, pimpleFoam, rhoCentralFoam, buoyantSimpleFoam 등
- **SU2** — 압축성/비압축성 RANS
- **커스텀 솔버** — CLAUDE.md 설정을 읽어 어떤 솔버에도 적응합니다

### 빠른 시작

```
사용자:  Re=100 실린더 주위 유동을 시뮬레이션하고 싶어요.
사용자:  /cfd-new
Claude: [솔버, 물리 모델, Re 수, 메시 해상도 질문]
        [OpenFOAM 케이스 스캐폴딩: 0/, constant/, system/]
        [경계 조건, fvSchemes, fvSolution 설정]

사용자:  /cfd-check
Claude: [checkMesh 실행, BC 검증, 필드 차원 확인]
        [결과: 메시 OK, 종횡비 관련 경고 2개]

사용자:  /cfd-run
Claude: [icoFoam 실행, Ux/Uy/p 잔차 모니터링]
        [t=5s에서 수렴 보고]

사용자:  /cfd-post
Claude: [시간에 따른 Cd, Cl 추출]
        [잔차 + 힘 계수 matplotlib 플롯 생성]

사용자:  /cfd-validate
Claude: [Cd=1.35를 문헌값(1.33±0.05)과 비교: PASS]

사용자:  /cfd-report
Claude: [simulation_report.md 전체 보고서 생성]
```

### 설치

```bash
git clone https://github.com/younglin90/cfdstack.git ~/.claude/skills/cfdstack
cd ~/.claude/skills/cfdstack && ./setup
```

그런 다음 프로젝트의 CLAUDE.md에 추가합니다:

```
## cfdstack
Available skills: /cfd-new, /cfd-check, /cfd-run, /cfd-post, /cfd-debug, /cfd-validate, /cfd-report
```

스킬이 표시되지 않는 경우: `cd ~/.claude/skills/cfdstack && ./setup`
