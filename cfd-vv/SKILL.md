---
name: cfd-vv
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-vv.
  Verification and Validation protocol for CFD solvers.
  Use when verifying a new numerical scheme implementation, measuring order of accuracy,
  performing grid convergence studies, or validating against benchmark data.
  Covers: Method of Manufactured Solutions (MMS), grid convergence index (GCI),
  order of accuracy verification, and benchmark comparison methodology.
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

# 검증 및 확인 (Verification & Validation)

## 개요

"코드가 돌아간다"는 검증이 아니다. **Verification**은 "방정식을 올바르게 풀고 있는가"이고, **Validation**은 "올바른 방정식을 풀고 있는가"이다.

**핵심 원칙:** 모든 새로운 수치 스킴 구현은 반드시 수렴 차수(order of accuracy)를 정량적으로 확인한 후에야 "구현 완료"로 간주한다.

## 사전 조건

- 구현한 스킴의 이론적 수렴 차수를 알고 있어야 한다 (e.g., 2nd order, 5th order)
- 솔버가 최소 3개 이상의 격자 해상도에서 실행 가능해야 한다
- 시간 적분의 차수가 공간 차수보다 같거나 높아야 한다 (시간 오차가 지배하지 않도록)

## 프로토콜

### Step 1: 검증 전략 선택

구현 상황에 따라 적절한 검증 방법을 선택한다.

```
해석해가 존재하는가?
├── YES → Step 2A: 해석해 비교
└── NO
    ├── 소스항을 자유롭게 추가할 수 있는가?
    │   ├── YES → Step 2B: MMS (Method of Manufactured Solutions)
    │   └── NO → Step 2C: 격자 수렴만으로 검증
    └── 벤치마크 데이터가 있는가?
        └── YES → Step 3: 벤치마크 비교 (Validation)
```

### Step 2A: 해석해 비교

**사용 조건:** 해석해(exact solution)가 알려진 문제

**절차:**
1. 해석해를 코드에 구현한다 (별도 함수)
2. 최소 4개 격자 해상도로 실행한다 (e.g., N=32, 64, 128, 256)
3. 각 해상도에서 L1, L2, L∞ 오차 노름을 계산한다
4. Step 4(수렴 차수 측정)로 진행한다

**오차 노름 계산:**
```
L1  = (1/N) Σ |u_numerical - u_exact|
L2  = sqrt( (1/N) Σ (u_numerical - u_exact)² )
L∞  = max |u_numerical - u_exact|
```
- 비균일 격자: 셀 체적으로 가중 평균해야 한다
- 벡터량: 각 성분별로 별도 계산

### Step 2B: Method of Manufactured Solutions (MMS)

**사용 조건:** 해석해가 없지만 소스항을 추가할 수 있는 경우

**원리:** 임의의 해석 함수를 "정답"으로 가정하고, 이를 지배 방정식에 대입하여 필요한 소스항을 역산한다.

**절차:**

1. **제조해 설계**: 충분히 부드러운 해석 함수를 선택한다
   ```
   예: ρ(x,y,t) = ρ_0 + ρ_1 · sin(2πx/L) · cos(2πy/L) · cos(ωt)
       u(x,y,t) = u_0 + u_1 · sin(2πx/L) · sin(2πy/L) · cos(ωt)
       ...
   ```
   
   **제조해 설계 규칙:**
   - 모든 변수가 공간적·시간적으로 변해야 한다 (상수 성분이 있으면 해당 항의 검증 불가)
   - 물리적으로 실현 가능해야 한다 (ρ > 0, p > 0, T > 0)
   - 주기 함수를 사용하면 경계조건이 간단해진다
   - 변수들이 서로 독립적으로 변해야 한다 (같은 sin으로 통일하면 상쇄 가능)

2. **소스항 유도**: 제조해를 지배 방정식에 대입하여 소스항 S를 구한다
   ```
   ∂Q/∂t + ∇·F(Q) = S
   여기서 S = ∂Q_mms/∂t + ∇·F(Q_mms)  (해석적으로 계산)
   ```
   
   **주의:**
   - 소스항 유도는 반드시 기호 계산(symbolic computation)으로 수행한다 (수작업 실수 방지)
   - SymPy, Mathematica, Maple 등 사용 권장
   - 유도된 소스항을 코드에 넣기 전에, 한 점에서 수치 확인한다

3. **솔버 실행**: 소스항을 포함하여 최소 4개 격자 해상도로 실행
4. **오차 계산**: 제조해와 수치해의 L1/L2/L∞ 오차 노름
5. Step 4(수렴 차수 측정)로 진행

### Step 2C: 격자 수렴 (해석해 없음)

**사용 조건:** 해석해도 없고 소스항 추가도 어려운 경우

**절차:**
1. 최소 3개 격자 해상도로 실행 (격자 세밀화 비율 r은 일정하게, 보통 r=2)
2. 동일 위치에서 물리량 추출
3. Richardson extrapolation으로 외삽해 추정
4. GCI(Grid Convergence Index) 계산으로 불확실도 평가
5. Step 4로 진행

### Step 3: 벤치마크 비교 (Validation)

**사용 조건:** 실험 데이터 또는 잘 검증된 수치 결과가 있는 경우

**절차:**
1. 벤치마크의 정확한 조건을 재현한다
   - 격자, 초기조건, 경계조건, 물성치를 논문과 동일하게
   - 차이가 있으면 반드시 기록
2. 논문과 동일한 물리량을 동일한 위치/시간에서 추출한다
3. 정량 비교 수행:
   - 프로파일 비교: 동일 라인의 값 오버레이
   - 적분량 비교: Cd, Cl, Nu, St 등 (허용 오차 명시)
   - 특이점 비교: 충격파 위치, 분리점, 재부착점
4. 결과를 `/cfd-vv/benchmarks.md`의 해당 케이스와 대조한다

**주의:** Validation은 Verification이 완료된 후에 수행한다. 코드가 방정식을 잘못 풀고 있으면 실험과 비교해도 의미 없다.

### Step 4: 수렴 차수 측정

**이 단계가 Verification의 핵심이다.**

**절차:**

1. **오차 테이블 작성:**
   ```markdown
   | 격자 크기 h | L1 error | L2 error | L∞ error |
   |-------------|----------|----------|----------|
   | 1/32        | 2.45e-3  | 3.12e-3  | 8.91e-3  |
   | 1/64        | 6.14e-4  | 7.83e-4  | 2.24e-3  |
   | 1/128       | 1.54e-4  | 1.96e-4  | 5.61e-4  |
   | 1/256       | 3.85e-5  | 4.90e-5  | 1.40e-4  |
   ```

2. **수렴 차수 계산:**
   ```
   p = log(e_coarse / e_fine) / log(r)
   
   여기서:
     e_coarse, e_fine = 두 연속 격자의 오차
     r = 격자 세밀화 비율 (h_coarse / h_fine)
   ```

3. **판정 기준:**
   | 이론 차수 | 관측 차수 | 판정 |
   |-----------|-----------|------|
   | p_theory  | p_obs ∈ [p_theory - 0.1, p_theory + 0.2] | PASS |
   | p_theory  | p_obs < p_theory - 0.5 | FAIL — 구현 오류 가능 |
   | p_theory  | p_obs > p_theory + 0.5 | 주의 — 우연의 상쇄 가능 |

4. **실패 시 진단:**
   - L∞만 차수가 낮은 경우 → 경계 근처 오류 가능 (경계조건 정밀도 확인)
   - 모든 노름에서 차수가 낮은 경우 → 스킴 구현 오류 (코드 리뷰)
   - 차수가 1 근처인 경우 → 1차 오차 지배 (limiter 또는 upwind bias 확인)
   - 차수가 불규칙한 경우 → 격자가 충분히 세밀하지 않을 수 있음 (더 세밀한 격자 추가)

### Step 5: GCI (Grid Convergence Index) 계산

**비균일 격자나 해석해 없는 경우의 불확실도 정량화.**

```
GCI_fine = Fs * |e_a| / (r^p - 1)

여기서:
  Fs = 안전계수 (3개 격자: 1.25, 2개 격자: 3.0)
  e_a = (f_fine - f_coarse) / f_fine  (상대 오차)
  r = 격자 세밀화 비율
  p = 관측된 수렴 차수
```

**점근적 수렴 범위 확인:**
```
GCI_23 / (r^p * GCI_12) ≈ 1.0  (±5% 이내이면 점근적 범위)
```

## 체크리스트

### Verification 완료 전 확인
- [ ] 최소 4개 격자 해상도에서 실행했는가
- [ ] L1, L2, L∞ 오차 노름을 모두 계산했는가
- [ ] 관측 수렴 차수가 이론 차수와 일치하는가 (±0.1)
- [ ] 시간 오차가 공간 오차를 지배하지 않는가 (시간 적분 차수 ≥ 공간 차수)
- [ ] 비균일 격자의 경우 셀 체적 가중 평균을 사용했는가
- [ ] 오차 테이블과 수렴 차수가 문서에 기록되었는가

### MMS 사용 시 추가 확인
- [ ] 제조해의 모든 변수가 공간·시간적으로 변하는가
- [ ] ρ > 0, p > 0, T > 0 조건을 만족하는가
- [ ] 소스항이 기호 계산으로 유도되었는가
- [ ] 소스항을 한 점에서 수치적으로 검증했는가

### Validation 시 추가 확인
- [ ] Verification이 먼저 완료되었는가
- [ ] 벤치마크 조건과 코드 설정의 차이가 기록되었는가
- [ ] 비교 기준(허용 오차)이 사전에 정의되었는가

## Red Flags — STOP

| 상황 | 조치 |
|------|------|
| Verification 없이 실험 데이터와 비교 | 중단. Verification 먼저. |
| 2개 격자만으로 "수렴 확인" | 중단. 최소 3개, 권장 4개. |
| 수렴 차수를 계산하지 않고 "오차가 줄어드니까 OK" | 중단. 차수를 정량적으로. |
| MMS 소스항을 손으로 유도 | 중단. 기호 계산으로 재유도. |
| 시간 적분 차수 < 공간 차수 | 중단. 시간 적분 차수를 높이거나 dt를 충분히 줄인다. |
| 격자 세밀화 비율이 불균일 (2x, 3x, 2x) | 중단. 일정한 비율 사용. |
| "대략 2차이니까 OK" (관측 차수 1.5) | 중단. 0.5 차이는 구현 오류 가능. |

## 워크플로 연계

```
/cfd-paper-to-code (구현) → /cfd-vv (검증) → /cfd-debug-adv (문제 시 디버깅)
```

- 벤치마크 케이스 목록: `cfd-vv/benchmarks.md`

## 관련 스킬

- `/cfd-paper-to-code` — 수식 구현 시 오류 방지
- `/cfd-debug-adv` — 수렴 차수가 맞지 않을 때 디버깅
- `/cfd-flux` — flux 함수 검증 시 활용
- `/cfd-time-integration` — 시간 적분 차수 확인
