---
name: cfd-debug
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-debug.
  Diagnose and fix CFD simulation convergence and stability problems.
  Identifies divergence cause, checks CFL number, detects bad mesh regions,
  validates boundary conditions, and recommends targeted fixes with before/after
  verification. Iron Law: never change solver settings without identifying root cause.
  Use when asked to "debug divergence", "simulation is blowing up", "residuals not
  converging", "fix instability", "CFL too high", or "solver crashed".
  Proactively suggest when /cfd-run reports divergence or failed convergence.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Glob
  - Grep
  - AskUserQuestion

---

# /cfd-debug — CFD 수렴 및 안정성 문제 진단

## 철칙

**근본 원인을 파악하지 않고 솔버 설정을 변경하지 않습니다.**
3단계 접근법: (1) 진단, (2) 가설 수립, (3) 수정 및 검증.
세 번의 수정 시도가 실패하면 중단하고 명확한 문제 설명과 함께 에스컬레이션합니다.

## 1단계: 문제 분류

사용자가 관찰한 내용을 확인하거나 로그를 직접 확인합니다:

```bash
# 솔버 로그 찾기
LOG=$(ls *.log log.* /tmp/cfdstack-solver.log 2>/dev/null | head -1)
echo "로그: $LOG"

# 충돌 징후 확인
if [ -n "$LOG" ]; then
  echo "=== 마지막 50줄 ==="
  tail -50 "$LOG"
  echo ""
  echo "=== 오류/경고 요약 ==="
  grep -iE "(fatal|error|nan|inf|floating|diverged|negative|illegal)" "$LOG" | tail -20
fi
```

발견된 내용을 바탕으로 문제를 분류합니다:

| 증상 | 예상 원인 |
|------|-----------|
| 잔차가 단조 증가 | 불안정한 이산화, 너무 큰 Δt, 또는 잘못된 BC |
| `FOAM FATAL ERROR: Floating point exception` | 필드에 NaN/Inf — 나쁜 초기 조건 또는 Δt가 너무 큼 |
| `Negative initial residual` | 불일치한 BC 또는 특이 압력 시스템 |
| 잔차 진동 후 미수렴 | 수치 진동, 불충분한 완화 |
| 매우 느린 수렴 (> 10,000회 반복) | 과도한 완화 감쇠, 나쁜 메시, 또는 잘못된 물리 모델 |
| `Courant Number mean/max`가 너무 높음 | 명시적 기법에 시간 간격이 너무 큼 |

## 2단계: CFL / Courant 수 확인

```bash
# 로그에서 Courant 수 이력 추출
LOG=$(ls *.log log.* /tmp/cfdstack-solver.log 2>/dev/null | head -1)
grep -E "Courant Number|CFL" "$LOG" 2>/dev/null | tail -20
```

**Courant 수 임계값:**

| 솔버 | 최대 CFL (안정) |
|------|----------------|
| icoFoam (명시적) | < 1.0 |
| pimpleFoam (암시적) | < 5-10 (nCorrectors ≥ 2인 경우) |
| simpleFoam (정상 상태) | 해당 없음 |
| rhoCentralFoam | < 0.5 |

CFL > 한계값이면 `system/controlDict`의 `deltaT`를 줄입니다. 권장:
- 새 Δt = 현재 Δt × (CFL 한계 / CFL 최대) × 0.8 (안전 계수)

적용할 편집 내용을 표시합니다:

```
// system/controlDict — deltaT 변경
deltaT          <새 값>;  // 이전 <기존 값>, CFL < 1을 위해 감소
```

자동 시간 간격 조정을 위해 `adjustTimeStep yes;`와 `maxCo 0.9;` 활성화도 고려합니다.

## 3단계: 필드의 NaN / Inf 확인

```bash
# 최신 시간 디렉토리에서 NaN/Inf 확인
LATEST=$(ls -d [0-9]*/ 2>/dev/null | sort -V | tail -1)
echo "$LATEST 에서 NaN/Inf 확인 중..."
for field in $LATEST*; do
  [ -f "$field" ] || continue
  if grep -qE "nan|inf|NaN|Inf" "$field" 2>/dev/null; then
    echo "  NaN/Inf 발견: $field"
  fi
done
```

필드에서 NaN이 발견되면 전파 위치를 추적합니다:
1. `0/` 초기 조건에서 비합리적인 값 확인
2. 호환되지 않는 BC 유형 확인 (예: 입구와 출구 모두 `zeroGradient` 압력)
3. 점성/밀도 값이 물리적으로 합리적인지 확인

## 4단계: 경계 조건 오류 진단

일반적인 BC 오류와 수정 방법:

### 1. 압력이 구속되지 않음 (순수 Neumann 문제)
**증상:** `singular or near-singular matrix`, 압력 미수렴
**원인:** 모든 압력 BC가 `zeroGradient` — 기준 압력 미설정
**수정:** 하나의 출구를 `fixedValue 0;`으로 설정하거나 fvSolution에 `pRefCell/pRefValue` 추가

```bash
grep -A3 "outlet\|SIMPLE\|pRef" system/fvSolution 0/p
```

### 2. 입구 속도 zeroGradient 압력 + 출구 fixedValue 압력
대부분의 내부 유동에 올바른 설정입니다. 확인:

```bash
echo "=== 입구 BC ==="
grep -A5 "inlet" 0/U 0/p
echo "=== 출구 BC ==="
grep -A5 "outlet" 0/U 0/p
```

### 3. 잘못된 난류 초기 조건
**증상:** k 또는 epsilon이 즉시 발산
**원인:** k 또는 epsilon 초기값이 0이거나 너무 작음
**수정:** 난류 강도 방법 사용:
- k = 1.5 × (I × U)²   (I = 0.05, 즉 5% 난류 강도)
- ε = Cμ^0.75 × k^1.5 / L   (L = 0.07 × D, 수력 직경)
- ω = k^0.5 / (Cμ^0.25 × L)

### 4. 출구에서의 역류
**증상:** 잔차 진동, 로그에 음의 셀 부피
**수정:** 출구 속도에 `inletOutlet` BC 사용하거나 더 긴 출구 연장 도메인 추가.

## 5단계: 문제 셀에 대한 메시 품질 확인

```bash
# 상세 출력으로 checkMesh 실행
checkMesh -allGeometry -allTopology 2>&1 | grep -E "(high|error|Illegal|non-ortho|skew|aspect)" | head -30
```

높은 비직교성 영역 식별:

```bash
# 비직교성을 보여주는 필드 생성
postProcess -func writeCellCentres 2>/dev/null
checkMesh -writeAllFields 2>&1 | tail -20
```

불량 셀이 특정 영역에 집중된 경우 국소 메시 정제 또는 스무딩을 제안합니다.

## 6단계: 완화 계수 확인

```bash
grep -A20 "relaxationFactors\|SIMPLE\|PIMPLE" system/fvSolution
```

**완화 계수 가이드라인:**

| 변수 | 일반 범위 | 발산 시 시작값 |
|------|-----------|----------------|
| U | 0.7 | 0.5 |
| p | 0.3 | 0.2 |
| k, epsilon | 0.7 | 0.5 |
| T (온도) | 0.9 | 0.7 |

발산 중이면 완화 계수를 20-30% 줄이고 재실행합니다.

## 7단계: 이산화 기법 확인

```bash
cat system/fvSchemes
```

높은 Reynolds 수 유동에서 `linear` 보간은 진동을 유발할 수 있습니다. 더 확산적인 upwind 기법으로 전환합니다:

| 필드 | 발산 시 사용 | 안정 (정확도 낮음) |
|------|-------------|-------------------|
| div(phi,U) | `linearUpwind grad(U)` | `upwind` |
| div(phi,k) | `linearUpwind grad(k)` | `upwind` |
| laplacian | `Gauss linear limited 0.5` | `Gauss linear uncorrected` |

## 8단계: 수정 제안 및 적용

각 식별된 문제에 대해:
1. 근본 원인을 명확히 서술
2. 변경할 구체적인 줄을 표시
3. 이것이 왜 수정되는지 설명
4. 변경 전 사용자에게 확인 요청

수정 사항 적용 후:

```bash
# 시간 디렉토리 정리 후 재시작
rm -rf [1-9]*/  # 0/ 제외 모든 시간 디렉토리 삭제
echo "정리 완료. /cfd-run으로 수정 사항을 테스트하세요."
```

## 9단계: 요약

```
CFD 디버그 보고서
================
로그 파일: <경로>
증상: <발산/NaN/느린 수렴/진동>

근본 원인
  <1-2문장 설명>

발견된 문제
  1. [치명적] <문제> → <적용된 수정>
  2. [경고]   <문제> → <권장 사항>

적용된 수정
  system/controlDict: deltaT <이전> → <이후>
  system/fvSolution:  relaxU <이전> → <이후>
  0/p: 출구 BC <이전> → <이후>

판정: 수정됨 (/cfd-run 재실행) / 추가 조사 필요 / 에스컬레이션
```

세 번의 수정 시도 후에도 문제가 해결되지 않으면 중단하고 다음 내용을 포함한 상세 에스컬레이션 보고서를 제공합니다:
- 전체 잔차 이력
- 메시 품질 요약
- 모든 BC 목록
- 솔버 설정
- 도메인 전문가에게 문의하거나 문제를 재정식화할 것을 권장
