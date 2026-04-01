---
name: cfd-flux
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-flux.
  Guide for implementing numerical flux functions in CFD solvers.
  Use when implementing Roe, AUSM, HLL, HLLC, Rusanov, or other approximate Riemann solvers.
  Covers: flux function selection, implementation checklist per family,
  common pitfalls, and structured/unstructured grid differences.
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

# Flux 함수 구현 가이드

## 개요

Flux 함수는 유한 체적법의 핵심이다. 셀 경계면에서의 수치 flux를 어떻게 계산하느냐가 솔버의 정확도, 안정성, 강건성을 결정한다.

**핵심 원칙:** flux 함수는 반드시 (1) 보존성, (2) 일관성(consistency), (3) 수치 확산의 적절한 양을 만족해야 한다.

## 사전 조건

- `/cfd-paper-to-code`의 Step 1~2 완료 (수식 전사 + 컨벤션 매핑)
- 셀 경계면의 좌/우 상태(L/R state) 결정 방법이 확정 (1차: 셀 값 그대로, 고차: 재구성 사용)
- 면 법선 벡터의 방향 규약 확인 (owner→neighbor, L→R 등)

## 프로토콜

### Step 1: Flux 함수 선택

문제 특성에 따라 적절한 flux 함수를 선택한다.

| 특성 | 권장 flux | 이유 |
|------|-----------|------|
| 강한 충격파 + 접촉면 | **HLLC** | 접촉면 해상도 우수, 충격파 안정 |
| 저마하 유동 (~M<0.3) | **AUSM+-up**, **SLAU**, **SLAU2** | 저마하 보정 내장 |
| All-speed (M=0~5+) | **AUSM+-up2**, **SLAU2** | 전 마하수 범위 |
| 빠른 프로토타이핑 | **Rusanov (Local Lax-Friedrichs)** | 가장 간단, 확산 큼 |
| 고정밀도 요구 | **Roe** (entropy fix 포함) | 정확한 특성 분해 |
| 저마하 + 기존 Roe/HLLC | **Roe + Rieper fix**, **HLLC + Li-Gu fix** | 1줄 수정으로 저마하 개선 |
| carbuncle 방지 | **SLAU2**, **HLLC + H-correction** | 강한 충격파 안정성 |
| MHD / 다물리 | **HLL** 또는 **HLLD** | 중간파 수에 맞게 확장 가능 |

### Step 2: 공통 구현 프레임워크

모든 flux 함수는 다음 인터페이스를 따른다:

```
입력:
  Q_L, Q_R  — 좌/우 보존 변수 (또는 원시 변수)
  n         — 면 법선 벡터 (단위 벡터 또는 면적 포함)
  
출력:
  F_num     — 수치 flux 벡터 (보존 변수 기준)
```

**구현 순서:**
1. 원시 변수 계산 (ρ, u, v, w, p, H, a)
2. 법선/접선 속도 분해 (V_n = u·n_x + v·n_y + w·n_z)
3. flux 함수 고유의 계산
4. 결과를 보존 flux로 반환

### Step 3: 패밀리별 구현 가이드

---

#### 3A. Roe Flux (Roe, 1981)

**원리:** 정확한 Riemann 문제의 선형화된 근사. 좌/우 상태 사이의 평균 Jacobian을 사용.

**구현 핵심:**

1. **Roe 평균 계산** (산술 평균이 아님!)
   ```
   R = √(ρ_R / ρ_L)
   ũ = (u_L + R·u_R) / (1 + R)
   ṽ = (v_L + R·v_R) / (1 + R)
   w̃ = (w_L + R·w_R) / (1 + R)
   H̃ = (H_L + R·H_R) / (1 + R)
   ã² = (γ-1) · (H̃ - 0.5·(ũ² + ṽ² + w̃²))
   ```

2. **고유값 (파동 속도)**
   ```
   λ_1 = Ṽ_n - ã
   λ_2 = Ṽ_n      (중복도 = 차원 수)
   λ_3 = Ṽ_n + ã
   ```

3. **Entropy Fix** — 반드시 포함
   ```
   λ가 0에 가까울 때 (sonic point 근처):
   |λ| → max(|λ|, δ)   여기서 δ = ε·ã (ε ≈ 0.1~0.3)
   
   또는 Harten-Hyman fix:
   if |λ| < δ: |λ| → (λ² + δ²) / (2δ)
   ```
   **entropy fix 없이 Roe flux를 사용하면 팽창 충격파(비물리적) 발생**

4. **최종 flux**
   ```
   F_Roe = 0.5·(F_L + F_R) - 0.5·Σ |λ_k|·α_k·r_k
   ```

**체크리스트:**
- [ ] Roe 평균이 산술 평균이 아닌 density-weighted 평균인가
- [ ] ã² > 0 보호가 있는가 (음수이면 sqrt에서 NaN)
- [ ] entropy fix가 포함되어 있는가
- [ ] 고유벡터 r_k가 논문과 일치하는가 (정규화 방식 주의)
- [ ] 파동 세기 α_k 계산이 정확한가

---

#### 3B. AUSM 계열 (Liou, 1996~2006)

**원리:** 대류 flux와 압력 flux를 분리하여 각각 계산.

**핵심 구조:**
```
F_AUSM = ṁ · Φ + p̃ · D

여기서:
  ṁ = face mass flux = a_{1/2} · (ρ_L·M⁺ + ρ_R·M⁻)
  Φ = upwind state: Φ_L (if ṁ>0), Φ_R (if ṁ≤0)
  p̃ = face pressure = p_L·P⁺ + p_R·P⁻
  D = [0, n_x, n_y, n_z, 0]^T
```

**AUSM+-up (Liou, 2006) 추가 항:**
- 저마하 보정: M 계산에 `f_a` scaling + 압력 확산 `K_p` 항
- 속도 확산: `K_u` 항 (선택적)

**체크리스트:**
- [ ] M⁺, M⁻ 분리 함수가 |M|≤1과 |M|>1 분기를 정확히 처리하는가
- [ ] P⁺, P⁻ 압력 분리 함수가 M⁺, M⁻와 호환되는가
- [ ] face 음속 a_{1/2} 계산 방법이 논문과 일치하는가
- [ ] 정지 접촉면 테스트를 통과하는가 (ṁ = 0이어야 함)
- [ ] 저마하 보정 계수 (K_p, K_u, f_a)가 논문 값과 일치하는가
- [ ] `ṁ > 0` 판정에서 부호가 면 법선 방향과 일치하는가

---

#### 3C. HLL / HLLC (Harten-Lax-van Leer, 1983 / Toro, 1994)

**HLL 원리:** 2개 파동 속도(S_L, S_R)로 근사. 접촉면을 해상하지 못함.

**HLLC 원리:** 3개 파동 속도(S_L, S_*, S_R)로 근사. 접촉면 해상.

**HLLC 핵심 구조:**
```
        S_L     S_*     S_R
    ────┤───────┤───────┤────
    U_L │ U_*L  │ U_*R  │ U_R

if S_L ≥ 0:       F = F_L
if S_L < 0 ≤ S_*: F = F_*L = F_L + S_L·(U_*L - U_L)
if S_* < 0 ≤ S_R: F = F_*R = F_R + S_R·(U_*R - U_R)
if S_R < 0:       F = F_R
```

**파동 속도 추정 (Davis, 1988):**
```
S_L = min(V_nL - a_L, V_nR - a_R)  또는  min(V_nL - a_L, Ṽ_n - ã)
S_R = max(V_nL + a_L, V_nR + a_R)  또는  max(V_nR + a_R, Ṽ_n + ã)
S_* = (p_R - p_L + ρ_L·V_nL·(S_L - V_nL) - ρ_R·V_nR·(S_R - V_nR))
      / (ρ_L·(S_L - V_nL) - ρ_R·(S_R - V_nR))
```

**체크리스트:**
- [ ] S_L, S_R 추정이 정확한가 (Davis / Einfeldt / Batten 중 어느 것?)
- [ ] S_* 계산에서 분모가 0이 되는 경우 보호가 있는가
- [ ] U_*L, U_*R 상태가 Rankine-Hugoniot 조건을 만족하는가
- [ ] 4분기 조건문의 경계값 처리가 정확한가 (등호 위치)
- [ ] 접촉면에서 압력과 법선속도가 연속인가

---

#### 3D. Rusanov / Local Lax-Friedrichs

**가장 간단한 flux 함수.**

```
F_Rusanov = 0.5·(F_L + F_R) - 0.5·S_max·(U_R - U_L)

S_max = max(|V_nL| + a_L, |V_nR| + a_R)
```

**체크리스트:**
- [ ] S_max 계산이 절대값을 포함하는가
- [ ] 확산이 과도하다면 고차 재구성(`/cfd-reconstruction`)으로 보상되고 있는가

---

#### 3E. SLAU / SLAU2 (Shima & Kitamura, 2011)

**원리:** AUSM 계열과 유사하지만, 고유값 기반이 아닌 압력 확산항(pressure-diffusion)을 사용. 저마하에서 우수한 성능.

**핵심 구조:**
```
ṁ = 0.5·(ρ_L·(V_nL + |V̄_n|) + ρ_R·(V_nR - |V̄_n|)) - (χ/a̅)·Δp

|V̄_n| = (1-g)·|V̄_n|_original + g·a̅    [g = -max(min(M_L,0),-1)·min(max(M_R,0),1)]
χ = (1 - M̂)²                          [M̂ = min(1, √(0.5·(V_nL² + V_nR²))/a̅)]

p_face = 0.5·(p_L + p_R) + 0.5·(P⁺_L - P⁻_R)·(p_L - p_R) + √(0.5·(u_L²+u_R²))·(P⁺_L + P⁻_R - 1)·0.5·(ρ_L+ρ_R)·a̅
```

**SLAU2 개선:** carbuncle 현상 억제를 위한 추가 압력 확산항.

**체크리스트:**
- [ ] χ 함수가 M→0에서 1, M→∞에서 0으로 수렴하는가
- [ ] g 함수가 초음속에서 올바르게 작동하는가
- [ ] a̅ (face 음속)의 계산이 AUSM+-up과 일치하는가
- [ ] 정지 접촉면 테스트 통과하는가

---

#### 3F. 저마하 보정 (Low-Mach Fix)

**기존 Roe/HLLC에 1줄 수정으로 저마하 성능을 개선하는 방법.**

**Rieper fix (JCP, 2011) — Roe에 적용:**
```
기존: |Δp| 관련 파동 세기를 그대로 사용
수정: 압력-속도 결합 파동의 세기를 f(M)로 스케일링
  f(M) = min(1, M_face)

효과: O(1/M) 수치 확산 → O(1)로 축소
```

**Li-Gu fix — AUSM 계열에 적용:**
```
pressure diffusion 항에 f(M) = min(1, M) 곱하기
```

**주의:** 경계 조건(입구/출구)에서는 압력 파동이 물리적이므로 스케일링을 적용하지 않는다.

---

### Step 4: 다차원 확장

1D flux를 다차원에 적용할 때의 주의점:

1. **법선/접선 분해**: 모든 속도를 면 법선 좌표계로 변환
   ```
   V_n = u·n_x + v·n_y + w·n_z  (법선 속도)
   V_t1, V_t2 = 접선 성분 (접촉면과 함께 전파)
   ```

2. **flux를 전역 좌표로 반환**: 면 법선 좌표에서 계산한 flux를 전역 좌표로 변환
   ```
   운동량 flux의 x-성분 = F_momentum_n · n_x + F_momentum_t1 · t1_x + ...
   ```

3. **면적 스케일링**: 단위 법선 벡터 사용 시 면적을 별도 곱셈. 면적 벡터 사용 시 내부에서 처리.

## 구조격자 vs 비구조격자

| 항목 | 구조격자 | 비구조격자 |
|------|----------|------------|
| 면 순회 | i+1/2, j+1/2, k+1/2 방향별 | face 리스트 순회 |
| L/R 결정 | i ↔ i+1 (방향 고정) | owner ↔ neighbour (방향이 면마다 다름) |
| 법선 벡터 | 격자 방향과 정렬 (dx, dy, dz 기반) | 면마다 별도 계산 필요 |
| 고차 재구성 | 규칙적 스텐실 (i-2, i-1, i, i+1, i+2) | 이웃 셀 탐색 필요 (k-exact 등) |
| ghost cell | 방향별 일정 수 | 경계 면별 관리 |

**비구조격자 추가 주의:**
- face 법선 방향이 owner→neighbour로 고정되어 있는지 확인
- face별로 L/R이 뒤집힐 수 있음 — flux 부호 일관성 필수
- gradient 재구성 시 least squares 또는 Green-Gauss 방법 필요

## 공통 검증 방법

모든 flux 함수 구현 후 반드시 수행:

1. **정지 접촉면**: U_L = U_R (속도와 압력 동일, 밀도만 다름) → flux가 정확히 F(U_L) = F(U_R)
2. **Sod shock tube**: 해석해와 비교 (→ `/cfd-vv/benchmarks.md` 1.1)
3. **Freestream preservation**: 균일 유동 → 잔차 = 0 (machine precision)
4. **대칭성**: F(U_L, U_R, n) = -F(U_R, U_L, -n) 확인
5. **극한 마하수**: M→0 (비압축 극한), M→∞ (강한 충격파)

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| Roe flux에 entropy fix 없음 | 중단. 반드시 추가. |
| L/R 상태가 면 법선 방향과 불일치 | 중단. 컨벤션 매핑 테이블 확인 (`/cfd-paper-to-code` Step 2). |
| HLLC에서 S_* 분모 보호 없음 | 중단. 영 나눗셈 보호 추가. |
| AUSM에서 정지 접촉면 테스트 실패 | 중단. mass flux ṁ 계산 재검토. |
| 구조/비구조 전환 시 flux 부호 반전 | 중단. face 법선 방향 규약 재확인. |
| Freestream preservation 실패 | 중단. 면적/체적 계산 또는 metric term 오류. |

## 관련 스킬

- `/cfd-paper-to-code` — flux 함수 논문의 수식 전사
- `/cfd-reconstruction` — 고차 재구성으로 L/R 상태 결정
- `/cfd-vv` — flux 함수 검증 (Sod, freestream preservation)
- `/cfd-debug-adv` — flux 관련 발산 시 패턴 A(즉시 발산) 참조
- `/cfd-euler` — Euler 방정식의 flux 형태
