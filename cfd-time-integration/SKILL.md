---
name: cfd-time-integration
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-time-integration.
  Guide for implementing time integration schemes in CFD solvers.
  Use when implementing explicit Runge-Kutta, SDIRK, BDF, or dual time stepping.
  Covers: scheme selection, Butcher tableau verification, CFL conditions,
  stability regions, and time accuracy verification.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

# 시간 적분 가이드

## 개요

시간 적분 스킴은 ODE 시스템 dQ/dt = R(Q)를 시간 전진시킨다. 선택이 솔버의 안정성, 정확도, 계산 비용을 결정한다.

**핵심 원칙:** 시간 적분 차수 ≥ 공간 이산화 차수여야 한다. 그렇지 않으면 시간 오차가 지배하여 고차 공간 스킴이 무의미해진다.

## 사전 조건

- 공간 이산화 차수 확정 (시간 차수가 이보다 같거나 높아야 함)
- 정상/비정상 문제 구분
- stiff 소스항 유무 (화학 반응, 난류 소스 등)

## 프로토콜

### Step 1: 스킴 선택

```
정상 상태(steady)?
├── YES → 시간 정확도 불필요
│   ├── explicit: Local time stepping + RK (수렴 가속)
│   └── implicit: Backward Euler + Newton-Krylov (최적)
│
└── NO (비정상)
    ├── stiff 소스항 있음?
    │   ├── YES → implicit 또는 IMEX 필수
    │   │   ├── SDIRK (implicit RK) — 고차 + A-stable
    │   │   ├── BDF2 — 다단계, 간단, 2차
    │   │   └── IMEX-RK — 대류(explicit) + 소스(implicit) 분리
    │   └── NO
    │       ├── CFL 제한이 허용 가능? → explicit RK
    │       └── CFL 제한이 심함? → implicit 또는 dual time stepping
    │
    └── 비정상 + 시간 정확도 필요?
        ├── 2차: BDF2, SDIRK2, RK2
        ├── 3차: SDIRK3, SSP-RK3
        └── 4차+: Classical RK4, SSPRK(5,4)
```

### Step 2: Explicit Runge-Kutta

**일반 형태:**
```
k_1 = R(Q^n)
k_2 = R(Q^n + Δt · a_{21} · k_1)
...
Q^{n+1} = Q^n + Δt · Σ b_i · k_i
```

**주요 스킴과 Butcher Tableau:**

**SSP-RK3 (Shu & Osher, 1988) — CFD에서 가장 많이 사용:**
```
Q^(1) = Q^n + Δt · R(Q^n)
Q^(2) = (3/4)Q^n + (1/4)Q^(1) + (1/4)Δt · R(Q^(1))
Q^{n+1} = (1/3)Q^n + (2/3)Q^(2) + (2/3)Δt · R(Q^(2))

Butcher tableau:
  0   |
  1   | 1
  1/2 | 1/4  1/4
  ----+----------
      | 1/6  1/6  2/3
```
- SSP(Strong Stability Preserving): TVD 성질 보존
- CFL 제한: CFL_SSP = 1.0 (SSP 계수 c = 1)

**Classical RK4:**
```
  0   |
  1/2 | 1/2
  1/2 | 0    1/2
  1   | 0    0    1
  ----+-------------
      | 1/6  1/3  1/3  1/6
```
- 4차 정확도, SSP 아님
- CFL 제한: ≈ 2√2 (선형 안정성)

**체크리스트 (Explicit RK):**
- [ ] Butcher tableau 계수가 논문과 일치하는가
- [ ] Σb_i = 1 인가 (일관성 조건)
- [ ] c_i = Σa_{ij} 인가 (각 단계의 시간 위치)
- [ ] SSP 스킴이면 SSP CFL 제한을 지키는가
- [ ] 중간 단계에서 경계조건을 업데이트하는가 (Q^(k)에서도 BC 적용)
- [ ] 잔차 R(Q)의 부호가 dQ/dt = R(Q) 또는 dQ/dt = -R(Q) 중 어느 규약인가

### Step 3: Implicit 스킴

#### BDF (Backward Differentiation Formula)

**BDF1 (Backward Euler):**
```
Q^{n+1} = Q^n + Δt · R(Q^{n+1})
→ Q^{n+1} - Δt · R(Q^{n+1}) = Q^n
→ Newton iteration: [I - Δt · ∂R/∂Q] · ΔQ = -(Q^{n+1,k} - Q^n - Δt · R(Q^{n+1,k}))
```
- 1차, L-stable, 무조건 안정
- 정상 상태 수렴에 적합 (시간 정확도 불필요)

**BDF2:**
```
Q^{n+1} = (4/3)Q^n - (1/3)Q^{n-1} + (2/3)Δt · R(Q^{n+1})
```
- 2차, A-stable
- 첫 번째 스텝은 BDF1로 시작 (Q^{n-1}이 없으므로)
- **3개 시간 레벨 필요 (메모리 주의)**

**BDF3 이상:** A-stable이 아님 → CFD에서 거의 사용 안 함

#### SDIRK (Singly Diagonally Implicit Runge-Kutta)

**원리:** Butcher tableau의 A 행렬이 하삼각 + 대각이 같은 값 → 각 단계에서 같은 야코비안 사용 가능.

**SDIRK2 (예: Alexander, 1977):**
```
γ = 1 - √2/2 ≈ 0.2929

  γ   | γ
  1   | 1-γ   γ
  ----+--------
      | 1-γ   γ
```
- 2차, L-stable
- 각 단계: [I - γΔt · ∂R/∂Q] · ΔQ = RHS (야코비안 동일)

**체크리스트 (Implicit):**
- [ ] Newton 반복의 수렴 판정 기준이 명확한가 (상대 잔차 < 1e-6 등)
- [ ] Jacobian ∂R/∂Q 계산이 정확한가 → `/cfd-debug-adv` Step 4
- [ ] BDF2 첫 스텝이 BDF1인가 (self-starting이 아님)
- [ ] SDIRK의 γ 값이 논문과 일치하는가
- [ ] 선형 시스템 솔버가 수렴하는가 → `/cfd-implicit`
- [ ] Δt가 너무 크면 Newton이 발산할 수 있음 (CFL ramp-up 필요)

### Step 4: Dual Time Stepping

**용도:** 비정상 유동의 implicit 시간 전진. 물리 시간(physical time)의 정확도를 유지하면서 내부 반복(pseudo time)으로 수렴.

**구조:**
```
외부 루프 (물리 시간, n → n+1):
  BDF2: (3Q^{n+1} - 4Q^n + Q^{n-1}) / (2Δt) = R(Q^{n+1})
  
  내부 루프 (pseudo time, m → m+1):
    Q^{n+1,m+1} = Q^{n+1,m} + Δτ · R*(Q^{n+1,m})
    R* = R(Q) - (3Q - 4Q^n + Q^{n-1}) / (2Δt)   [수정 잔차]
    
  수렴 판정: |R*| < ε (내부 잔차가 충분히 작음)
```

**체크리스트:**
- [ ] 물리 시간 스텝 Δt와 pseudo 시간 스텝 Δτ를 구분하고 있는가
- [ ] 내부 루프 수렴 판정 기준이 물리 시간 정확도를 보장하기에 충분한가
- [ ] 내부 루프에서 local time stepping 사용 가능 (Δτ는 CFL 무관)
- [ ] 외부 BDF2의 첫 물리 스텝은 BDF1인가

### Step 5: CFL 조건 및 시간 단계 계산

**Explicit 스킴:**
```
Δt = CFL · min_cells( V_cell / Σ_faces (|V_n| + a) · A_face )

또는 1D 근사:
Δt = CFL · min( Δx / (|u| + a), Δy / (|v| + a), Δz / (|w| + a) )
```

**주의:**
- CFL ≤ 1 (일반적), SSP-RK3에서 CFL_SSP = 1
- 점성항이 있으면: Δt_vis = Δx² / (2ν), Δt = min(Δt_conv, Δt_vis)
- 소스항이 있으면: Δt_src = 1 / |∂S/∂Q|_max 도 고려

**Implicit 스킴:**
- 이론적으로 CFL 제한 없음
- 실제: Newton 수렴을 위해 CFL ramp-up (CFL = 1 → 100 → 1000...)
- 정상 상태: 잔차 감소율로 CFL 조정

## 공통 검증 방법

1. **시간 정확도 테스트**: isentropic vortex를 여러 Δt로 실행, 오차 vs Δt 그래프에서 기울기 = 이론 차수
2. **정상 상태 수렴**: 잔차가 machine precision까지 떨어지는가 (BDF1, Newton-Krylov)
3. **에너지 보존**: 비점성 문제에서 총 에너지가 보존되는가 (시간 적분 오차 범위 내)
4. **SSP 성질**: 충격파 문제에서 SSP-RK와 비-SSP RK 비교 (진동 유무)

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| 시간 차수 < 공간 차수 | 중단. 시간 적분 차수를 높이거나 공간 차수를 의도적으로 맞춤. |
| Butcher tableau Σb_i ≠ 1 | 중단. 계수 재확인. |
| Explicit에서 CFL > 1 | 중단. 안정성 위반. |
| BDF2 첫 스텝이 BDF1이 아님 | 중단. 자기 출발(self-start) 처리. |
| Newton이 3~5회 내에 수렴하지 않음 | 중단. Jacobian 정확성 확인 (`/cfd-debug-adv` Step 4). |
| 잔차 부호 규약 불일치 (dQ/dt = R vs dQ/dt = -R) | 중단. 코드 전체에서 부호 통일. |
| 중간 RK 단계에서 BC 미적용 | 중단. 모든 중간 단계에서 BC 업데이트. |

## 관련 스킬

- `/cfd-reconstruction` — 시간 차수와 공간 차수의 균형
- `/cfd-implicit` — implicit 스킴의 선형 시스템 풀이
- `/cfd-vv` — 시간 정확도 검증 (수렴 차수 측정)
- `/cfd-debug-adv` — Jacobian 검증 (Step 4), 발산 시 CFL 진단
