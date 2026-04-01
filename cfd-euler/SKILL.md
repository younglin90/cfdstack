---
name: cfd-euler
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-euler.
  Guide for implementing Euler equation solvers for compressible inviscid flow.
  Covers: conservation law formulation, characteristic analysis, equation of state,
  shock capturing, entropy conditions, and common implementation pitfalls.
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

# Euler 방정식 솔버 가이드

## 개요

Euler 방정식은 비점성 압축성 유동의 지배 방정식이다. 쌍곡형(hyperbolic) 보존법칙으로, 불연속해(충격파, 접촉면)를 포함할 수 있다.

**핵심 원칙:** Euler 솔버는 (1) 보존형으로 구현해야 하고, (2) 엔트로피 조건을 만족해야 하며, (3) 양보존(positivity) — ρ > 0, p > 0 — 을 유지해야 한다.

## 사전 조건

- 보존형 vs 비보존형의 차이를 이해하는가
- 사용할 flux 함수를 선택했는가 → `/cfd-flux`
- 사용할 재구성 스킴을 선택했는가 → `/cfd-reconstruction`

## 방정식 정리

### 보존형 (Conservative Form)

**3D:**
```
∂Q/∂t + ∂F/∂x + ∂G/∂y + ∂H/∂z = 0

Q = [ρ, ρu, ρv, ρw, ρE]^T

F = [ρu, ρu²+p, ρuv, ρuw, (ρE+p)u]^T
G = [ρv, ρuv, ρv²+p, ρvw, (ρE+p)v]^T
H = [ρw, ρuw, ρvw, ρw²+p, (ρE+p)w]^T
```

**닫힘 조건 (이상 기체):**
```
p = (γ-1) · ρ · (E - 0.5·(u²+v²+w²))
  = (γ-1) · (ρE - 0.5·(ρu²+ρv²+ρw²)/ρ)

a = √(γp/ρ)           (음속)
H = (ρE + p) / ρ       (총 엔탈피)
  = E + p/ρ
T = p / (ρ·R_gas)      (온도, 필요 시)
```

### 법선 flux (면 기준)

면 법선 n = (n_x, n_y, n_z)에 대한 법선 flux:
```
V_n = u·n_x + v·n_y + w·n_z  (법선 속도)

F_n = [ρ·V_n,
       ρu·V_n + p·n_x,
       ρv·V_n + p·n_y,
       ρw·V_n + p·n_z,
       (ρE+p)·V_n]
```

### 특성 분석 (Characteristic Analysis)

법선 방향의 Jacobian A_n = ∂F_n/∂Q의 고유값:
```
λ_1 = V_n - a    (좌향 음파)
λ_2 = V_n        (엔트로피파, 중복도 = 1)
λ_3 = V_n        (전단파, 중복도 = d-1, d=차원)
λ_4 = V_n + a    (우향 음파)
```

**파동의 물리적 의미:**
| 파동 | 고유값 | 전파 물리량 | 불연속 유형 |
|------|--------|-------------|-------------|
| 음파 (acoustic) | V_n ± a | 압력, 밀도, 속도 | 충격파 / 팽창파 |
| 엔트로피 (entropy) | V_n | 밀도 (압력/속도 불변) | 접촉면 |
| 전단 (shear) | V_n | 접선 속도 | 전단층 |

## 프로토콜

### Step 1: 상태 방정식 (EOS) 구현

가장 먼저, 독립적으로 구현하고 테스트한다.

**이상 기체:**
```
p = (γ-1) · ρ · e_internal
e_internal = E - 0.5·|v|²
a = √(γ·p/ρ)
```

**실제 기체 (필요 시):**
- Stiffened gas: p = (γ-1)·ρ·e - γ·p_∞
- Van der Waals
- 테이블 기반 EOS

**검증:** 알려진 (ρ, u, p) 조합에서 E, a, H 계산 → 손계산과 비교

### Step 2: 보존 변수 ↔ 원시 변수 변환

```
보존 → 원시:
  ρ = Q[0]
  u = Q[1] / ρ
  v = Q[2] / ρ
  w = Q[3] / ρ
  E = Q[4] / ρ
  p = (γ-1) · (Q[4] - 0.5·(Q[1]²+Q[2]²+Q[3]²)/Q[0])

원시 → 보존:
  Q[0] = ρ
  Q[1] = ρ·u
  Q[2] = ρ·v
  Q[3] = ρ·w
  Q[4] = p/(γ-1) + 0.5·ρ·(u²+v²+w²)
```

**주의:**
- 보존→원시 변환 시 ρ = 0이면 나눗셈 불가 → 보호 필요
- p 계산: 큰 수의 상쇄(ρE - 0.5·ρ|v|²)로 정밀도 손실 가능 (고마하에서)
- **이 변환 함수가 정확하지 않으면 모든 것이 틀림** → 단위 테스트 필수

### Step 3: Flux 계산

`/cfd-flux`를 따라 구현. Euler 방정식 특화 주의사항:

**flux 함수에 넘길 물리량:**
```
좌측: ρ_L, u_L, v_L, w_L, p_L, H_L, a_L
우측: ρ_R, u_R, v_R, w_R, p_R, H_R, a_R
면 법선: n_x, n_y, n_z (+ 면적 or 단위벡터)
```

**H 계산 주의:** H = (ρE + p) / ρ = E + p/ρ. Q[4]가 ρE인지 E인지 확인.

### Step 4: 엔트로피 조건

**물리적 의미:** 충격파를 통과할 때 엔트로피는 증가해야 한다 (2법칙). 수치 스킴이 이를 위반하면 비물리적 팽창 충격파가 발생.

**보장 방법:**
1. Godunov-type flux: 정확한 Riemann solver는 자동 만족
2. Roe flux: **반드시 entropy fix 필요** → `/cfd-flux` 3A
3. AUSM: 구조적으로 만족 (upwind)
4. Lax-Friedrichs: 과도한 확산이 자동 보장

**검증:** 123 문제 (Einfeldt et al.) — 두 팽창파가 만나는 케이스
```
좌측: ρ=1, u=-2, p=0.4
우측: ρ=1, u=2,  p=0.4
```
entropy fix 없는 Roe는 중앙에서 음의 밀도 생성.

### Step 5: 양보존 (Positivity Preservation)

**ρ > 0, p > 0을 유지해야 한다.** 위반 시 음속 √(γp/ρ)에서 NaN.

**방법:**
1. **사후 보정 (floor):** ρ = max(ρ, ρ_min), p = max(p, p_min) — 간단하지만 보존 위반
2. **양보존 limiter:** Zhang & Shu (2010) — 고차 스킴에서도 양보존
3. **Flux 선택:** HLL/HLLC는 Roe보다 양보존에 유리
4. **CFL 제한:** 양보존을 위한 CFL 조건이 일반 CFL보다 엄격할 수 있음

### Step 6: 소스항 (필요 시)

| 소스항 | 수식 | 주의 |
|--------|------|------|
| 중력 | S = [0, ρg_x, ρg_y, ρg_z, ρ(u·g_x+v·g_y+w·g_z)] | well-balanced 스킴 필요 (정수압 평형 유지) |
| 축대칭 (원통좌표) | S = -[ρv_r, ρv_r², ρv_rv_θ, ...]/r | 1/r 특이점 주의 (r=0) |
| 회전 프레임 | 코리올리 + 원심력 | 관성 프레임과 회전 프레임 혼동 주의 |

## 체크리스트

### EOS + 변수 변환
- [ ] 이상 기체 상수 γ가 정확한가 (공기: 1.4, 단원자: 5/3)
- [ ] 보존→원시 변환이 단위 테스트를 통과하는가
- [ ] 원시→보존→원시 왕복(round-trip)에서 machine precision 일치하는가
- [ ] H = E + p/ρ = (ρE + p)/ρ 두 경로가 일치하는가

### 보존법칙
- [ ] **보존형**으로 구현했는가 (비보존형은 충격파 위치가 틀림)
- [ ] 전체 보존량 Σ(Q·V_cell)이 시간에 따라 보존되는가 (닫힌 계)
- [ ] flux가 면을 공유하는 두 셀에서 정확히 상쇄되는가 (telescoping)

### 충격파 처리
- [ ] 엔트로피 조건이 만족되는가 (Einfeldt 123 문제)
- [ ] ρ > 0, p > 0이 유지되는가
- [ ] Sod shock tube 결과가 해석해와 일치하는가

### 저마하 (M << 1)
- [ ] 저마하에서 압력 진동 (checkerboard) 없는가
- [ ] AUSM+-up 또는 preconditioned flux 사용 (필요 시)
- [ ] 수치 확산이 O(1/M)으로 발산하지 않는가 (아래 참조)

### 저마하 문제 상세 (Low-Mach Issues)

**문제:** 밀도 기반(density-based) Euler 솔버는 M → 0에서 3가지 문제를 겪는다:

1. **과도한 수치 확산**: Roe/HLLC의 확산항이 O(a) = O(1/M·|u|)로 증가하여 해를 평탄화
2. **압력 checkerboard**: 운동량 방정식의 압력 항이 비물리적 진동
3. **stiffness**: 음파 속도 a >> 유동 속도 u → explicit CFL이 극도로 작아짐

**해결 방법:**

| 방법 | 원리 | 적용 대상 |
|------|------|-----------|
| Low-Mach preconditioning | 고유값 스케일링으로 조건수 개선 | Roe, HLLC 등 기존 flux |
| AUSM+-up / SLAU2 | flux 내에 저마하 보정 내장 | AUSM 계열 flux |
| Pressure-based solver | ∇·u=0 직접 풀이 | 비압축→약압축 영역 |

**Preconditioning 기본 형태 (Weiss-Smith / Turkel):**
```
P⁻¹ · ∂Q/∂t + ∂F/∂x = 0

P⁻¹ = preconditioning 행렬: 음파 고유값을 a → max(|u|, ε·a)로 축소
효과: 조건수 a/|u| = O(1/M) → O(1)
```

**주의:** Preconditioning은 정상 상태에서만 적용. 비정상(time-accurate)이면 dual time stepping과 결합하여 물리 시간 항에는 preconditioning을 적용하지 않는다.

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| 비보존형으로 구현 | 중단. 보존형으로 재구현 (충격파 위치가 틀림). |
| √(γp/ρ)에서 NaN | 중단. p < 0 또는 ρ < 0 발생 → 양보존 확인. |
| 보존량이 시간에 따라 변화 (닫힌 계) | 중단. flux 보존성 확인 (면 공유 상쇄). |
| Einfeldt 123에서 음의 밀도 | 중단. entropy fix 또는 HLL로 전환. |
| 고마하에서 p 계산 정밀도 손실 | 주의. double precision 확인, 또는 p를 별도 방정식으로. |
| M < 0.1에서 Roe flux가 해를 과도하게 평탄화 | 중단. preconditioning 또는 AUSM+-up으로 전환. |

## 관련 스킬

- `/cfd-flux` — Roe, AUSM, HLLC 등 flux 함수 구현
- `/cfd-reconstruction` — 고차 재구성
- `/cfd-time-integration` — 시간 적분
- `/cfd-vv` — Sod, isentropic vortex, double Mach reflection 벤치마크
- `/cfd-ns` — 점성항 추가 시
- `/cfd-paper-to-code` — Euler flux의 수식 전사
