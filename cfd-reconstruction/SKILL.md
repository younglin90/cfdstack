---
name: cfd-reconstruction
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-reconstruction.
  Guide for implementing high-order spatial reconstruction in FVM solvers.
  Use when implementing MUSCL, WENO, compact, or other reconstruction schemes
  to determine left/right states at cell faces.
  Covers: limiter selection, stencil management, ghost cell requirements,
  and characteristic vs component-wise reconstruction.
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

# 고차 재구성 가이드

## 개요

고차 재구성(reconstruction)은 셀 평균값으로부터 셀 경계면의 좌/우 상태를 고정밀도로 추정하는 과정이다. 1차 스킴(셀 값 그대로 사용)보다 높은 정밀도를 얻으려면 반드시 필요하다.

**핵심 원칙:** 재구성의 차수가 솔버 전체의 공간 정밀도를 결정한다. 재구성 차수 > flux 함수 차수이면 재구성이 무의미하고, 재구성 차수 < 시간 적분 차수이면 공간 오차가 지배한다.

## 사전 조건

- 목표 공간 정밀도 차수를 결정했는가 (2nd, 3rd, 5th 등)
- 사용 가능한 ghost cell 수를 확인했는가
- 구조격자 / 비구조격자 확인

## 프로토콜

### Step 1: 재구성 스킴 선택

| 목표 차수 | 구조격자 추천 | 비구조격자 추천 | ghost cell 수 (한 쪽) |
|-----------|--------------|----------------|----------------------|
| 2차 | MUSCL + limiter | Green-Gauss / LSQ gradient | 1~2 |
| 3차 | MUSCL-Hancock (κ=1/3) | k-exact (k=2) | 2 |
| 5차 | WENO-JS / WENO-Z | WENO on unstructured | 3 |
| 6차+ | Compact (implicit) | Compact DG 기반 | 스킴 의존 |

### Step 2: 스킴별 구현 가이드

---

#### 2A. MUSCL (Monotone Upstream-centered Schemes for Conservation Laws)

**원리:** 셀 내부에서 선형 분포를 가정하고, limiter로 진동을 억제.

**기본 형태 (1D, 셀 i의 오른쪽 면 i+1/2):**
```
q_L = q_i   + 0.5 · φ(r) · (q_i - q_{i-1})     [또는]
q_L = q_i   + 0.25 · ((1-κ)·Δ⁻ + (1+κ)·Δ⁺)     [κ-scheme]
q_R = q_{i+1} - 0.5 · φ(1/r) · (q_{i+1} - q_i)

여기서:
  Δ⁻ = q_i - q_{i-1},  Δ⁺ = q_{i+1} - q_i
  r = Δ⁺ / Δ⁻  (기울기 비)
  κ = -1 (fully upwind), 0 (Fromm), 1/3 (3rd order upwind-biased), 1 (central)
```

**Limiter 선택:**
| Limiter | 공식 φ(r) | 특성 |
|---------|-----------|------|
| minmod | max(0, min(1, r)) | 가장 확산적, 가장 안정 |
| van Leer | (r + \|r\|) / (1 + \|r\|) | 부드러운 전환, 좋은 균형 |
| van Albada | (r² + r) / (r² + 1) | 부드러운, 미분 가능 |
| superbee | max(0, min(2r, 1), min(r, 2)) | 가장 날카로운, 가장 덜 확산 |
| MC (monotonized central) | max(0, min(2r, (1+r)/2, 2)) | superbee와 minmod 사이 |

**체크리스트:**
- [ ] r 계산에서 Δ⁻ = 0일 때 보호가 있는가 (ε 추가 또는 부호 판정)
- [ ] limiter가 TVD 조건을 만족하는가 (Sweby diagram 내부)
- [ ] κ 값이 논문과 일치하는가
- [ ] 좌/우 재구성의 기울기 방향이 일관적인가
- [ ] 비구조격자: gradient를 limiter와 결합하는 방식이 Barth-Jespersen 또는 Venkatakrishnan 인가

---

#### 2B. WENO (Weighted Essentially Non-Oscillatory)

**원리:** 여러 서브스텐실의 다항식 보간을 비선형 가중 조합. 불연속 근처에서는 smooth stencil에 가중치를 집중.

**WENO-5 (WENO-JS, Jiang & Shu, 1996):**

3개 서브스텐실로 face 값을 재구성 (i+1/2 면, left-biased):
```
스텐실 0: {i-2, i-1, i}     → q₀ = (1/3)q_{i-2} - (7/6)q_{i-1} + (11/6)q_i
스텐실 1: {i-1, i, i+1}     → q₁ = -(1/6)q_{i-1} + (5/6)q_i + (1/3)q_{i+1}
스텐실 2: {i, i+1, i+2}     → q₂ = (1/3)q_i + (5/6)q_{i+1} - (1/6)q_{i+2}

이상적 가중치: d₀ = 1/10, d₁ = 6/10, d₂ = 3/10
```

**Smoothness Indicators (β):**
```
β₀ = (13/12)(q_{i-2} - 2q_{i-1} + q_i)² + (1/4)(q_{i-2} - 4q_{i-1} + 3q_i)²
β₁ = (13/12)(q_{i-1} - 2q_i + q_{i+1})² + (1/4)(q_{i-1} - q_{i+1})²
β₂ = (13/12)(q_i - 2q_{i+1} + q_{i+2})² + (1/4)(3q_i - 4q_{i+1} + q_{i+2})²
```

**비선형 가중치:**
```
α_k = d_k / (ε + β_k)²    [WENO-JS]
ω_k = α_k / Σα_k

최종: q_{i+1/2}^L = Σ ω_k · q_k
```

**WENO-Z (Borges et al., 2008) — 개선된 가중치:**
```
τ₅ = |β₀ - β₂|
α_k = d_k · (1 + (τ₅ / (ε + β_k))^p)    [p = 1 또는 2]
```
WENO-Z는 smooth 영역에서 이상적 가중치에 더 가까워 정확도가 높다.

**ε 값:**
- WENO-JS: ε = 1e-6 (전통적) 또는 1e-40 (Henrick et al.)
- WENO-Z: ε = 1e-40 (너무 크면 smooth 영역 정확도 저하)
- **ε이 결과에 영향을 미치면 안 됨** — 영향이 크다면 구현 오류 의심

**체크리스트:**
- [ ] 서브스텐실 보간 계수가 논문과 일치하는가
- [ ] 이상적 가중치 합이 1인가 (d₀ + d₁ + d₂ = 1)
- [ ] smoothness indicator β의 계수가 정확한가 (13/12, 1/4)
- [ ] **스텐실 순서(S0, S1, S2)가 일관적인가** — 뒤집히면 smooth에서 정상이나 충격파에서 실패
- [ ] ghost cell이 3개 이상인가 (2개이면 초기화 안 된 메모리 참조)
- [ ] right-biased 재구성은 left-biased의 대칭(mirror)으로 구현했는가
- [ ] ε 값이 적절한가 (1e-6 vs 1e-40, 스킴에 따라 다름)

---

#### 2C. Compact (Implicit) 스킴

**원리:** 면 값이 이웃 면 값과 암묵적으로 연결되는 연립방정식. 좁은 스텐실로 높은 차수 달성.

**일반 형태:**
```
β·q_{i-1/2} + q_{i+1/2} + β·q_{i+3/2} = a·q_i + b·q_{i+1}

β, a, b 값에 따라 4차, 6차 등 가능
```

**체크리스트:**
- [ ] 삼중대각(tridiagonal) 또는 오중대각 시스템이 올바르게 조립되는가
- [ ] 경계에서 시스템 닫기(closure)가 명시되어 있는가 (one-sided stencil 사용)
- [ ] 주기 경계 시 순환 삼중대각 솔버를 사용하는가
- [ ] 비구조격자에 적용할 경우 DG 기반 변환인가

---

#### 2D. TENO (Targeted ENO, Fu et al., 2016)

**원리:** WENO의 smooth 가중치 대신, 각 서브스텐실을 "smooth" 또는 "non-smooth"로 **이진 분류**. 불연속 근처에서 더 날카로운 해상도.

**WENO와의 차이:**
```
WENO: ω_k = d_k / (ε + β_k)^p          [연속적 가중치, 0~1 사이]
TENO: δ_k = { 1 if C_T·β_k/β_max < 1   [이진 선택: 0 또는 1]
             { 0 otherwise

최종 가중치: ω_k = d_k·δ_k / Σ(d_j·δ_j)
```

**장점:** smooth 영역에서 이상적 차수에 더 가까움, 불연속에서 진동이 적음
**단점:** C_T 파라미터 선택이 민감 (C_T = 1e-5 ~ 1e-7, 문제 의존적)

**체크리스트:**
- [ ] β_k (smoothness indicator)는 WENO와 동일한 공식 사용
- [ ] C_T 값이 논문과 일치하는가 (TENO5: C_T = 1e-6 기본)
- [ ] 모든 δ_k = 0이 되는 경우 fallback이 있는가 (1차로 전환)
- [ ] smooth 문제에서 이론 차수가 WENO보다 같거나 높은가

---

#### 2E. 양보존 Limiter (Zhang-Shu, 2010)

**목적:** 고차 재구성 후에도 ρ > 0, p > 0을 보장.

**원리:** 재구성된 face 값을 셀 평균과 혼합하여 양보존 달성.
```
재구성 후:
  q_face_limited = θ · (q_face - q̄) + q̄

  θ = min(1, (q̄ - q_min) / (q̄ - q_face_min))

  q̄ = 셀 평균 (양수 보장됨)
  q_face_min = 모든 Gauss 점에서의 최소값
  q_min = 허용 최소값 (ε > 0)
```

**적용 순서:**
1. 먼저 밀도 ρ에 적용 → ρ > 0 보장
2. 그 다음 압력 p에 적용 → p > 0 보장
3. **에너지 ρE에 직접 적용하지 않는다** — p를 통해 간접 제한

**체크리스트:**
- [ ] ρ와 p 각각에 별도로 적용했는가
- [ ] θ 계산에서 q̄ = q_face_min일 때 0/0 보호가 있는가
- [ ] 양보존 limiter가 재구성 후, flux 계산 전에 적용되는가
- [ ] 이 limiter가 수렴 차수를 저하시키지 않는가 (smooth에서 θ = 1)

---

### Step 3: 재구성 변수 선택

어떤 변수를 재구성하느냐에 따라 결과가 달라진다.

| 재구성 변수 | 장점 | 단점 |
|------------|------|------|
| 보존 변수 (ρ, ρu, ρE) | 보존법칙에 직접 대응 | 음의 압력/밀도 발생 가능 |
| 원시 변수 (ρ, u, p) | 물리적 해석 직관적, 음압 덜 발생 | 보존성 약간 저하 |
| 특성 변수 (characteristic) | 진동 최소, 최적의 limiter 성능 | 구현 복잡, 비용 높음 |

**권장:**
- 기본: **원시 변수** (안정성 + 단순함)
- 강한 충격파: **특성 변수** (Roe 평균에서의 고유벡터로 투영/역투영)
- 저마하: **원시 변수** (보존 변수 재구성은 저마하에서 불안정 가능)

### Step 4: 다차원 확장

**구조격자:**
- 차원별 분리(dimension-by-dimension): x, y, z 각각 1D 재구성 적용
- 간단하지만 엄밀한 다차원 정확도는 아님 (corner coupling 부재)

**비구조격자:**
- gradient 기반: ∇q를 Green-Gauss 또는 LSQ로 계산 → face에서 선형 외삽
- k-exact: 이웃 셀을 사용한 다항식 피팅 → face에서 평가
- WENO: 비구조 격자용 WENO 가중치 계산 (기하학적 가중치 필요)

## 공통 검증 방법

1. **수렴 차수 테스트**: smooth 문제 (isentropic vortex)에서 이론 차수 확인 → `/cfd-vv`
2. **Shu-Osher 문제**: 고주파 진동 해상도 비교 → `/cfd-vv/benchmarks.md` 1.3
3. **정지 접촉면**: limiter가 접촉면을 과도하게 평탄화하지 않는지 확인
4. **1차로 전환**: limiter/WENO를 끄면 1차 수렴으로 돌아가는지 확인 (코드 경로 검증)

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| WENO ghost cell이 스텐실 폭보다 적음 | 중단. ghost cell 수 확인 (WENO-5: 최소 3개). |
| smooth 문제에서 이론 차수가 안 나옴 | 중단. 보간 계수 재확인. |
| limiter를 끄면 발산하지만 켜면 1차 | 중단. limiter가 너무 강하게 작동 (φ≈0 항상). |
| 충격파에서 진동 (Gibbs 현상) | 중단. limiter/WENO 가중치 로직 재검토. |
| 비구조격자에서 freestream preservation 실패 | 중단. gradient 재구성 + 면적분 일관성 확인. |
| WENO의 ε 값을 바꾸면 결과가 크게 변함 | 중단. smoothness indicator 계산 오류 의심. |

## 관련 스킬

- `/cfd-flux` — 재구성된 L/R 상태를 사용하는 flux 함수
- `/cfd-paper-to-code` — 재구성 스킴 논문의 수식 전사
- `/cfd-vv` — 수렴 차수 검증 (isentropic vortex)
- `/cfd-time-integration` — 시간 적분 차수와의 균형 확인
