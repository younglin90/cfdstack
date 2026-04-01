---
name: cfd-turbulence
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-turbulence.
  Guide for implementing RANS turbulence models in CFD solvers.
  Covers: k-epsilon, k-omega SST, Spalart-Allmaras, wall functions,
  y+ requirements, turbulence boundary conditions, and common pitfalls.
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

# 난류 모델 구현 가이드

## 개요

RANS 난류 모델은 Reynolds 평균 방정식을 닫기 위해 난류 점성 μ_t를 계산한다. 모델 방정식은 stiff 소스항을 포함하므로 수치적 안정성에 특별한 주의가 필요하다.

**핵심 원칙:** (1) 난류 변수의 양보존(k > 0, ε > 0, ω > 0), (2) 벽 근처 처리(y+ 범위), (3) 소스항의 implicit 처리가 3대 핵심.

## 사전 조건

- NS 솔버가 층류에서 검증 완료 → `/cfd-ns`
- 벽면 근처 격자 해상도(y+) 결정
- 정상/비정상 결정

## 모델별 구현 가이드

### 모델 A: Spalart-Allmaras (SA)

**방정식:** 1개 수송 방정식 (수정된 난류 점성 ν̃)

```
∂ν̃/∂t + u·∇ν̃ = c_b1·S̃·ν̃                          [생성]
                - c_w1·f_w·(ν̃/d)²                     [소멸]
                + (1/σ)·∇·((ν+ν̃)·∇ν̃) + c_b2·(∇ν̃)²  [확산]

μ_t = ρ·ν̃·f_v1
f_v1 = χ³/(χ³+c_v1³),  χ = ν̃/ν
```

**모델 상수:**
```
c_b1 = 0.1355, σ = 2/3, c_b2 = 0.622
c_v1 = 7.1, c_w1 = c_b1/κ² + (1+c_b2)/σ
c_w2 = 0.3, c_w3 = 2.0, κ = 0.41
```

**체크리스트:**
- [ ] f_v1, f_v2, f_w 함수가 정확한가
- [ ] S̃ (수정된 와도 크기)가 음수가 되지 않도록 보호하는가
- [ ] d (벽면 거리) 계산이 정확한가
- [ ] 확산항의 c_b2·(∇ν̃)² 항을 빠뜨리지 않았는가 — **자주 빠뜨림**
- [ ] ν̃ > 0 보호가 있는가

---

### 모델 B: k-ε (Standard, Realizable, RNG)

**Standard k-ε (Launder & Spalding, 1974):**

```
∂(ρk)/∂t + ∇·(ρu·k) = ∇·((μ + μ_t/σ_k)·∇k) + P_k - ρε
∂(ρε)/∂t + ∇·(ρu·ε) = ∇·((μ + μ_t/σ_ε)·∇ε) + C_ε1·(ε/k)·P_k - C_ε2·ρ·ε²/k

μ_t = ρ · C_μ · k² / ε
P_k = τ_ij · ∂u_i/∂x_j   (생성항 = Reynolds 응력 × 변형률)
```

**모델 상수:**
```
C_μ = 0.09, C_ε1 = 1.44, C_ε2 = 1.92
σ_k = 1.0, σ_ε = 1.3
```

**체크리스트:**
- [ ] P_k 계산이 정확한가: P_k = μ_t·S² (S = |S_ij|, 변형률 텐서 크기)
- [ ] ε 방정식의 ε/k 항에서 k → 0 보호가 있는가
- [ ] k, ε > 0 보호가 있는가 (floor 또는 implicit 소스 처리)
- [ ] 벽면 근처: **wall function 필수** (low-Re 모델이 아니면)

---

### 모델 C: k-ω SST (Menter, 1994)

**가장 많이 사용되는 모델.** 벽 근처에서 k-ω, 외부에서 k-ε의 장점을 결합.

```
∂(ρk)/∂t + ∇·(ρu·k) = ∇·((μ + σ_k·μ_t)·∇k) + P̃_k - β*·ρ·ω·k
∂(ρω)/∂t + ∇·(ρu·ω) = ∇·((μ + σ_ω·μ_t)·∇ω) + α·S² - β·ρ·ω² + 2(1-F1)·ρ·σ_ω2/(ω)·∇k·∇ω

μ_t = ρ · a_1 · k / max(a_1·ω, S·F2)
```

**핵심 함수:**
```
F1 = tanh(arg1⁴)
  arg1 = min(max(√k/(β*·ω·d), 500ν/(d²·ω)), 4ρ·σ_ω2·k/(CD_kω·d²))
  CD_kω = max(2ρ·σ_ω2·(1/ω)·∇k·∇ω, 1e-10)

F2 = tanh(arg2²)
  arg2 = max(2√k/(β*·ω·d), 500ν/(d²·ω))
```

**blending:**
```
φ = F1·φ_1 + (1-F1)·φ_2

상수 세트 1 (k-ω): σ_k1=0.85, σ_ω1=0.5, β1=0.075, α1=5/9
상수 세트 2 (k-ε): σ_k2=1.0, σ_ω2=0.856, β2=0.0828, α2=0.44
공통: β*=0.09, a_1=0.31, κ=0.41
```

**체크리스트:**
- [ ] F1, F2 blending 함수가 정확한가
- [ ] CD_kω에 1e-10 floor가 있는가 (0 나눗셈 방지)
- [ ] P̃_k = min(P_k, 10·β*·ρ·ω·k) 생성 제한이 적용되었는가
- [ ] μ_t의 max(a_1·ω, S·F2)에서 S는 변형률 크기인가 (와도가 아님)
- [ ] 벽면 거리 d 계산이 정확한가
- [ ] ω의 벽면 경계조건이 정확한가 (아래 참조)
- [ ] k, ω > 0 보호가 있는가

---

## 벽면 처리

### y+ 범위와 모델 선택

```
y+ = y · u_τ / ν
u_τ = √(τ_wall / ρ)
```

| y+ 범위 | 벽면 처리 | 적합한 모델 |
|---------|-----------|-------------|
| y+ < 1 | Low-Re (벽면 해상) | k-ω SST, SA |
| 1 < y+ < 5 | 위험 구간 (buffer layer) | 어느 쪽도 부정확 |
| 30 < y+ < 300 | Wall function | k-ε + wall function |
| y+ > 300 | 과도하게 조밀하지 않음 | wall function 사용 가능 |

### Wall Function (y+ > 30)

**Standard wall function:**
```
u+ = (1/κ) · ln(E · y+)    [log-law, κ=0.41, E=9.8]

u+ = u_P / u_τ,  y+ = y_P · u_τ / ν

→ τ_wall = ρ · C_μ^(1/4) · k_P^(1/2) · u_P / u+
```

**k, ε 벽면 경계조건 (wall function):**
```
k: ∂k/∂n = 0 (Neumann, zero gradient)
ε: ε_P = C_μ^(3/4) · k_P^(3/2) / (κ · y_P)   [첫 셀에서 고정]
```

**ω 벽면 경계조건 (low-Re, y+ < 5):**
```
방법 1 (Menter): ω_wall = 10 · 6ν / (β1 · Δy²)   [첫 셀 높이 Δy]
방법 2 (Wilcox): ω_wall = 6ν / (β1 · d²)          [벽면 거리 d]
```

### 벽면 처리 체크리스트
- [ ] y+ 값을 계산하여 출력하는가 (사후 검증)
- [ ] y+ < 1 인데 wall function 사용 → 결과 불정확
- [ ] y+ > 5 인데 low-Re 벽면 조건 사용 → 결과 불정확
- [ ] wall function의 u+ 계산에서 반복법(Newton) 사용하는가 (implicit u_τ)
- [ ] Spalding의 all-y+ wall function 고려 (buffer layer 커버)

## 소스항 처리

**난류 모델의 소스항은 매우 stiff하다.** Explicit 처리 시 발산.

**Implicit 소스 분리:**
```
S(φ) = S_pos - S_neg · φ    [양의 소스와 음의 소스 분리]

생성: explicit (양의 소스)
소멸: implicit (φ에 비례하는 항을 계수 행렬에 넣음)

예: ε 방정식
  S_ε = C_ε1·(ε/k)·P_k     [explicit, 양]
      - C_ε2·ρ·ε²/k          [= C_ε2·ρ·(ε/k)·ε → implicit으로 ε 계수를 A 행렬에]
```

**이 방법을 안 쓰면:** k, ε, ω가 음수로 가거나 발산.

## 초기조건 및 경계조건

### 자유류 (Freestream) 초기조건

```
k_inf = 1.5 · (Tu · U_inf)²    [Tu = 난류 강도, 보통 0.01~0.05]
ε_inf = C_μ · k_inf^(3/2) / l_t  [l_t = 난류 길이 스케일]
ω_inf = k_inf / (C_μ · ν_t_inf)  또는  ε_inf / (C_μ · k_inf)
ν̃_inf = 3~5 · ν_inf             [SA 모델]
```

### 입구/출구
```
입구: k, ε(또는 ω) 고정 (Dirichlet)
출구: ∂k/∂n = 0, ∂ε/∂n = 0 (zero gradient)
대칭: ∂k/∂n = 0, ∂ε/∂n = 0
```

## 하이브리드 RANS-LES (DES 계열)

RANS 모델을 벽 근처에서만 사용하고, 나머지 영역은 LES로 전환하는 방법.

### DES (Detached Eddy Simulation, Spalart 1997)

**핵심:** RANS 모델의 길이 스케일을 격자 크기로 교체.
```
SA 기반: d̃ = min(d, C_DES · Δ)
  d = 벽면 거리 (RANS 모드)
  Δ = max(Δx, Δy, Δz)  (LES 모드)
  C_DES = 0.65

k-ω SST 기반: β*·ω·k → β*·ω·k · F_DES
  F_DES = max(L_t / (C_DES · Δ), 1)
  L_t = √k / (β*·ω)
```

### DDES (Delayed DES, Spalart 2006)

**문제 해결:** DES는 경계층 내부에서 의도치 않게 LES 모드로 전환 (Grid-Induced Separation, GIS). DDES는 이를 방지.
```
d̃ = d - f_d · max(0, d - C_DES · Δ)

f_d = 1 - tanh((8·r_d)³)
r_d = (ν_t + ν) / (κ²·d²·√(U_{i,j}·U_{i,j}))
```
- f_d ≈ 0: 경계층 내부 → RANS 유지
- f_d ≈ 1: 경계층 밖 → DES(LES) 모드

### IDDES (Improved DDES, Shur 2008)

DDES에 Wall-Modelled LES (WMLES) 기능 추가. 격자가 충분히 세밀하면 벽 근처도 LES로 전환 가능.

### 하이브리드 RANS-LES 체크리스트
- [ ] 경계층 내부에서 RANS 모드가 유지되는가 (DDES의 f_d ≈ 0 확인)
- [ ] LES 영역에서 격자가 등방적(isotropic)에 가까운가 (Δx ≈ Δy ≈ Δz)
- [ ] **시간 적분이 2차 이상인가** (DES/LES는 시간 정확도 필수)
- [ ] 수치 스킴 확산이 충분히 낮은가 (중심 차분 또는 저확산 upwind)
- [ ] Δ 계산이 셀 형상을 반영하는가 (max vs cube root of volume)

### 하이브리드 RANS-LES Red Flags
- GIS (Grid-Induced Separation): DES에서 경계층이 비물리적으로 분리 → DDES로 전환
- Log-layer mismatch: RANS→LES 전환 영역에서 속도 프로파일 불연속 → IDDES 사용

## 검증

1. **Flat plate**: Cf(x) vs Schlichting/Wieghardt 데이터 → `/cfd-vv/benchmarks.md` 5.1
2. **Backward-facing step**: 재부착 길이 x_r/h → `/cfd-vv/benchmarks.md` 5.2
3. **Channel flow (DNS 비교)**: y+ vs u+ 프로파일이 log-law 일치
4. **법칙 확인**: 벽면 근처 u+ = y+ (viscous sublayer, y+ < 5)

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| k < 0 또는 ε < 0 또는 ω < 0 | 중단. 소스항 implicit 처리 + floor 확인. |
| y+ ∈ [5, 30] (buffer layer) | 경고. 격자 수정: y+ < 1 또는 y+ > 30. |
| wall function을 y+ < 5에서 사용 | 중단. low-Re 벽면 조건으로 변경. |
| 소멸항을 explicit으로 처리 | 중단. implicit으로 변경 (A 행렬에 넣음). |
| k-ω SST의 F1에서 d=0 (벽면 거리) | 중단. 벽면 거리 계산 확인. |
| μ_t가 μ_lam보다 1e6배 이상 | 경고. 난류 변수 초기조건 또는 소스항 확인. |
| SA의 c_b2·(∇ν̃)² 항 누락 | 중단. 자주 빠뜨리는 항. 반드시 포함. |

## 관련 스킬

- `/cfd-ns` — Navier-Stokes 점성항 (μ_eff = μ + μ_t)
- `/cfd-euler` — 기본 보존법칙
- `/cfd-implicit` — 소스항 linearization, 선형 솔버
- `/cfd-vv` — flat plate, backward-facing step 벤치마크
- `/cfd-debug-adv` — 난류 모델 발산 시 패턴 B(점진적 발산) 참조
