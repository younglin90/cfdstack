---
name: cfd-ns
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-ns.
  Guide for implementing Navier-Stokes viscous terms in CFD solvers.
  Covers: viscous flux discretization, heat conduction, compressible vs incompressible
  formulations, pressure-velocity coupling, and viscous term Jacobian.
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

# Navier-Stokes 점성항 가이드

## 개요

Navier-Stokes 방정식은 Euler 방정식에 점성 응력과 열전도를 추가한 것이다. 점성항은 2차 미분(확산)이므로 이산화 방법이 대류항과 본질적으로 다르다.

**핵심 원칙:** 점성항은 중심 차분(central)으로, 대류항은 upwind로. 두 항의 이산화를 혼동하지 않는다.

## 사전 조건

- Euler 솔버가 검증 완료 → `/cfd-euler`
- 층류(laminar) vs 난류(turbulent) 결정
- 압축성 vs 비압축성 결정

## 방정식 정리

### 압축성 Navier-Stokes

```
∂Q/∂t + ∂(F_inv - F_vis)/∂x + ∂(G_inv - G_vis)/∂y + ∂(H_inv - H_vis)/∂z = 0
```

**점성 flux (x-방향):**
```
F_vis = [0,
         τ_xx,
         τ_xy,
         τ_xz,
         u·τ_xx + v·τ_xy + w·τ_xz + k·∂T/∂x]
```

**응력 텐서 (Newtonian 유체):**
```
τ_xx = μ · (2·∂u/∂x - (2/3)·∇·v)
τ_yy = μ · (2·∂v/∂y - (2/3)·∇·v)
τ_zz = μ · (2·∂w/∂z - (2/3)·∇·v)
τ_xy = τ_yx = μ · (∂u/∂y + ∂v/∂x)
τ_xz = τ_zx = μ · (∂u/∂z + ∂w/∂x)
τ_yz = τ_zy = μ · (∂v/∂z + ∂w/∂y)

∇·v = ∂u/∂x + ∂v/∂y + ∂w/∂z  (체적 팽창률)
```

**Stokes 가설:** 체적 점성 λ = -2μ/3 (논란 있지만 표준적)

**열전도:**
```
q_x = -k · ∂T/∂x    (Fourier 법칙)
k = μ · c_p / Pr      (Prandtl 수로부터)
```

### 비압축성 Navier-Stokes

```
∇·u = 0                                    (연속 방정식)
∂u/∂t + (u·∇)u = -∇p/ρ + ν·∇²u + f      (운동량)
```

**핵심 차이:**
| 항목 | 압축성 | 비압축성 |
|------|--------|----------|
| 상태 방정식 | p = f(ρ, e) | p는 Lagrange 승수 (EOS 없음) |
| 연속 방정식 | ∂ρ/∂t + ∇·(ρu) = 0 | ∇·u = 0 (발산 없음 제약) |
| 에너지 방정식 | 연립 (음속으로 결합) | 선택적 (Boussinesq 등) |
| 압력 결정 | 상태 방정식 | Poisson 방정식 풀이 |
| 시간 적분 | 명시적/암시적 | pressure-velocity coupling 필수 |

## 프로토콜

### Step 1: 점성 flux 이산화

**gradient 계산이 핵심이다.** 점성 flux에는 속도와 온도의 gradient가 필요하다.

**구조격자 — 중심 차분:**
```
(∂u/∂x)_{i+1/2} = (u_{i+1} - u_i) / Δx

교차 미분항:
(∂u/∂y)_{i+1/2} = 0.5 · ((∂u/∂y)_i + (∂u/∂y)_{i+1})
                 = 0.5 · ((u_{i,j+1} - u_{i,j-1})/(2Δy) + (u_{i+1,j+1} - u_{i+1,j-1})/(2Δy))
```

**비구조격자 — face gradient:**
```
방법 1: Green-Gauss gradient를 face 중심으로 보간
  (∇u)_f = 0.5 · ((∇u)_owner + (∇u)_neighbour)

방법 2: face 법선 방향 gradient (직접 계산)
  (∂u/∂n)_f = (u_N - u_O) / |d_ON|

방법 3: 보정된 gradient (non-orthogonal correction)
  (∇u)_f · n̂ = (u_N - u_O) / |d_ON| + 보정항
```

**비직교 보정 (non-orthogonal correction):**
비구조격자에서 셀 중심 연결선이 면 법선과 평행하지 않으면 보정 필요.
```
면 법선 n̂과 셀 중심 연결 벡터 d̂_ON의 각도 > 0 → 보정
보정항 = (∇u)_f · (n̂ - d̂·(n̂·d̂)/|d̂|²) · [추가 보정]
```

### Step 2: 점성 계수

**층류:**
```
Sutherland 법칙: μ(T) = μ_ref · (T/T_ref)^(3/2) · (T_ref + S) / (T + S)
  공기: μ_ref = 1.716e-5, T_ref = 273.15, S = 110.4

또는 power law: μ(T) = μ_ref · (T/T_ref)^ω
  공기: ω ≈ 0.67
```

**난류:** → `/cfd-turbulence`
```
μ_eff = μ_lam + μ_turb
k_eff = k_lam + k_turb = μ_lam·c_p/Pr_lam + μ_turb·c_p/Pr_turb
```

### Step 3: 압축성 점성 flux 조립

**면에서의 점성 flux:**
```
F_vis · n = [0,
             τ_xx·n_x + τ_xy·n_y + τ_xz·n_z,
             τ_xy·n_x + τ_yy·n_y + τ_yz·n_z,
             τ_xz·n_x + τ_yz·n_y + τ_zz·n_z,
             (u·τ_xx + v·τ_xy + w·τ_xz)·n_x + ... + k·(∂T/∂n)]
```

**에너지 방정식 점성항 주의:**
- 일 항(work term): u_j · τ_{ij} — 속도와 응력의 곱
- 열전도 항: k · ∂T/∂n
- **두 항 모두 빠뜨리지 않는다** (일 항을 잊는 경우가 많음)

### Step 4: 비압축성 — Pressure-Velocity Coupling

비압축성에서는 압력이 상태 방정식이 아닌 연속 방정식(∇·u=0)에서 결정된다.

**주요 알고리즘:**

| 알고리즘 | 유형 | 특성 |
|----------|------|------|
| SIMPLE | 정상/비정상 | 압력-속도 순차 보정. under-relaxation 필요 |
| SIMPLEC | 정상 | SIMPLE 개선. 압력 보정 계수 수정 |
| PISO | 비정상 | 2회 보정. non-iterative (비정상에 적합) |
| Projection | 비정상 | Chorin's fractional step. 간단명확 |
| Coupled | 정상/비정상 | 압력-속도 동시 풀이. 수렴 빠름, 메모리 큼 |

**Projection Method (Chorin, 1968):**
```
1. 예측: u* = u^n + Δt · (-( u·∇)u + ν·∇²u + f)   [압력항 무시]
2. 압력 Poisson: ∇²p = (1/Δt) · ∇·u*
3. 보정: u^{n+1} = u* - Δt · ∇p
4. 확인: ∇·u^{n+1} ≈ 0 (machine precision)
```

**SIMPLE:**
```
1. 운동량 예측: A·u* = H - ∇p'     [p' = 이전 반복의 압력]
2. 압력 보정: ∇·((1/A)·∇p') = ∇·u*
3. 속도 보정: u = u* - (1/A)·∇p'
4. 압력 업데이트: p = p' + α_p · p'_correction
5. 수렴까지 반복
```

### Step 5: 벽면 경계조건

**No-slip 벽면:**
```
u_wall = 0 (또는 벽 속도)
∂p/∂n = 0 (일반적)
T_wall = T_specified (등온) 또는 ∂T/∂n = q_specified (등열유속)
```

**벽면 전단 응력:**
```
τ_wall = μ · (∂u/∂n)|_wall

1차 셀의 근사:
τ_wall ≈ μ · u_1 / y_1   (y_1 = 첫 셀 중심과 벽면 거리)
```

**열전달:**
```
q_wall = -k · (∂T/∂n)|_wall ≈ k · (T_1 - T_wall) / y_1
Nu = q_wall · L / (k · ΔT)
```

## 체크리스트

### 점성항 이산화
- [ ] 응력 텐서의 체적 팽창률 항 -(2/3)μ∇·v가 포함되었는가
- [ ] 교차 미분항 (∂u/∂y, ∂v/∂x 등)이 정확한가
- [ ] 에너지 방정식의 일 항(u·τ)이 포함되었는가
- [ ] 비구조격자: 비직교 보정이 적용되었는가
- [ ] 점성 flux가 중심 차분(central)으로 이산화되었는가 (upwind 아님)

### 비압축성
- [ ] ∇·u = 0이 machine precision 수준으로 만족되는가
- [ ] 압력 Poisson의 경계조건이 정확한가 (Neumann: ∂p/∂n = 0 at walls)
- [ ] SIMPLE의 under-relaxation 계수가 적절한가 (α_u ≈ 0.7, α_p ≈ 0.3)
- [ ] 정상 상태에서 잔차가 충분히 작은가

### 검증
- [ ] Couette flow: 선형 속도 프로파일 → `/cfd-vv/benchmarks.md` 4.1
- [ ] Poiseuille flow: 포물선 속도 프로파일 → `/cfd-vv/benchmarks.md` 4.2
- [ ] 점성항만의 수렴 차수가 이론과 일치하는가 (대류항 없이)

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| 점성 flux를 upwind로 이산화 | 중단. 중심 차분으로. |
| τ에서 (2/3)μ∇·v 항 누락 | 중단. 2D에서도 필요 (∂w/∂z = 0이어도 ∂u/∂x + ∂v/∂y ≠ 0). |
| 에너지 방정식에서 일 항 누락 | 중단. 고마하에서 온도 오류 발생. |
| 비압축성에서 ∇·u ≠ 0 | 중단. 압력 Poisson 또는 projection 확인. |
| 점성 응력 부호 혼동 (τ vs -τ) | 중단. 논문의 부호 규약과 코드 일치 확인. |
| 비직교 격자에서 보정 없이 2차 정확도 주장 | 중단. 비직교 보정 필요. |

## 관련 스킬

- `/cfd-euler` — 비점성 flux (대류항)
- `/cfd-flux` — 대류 flux 함수
- `/cfd-turbulence` — 난류 모델 (μ_turb 계산)
- `/cfd-implicit` — 점성항 Jacobian, 압력 Poisson 솔버
- `/cfd-vv` — Couette, Poiseuille, lid-driven cavity 벤치마크
