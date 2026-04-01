---
name: cfd-implicit
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-implicit.
  Guide for implementing implicit solvers in CFD: Jacobian derivation,
  sparse matrix assembly, iterative linear solvers, and preconditioners.
  Use when implementing Newton-Krylov methods, implicit time integration,
  or pressure-velocity coupling.
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

# Implicit 솔버 구현 가이드

## 개요

Implicit 솔버는 매 시간 단계마다 선형 시스템 `A·x = b`를 풀어야 한다. A는 Jacobian에서 유도되고, 희소(sparse) 행렬이며, 시스템의 크기는 `(N_cells × N_vars)`이다. 이 과정의 구현 품질이 implicit 솔버의 수렴성과 효율을 결정한다.

**핵심 원칙:** Jacobian이 틀리면 Newton이 수렴하지 않고, 선형 솔버가 수렴하지 않으면 비선형 솔버도 실패한다. 두 레벨 모두 독립적으로 검증해야 한다.

## 사전 조건

- 잔차 함수 R(Q)가 정확하게 구현되어 있어야 한다 (explicit 솔버에서 검증 완료)
- 격자의 연결 정보(connectivity)에 접근 가능해야 한다
- 목표: Newton-Krylov, defect correction, 또는 pressure-velocity coupling 중 어느 것인지 결정

## 프로토콜

### Step 1: Jacobian 전략 선택

```
Jacobian을 어떻게 계산할 것인가?
├── A. 해석적 Jacobian (hand-derived)
│   - 가장 빠름, 가장 정확
│   - 구현 복잡, 유지보수 어려움
│   - flux 함수 변경 시 Jacobian도 재유도 필요
│
├── B. 수치적 Jacobian (finite difference)
│   - 구현 간단: J_{ij} ≈ (R(Q + ε·e_j) - R(Q)) / ε
│   - 느림 (N_vars 배의 잔차 평가)
│   - ε 선택이 민감 (너무 크면 부정확, 너무 작으면 반올림 오류)
│
├── C. 자동 미분 (AD, algorithmic differentiation)
│   - 정확하고 자동적
│   - forward mode: 열 단위, reverse mode: 행 단위
│   - 도구 의존적 (Tapenade, CoDiPack, JAX 등)
│
└── D. 근사 Jacobian (approximate)
    - 1차 upwind flux의 Jacobian만 사용 (고차 항 무시)
    - 소스항 Jacobian만 해석적, flux는 근사
    - Newton 수렴률 저하, 하지만 구현 용이
```

**권장:**
- 개발 초기: **B (수치적)** → 정확한 기준선 확보
- 성능 필요: **A (해석적)** 또는 **C (AD)**
- 빠른 프로토타입: **D (근사)** + 더 많은 Newton 반복

### Step 2: 해석적 Jacobian 유도

해석적 Jacobian을 선택한 경우:

**유도 순서:**
```
1. 잔차를 구성 요소로 분해:
   R(Q)_i = (1/V_i) · Σ_faces F_num(Q_L, Q_R, n) · A_f + S(Q_i) · V_i

2. 셀 i에 대한 Jacobian 블록:
   ∂R_i/∂Q_i  = (1/V_i) · Σ_faces (∂F/∂Q_L · ∂Q_L/∂Q_i) · A_f + ∂S/∂Q_i
   ∂R_i/∂Q_j  = (1/V_i) · (∂F/∂Q_R · ∂Q_R/∂Q_j) · A_f   [j = 이웃 셀]

3. 재구성이 있으면:
   ∂Q_L/∂Q_i ≠ I  (limiter, MUSCL, WENO의 기여)
   → 1차 근사: ∂Q_L/∂Q_i ≈ I (재구성 Jacobian 무시) — 흔한 근사
```

**Flux Jacobian ∂F/∂Q_L, ∂F/∂Q_R:**
- Roe flux: 해석적 유도 가능 (고유값/고유벡터 미분 포함)
- AUSM: M⁺, M⁻, P⁺, P⁻의 미분이 필요
- HLLC: 파동 속도 S_L, S_*, S_R의 미분 필요
- **근사: 1차 upwind flux의 Jacobian 사용** (가장 흔한 실무 선택)

### Step 3: 희소 행렬 조립

**행렬 구조:**

CFD의 Jacobian은 블록 희소 행렬이다:
```
각 셀 i에 대해:
  대각 블록: A_ii = I/(Δt) - ∂R_i/∂Q_i     [N_vars × N_vars]
  비대각 블록: A_ij = -∂R_i/∂Q_j            [이웃 셀 j에 대해]
```

**저장 형식:**

| 형식 | 특성 | 적합한 경우 |
|------|------|------------|
| CSR (Compressed Sparse Row) | 범용, 라이브러리 호환 | 비구조격자, PETSc/Trilinos 사용 시 |
| BSR (Block Sparse Row) | 블록 단위 저장, BLAS 활용 | N_vars가 큰 경우 (5-변수 Euler 등) |
| 밴드 행렬 | 대역폭 고정 | 구조격자, 1D/2D |
| 행렬-free (matrix-free) | 행렬 저장 안 함, J·v만 계산 | 메모리 제한, GMRES와 결합 |

**조립 절차:**
```
1. 비영(non-zero) 패턴 결정: 격자 연결성에서 유도
   - 구조격자: 대역폭 = (2·N_dims + 1) · N_vars
   - 비구조격자: 셀별 이웃 수가 다름 → CSR 필수
   
2. 패턴을 한 번만 생성 (격자가 안 변하면)

3. 매 Newton 반복에서 값만 업데이트
```

**체크리스트:**
- [ ] 행렬의 대각 블록이 I/(Δt) 항을 포함하는가
- [ ] 비대각 블록의 부호가 정확한가 (A_ij = -∂R_i/∂Q_j)
- [ ] 블록 크기가 N_vars와 일치하는가
- [ ] 경계 셀의 행렬 행이 BC를 반영하는가

### Step 3.5: Matrix-Free Newton-Krylov

행렬을 조립하지 않고 Jacobian-vector product `J·v`만 계산하는 방법. 메모리 절약이 크고, Jacobian 유도가 불필요.

**핵심 아이디어:**
```
J·v ≈ (R(Q + ε·v) - R(Q)) / ε    [forward difference]
    ≈ (R(Q + ε·v) - R(Q - ε·v)) / (2ε)  [central, 더 정확]

ε = √(machine_eps) · ‖Q‖ / ‖v‖   [일반적 선택]
  또는 ε = √(machine_eps) · max(1, ‖Q‖) / ‖v‖
```

**장점:**
- Jacobian 유도/코딩 불필요 — 잔차 함수 R(Q)만 있으면 됨
- 고차 flux, 복잡한 소스항에서도 자동으로 정확
- 메모리: O(N) vs 행렬 조립 O(N · nnz)

**단점:**
- 매 Krylov 반복마다 잔차 평가 1회 추가 (비용)
- **전처리기가 여전히 행렬 필요** — 근사 Jacobian(1차 upwind)으로 전처리기만 조립
- ε 선택이 민감 (너무 크면 부정확, 너무 작면 반올림 오류)

**실무 패턴:**
```
1. 근사 Jacobian으로 ILU 전처리기 조립 (1차 upwind, 저비용)
2. GMRES의 J·v는 matrix-free로 계산 (정확한 고차 flux 반영)
3. Newton 수렴: 2차 수렴 달성 (정확한 J·v 덕분)
```

**체크리스트:**
- [ ] ε 값이 ‖Q‖와 ‖v‖에 비례하여 스케일링되는가
- [ ] ε를 10배 변경해도 결과가 변하지 않는가 (변하면 구현 오류)
- [ ] 전처리기용 근사 Jacobian이 별도로 조립되는가
- [ ] R(Q + ε·v) 계산 시 경계조건도 올바르게 적용되는가

### Step 4: 반복 선형 솔버

**직접 풀이(direct)는 3D CFD에서 비현실적** → 반복 솔버 사용.

| 솔버 | 대칭 필요 | 특성 | 권장 |
|------|-----------|------|------|
| GMRES | 아니오 | 가장 범용, 재시작 필요 (메모리) | **기본 선택** |
| BiCGSTAB | 아니오 | 메모리 효율, 불규칙 수렴 | GMRES 메모리 부족 시 |
| CG | 대칭 양정치 | 가장 효율적 (조건 맞으면) | 압력 Poisson만 |
| Flexible GMRES | 아니오 | 가변 전처리기 허용 | multigrid 전처리 시 |

**GMRES 파라미터:**
- restart: 30~100 (메모리 vs 수렴 트레이드오프)
- tolerance: 상대 잔차 1e-2 ~ 1e-4 (inexact Newton 기법)
- max iterations: 100~500

**Inexact Newton-Krylov:**
```
초기: η = 0.1 (선형 솔버 느슨하게)
Newton 수렴에 따라: η → 1e-4 (선형 솔버 정밀하게)

Eisenstat-Walker 선택:
  η_{k+1} = |‖R_{k+1}‖ - ‖R_k + J_k·Δx_k‖| / ‖R_k‖
```

### Step 5: 전처리기 (Preconditioner)

**전처리가 없으면 반복 솔버가 수렴하지 않거나 매우 느리다.**

| 전처리기 | 복잡도 | 효과 | 적합한 경우 |
|----------|--------|------|------------|
| Block Jacobi | 낮음 | 약함 | 첫 구현, 디버깅 용 |
| ILU(0) | 중간 | 양호 | **기본 선택** |
| ILU(k) | 중~높음 | 우수 | ILU(0) 부족 시 |
| Block ILU | 중간 | 양호 | 블록 희소 행렬에 자연스러움 |
| Additive Schwarz | 높음 | 우수 | 병렬 환경 |
| Algebraic Multigrid (AMG) | 높음 | 최우수 | 타원형 문제 (압력 방정식) |
| LU-SGS | 중간 | 양호 | 간단한 implicit, sweep 기반 |

**LU-SGS (Lower-Upper Symmetric Gauss-Seidel):**
```
Forward sweep:  D·ΔQ* = -R - L·ΔQ*
Backward sweep: D·ΔQ  = D·ΔQ* - U·ΔQ

D = 대각 블록, L = 하삼각, U = 상삼각
```
- 행렬 조립 없이 sweep만으로 근사 역행렬
- CFD에서 매우 흔한 선택 (특히 정상 상태)
- 셀 순회 순서가 수렴에 영향 (비구조격자 주의)

### Step 6: Newton 반복 구조

**전체 구조:**
```
While (‖R(Q)‖ > tol):
  1. 잔차 계산: R(Q^k)
  2. Jacobian 계산/업데이트: J = ∂R/∂Q|_{Q^k}  (또는 matrix-free)
  3. 전처리기 업데이트 (필요 시)
  4. 선형 시스템 풀이: J · ΔQ = -R(Q^k)   [GMRES + preconditioner]
  5. 업데이트: Q^{k+1} = Q^k + α · ΔQ
  6. (선택) Line search: α를 조정하여 ‖R(Q^{k+1})‖ < ‖R(Q^k)‖ 보장
  7. 수렴 확인
```

**Newton 수렴 판정:**
- 상대 잔차: ‖R(Q^k)‖ / ‖R(Q^0)‖ < 1e-6
- 절대 잔차: ‖R(Q^k)‖ < 1e-12
- 최대 반복: 10~20 (초과 시 Δt 줄이기)
- **2차 수렴 관찰**: ‖R^{k+1}‖ ≈ C · ‖R^k‖² (정확한 Jacobian의 증거)

## 체크리스트

### Jacobian
- [ ] 수치 Jacobian과 비교하여 검증했는가 (`/cfd-debug-adv` Step 4)
- [ ] flux Jacobian과 소스항 Jacobian을 분리 검증했는가
- [ ] 경계 조건의 Jacobian도 검증했는가
- [ ] Δt → ∞ 에서 대각 우세가 사라짐을 인지하고 있는가

### 선형 솔버
- [ ] 전처리기 없이 GMRES가 수렴하는가 (느려도 수렴해야)
- [ ] 전처리기 추가 후 반복 수가 유의미하게 줄었는가
- [ ] restart 값이 적절한가 (너무 작으면 수렴 실패)
- [ ] 선형 잔차가 Newton 수렴을 저해하지 않는가

### Newton 반복
- [ ] 2차 수렴이 관찰되는가 (해 근처에서)
- [ ] 3~5회 Newton 반복 내에 수렴하는가
- [ ] 수렴 실패 시 Δt를 줄이면 수렴하는가 → Jacobian은 맞고 Δt가 너무 큰 것
- [ ] Δt를 줄여도 수렴하지 않으면 → Jacobian 오류 → 검증

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| Newton이 2차 수렴하지 않음 | 중단. Jacobian 검증 (`/cfd-debug-adv` Step 4). |
| GMRES가 restart마다 잔차 정체 | 중단. restart 증가 또는 전처리기 강화. |
| ILU 분해 중 zero pivot | 중단. 행렬 대각 항 확인 (Δt 항 누락 가능). |
| 선형 솔버 수렴, Newton 발산 | 중단. 선형 tolerance가 너무 느슨하거나 line search 필요. |
| LU-SGS의 셀 순회 순서 변경 시 결과 변화 | 주의. 비구조격자에서 정상. 수렴 결과는 동일해야 함. |
| 행렬-free에서 ε 변경 시 결과 변화 | 중단. ε = √(machine_eps) · ‖Q‖ 사용. |

## 관련 스킬

- `/cfd-time-integration` — implicit 시간 적분 스킴 (BDF, SDIRK)
- `/cfd-debug-adv` — Jacobian 검증 (Step 4), Newton 발산 진단
- `/cfd-flux` — flux Jacobian 유도 시 참조
- `/cfd-euler` — Euler 방정식의 Jacobian 구조
- `/cfd-ns` — Navier-Stokes 점성항 Jacobian
