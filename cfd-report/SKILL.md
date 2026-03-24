---
name: cfd-report
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-report.
  Generate a professional CFD simulation report in Markdown (or PDF if pandoc
  is available). Includes case description, mesh statistics, solver settings,
  convergence history, key results, validation summary, and figures.
  Use when asked to "generate a report", "write up the simulation", "create
  simulation documentation", or "make a CFD report".
  Proactively suggest after /cfd-validate or when the simulation is complete.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion

---

# /cfd-report — 시뮬레이션 보고서 생성

## 1단계: 보고서 정보 수집

사용자에게 묻습니다:

> 시뮬레이션 보고서 정보:
> 1. **보고서 제목** — 예: "Re=100 원형 실린더 주위 유동"
> 2. **작성자 이름**
> 3. **형식** — Markdown만, 또는 PDF도 (pandoc 필요)?
> 4. **그림 포함 여부** — 예/아니오 (residuals.png, force_coefficients.png 등)

## 2단계: 케이스 메타데이터 수집

```bash
# 케이스 정보
echo "=== 케이스 디렉토리 ==="
pwd

echo "=== 솔버 ==="
grep -oP '(?<=application\s{1,20})\w+' system/controlDict 2>/dev/null | head -1

echo "=== 시뮬레이션 시간 ==="
grep -E "startTime|endTime|deltaT" system/controlDict 2>/dev/null

echo "=== cfdstack 설정 ==="
cat .cfdstack 2>/dev/null || echo "(없음)"

echo "=== 최신 시간 디렉토리 ==="
ls -d [0-9]*/ 2>/dev/null | sort -V | tail -1
```

## 3단계: 메시 통계 수집

```bash
# 메시가 있으면 checkMesh 실행
if [ -d "constant/polyMesh/faces" ]; then
  checkMesh 2>&1 | grep -E "(cells:|faces:|points:|non-ortho|skewness|Mesh OK|FAILED)" | head -20
else
  echo "메시가 아직 생성되지 않음"
fi
```

## 4단계: 수렴 데이터 수집

```bash
# 솔버 로그 찾기
LOG=$(ls *.log log.* /tmp/cfdstack-solver.log 2>/dev/null | head -1)
if [ -n "$LOG" ]; then
  echo "로그: $LOG"
  # 최종 잔차
  grep "Final residual" "$LOG" | tail -10
  # 실행 시간
  grep -E "ExecutionTime|ClockTime" "$LOG" | tail -3
fi
```

## 5단계: 결과 및 검증 데이터 수집

```bash
# 힘 계수
DATA_FILE=$(ls postProcessing/forceCoeffs/*/coefficient.dat 2>/dev/null | head -1)
if [ -n "$DATA_FILE" ]; then
  python3 -c "
import numpy as np
data = np.loadtxt('$DATA_FILE', comments='#')
n = len(data)
start = int(0.8*n)
print(f'평균 Cd = {np.mean(data[start:,1]):.4f} ± {np.std(data[start:,1]):.4f}')
print(f'평균 Cl = {np.mean(data[start:,3]):.4f} ± {np.std(data[start:,3]):.4f}')
" 2>/dev/null
fi

# 검증 결과 확인
ls cfd-validate.py 2>/dev/null && python3 cfd-validate.py 2>/dev/null || true
```

## 6단계: 사용 가능한 그림 탐색

```bash
ls *.png *.jpg 2>/dev/null
```

## 7단계: 보고서 작성

수집된 모든 정보로 `simulation_report.md`를 작성합니다:

```markdown
# <보고서 제목>

**작성자:** <이름>
**날짜:** <날짜>
**케이스 디렉토리:** `<경로>`
**솔버:** <solver>

---

## 1. 문제 설명

<물리적 문제, 형상, 목표에 대한 간략한 설명.
시뮬레이션하는 유동, Reynolds 수 / Mach 수,
주요 물리 현상을 포함합니다.>

### 1.1 형상

<계산 도메인 및 형상에 대한 설명.>

### 1.2 유동 매개변수

| 매개변수 | 값 |
|---------|-----|
| Reynolds 수 | <Re> |
| 유입 속도 | <U> m/s |
| 특성 길이 | <L> m |
| 동점성계수 | <ν> m²/s |
| 유체 | <공기/물/기타> |

---

## 2. 계산 설정

### 2.1 솔버

| 설정 | 값 |
|------|-----|
| 솔버 | <solver> |
| 시뮬레이션 유형 | <정상/비정상> |
| 시작 시간 | <t0> |
| 종료 시간 | <tf> |
| 시간 간격 Δt | <dt> |
| 쓰기 간격 | <wi> |

### 2.2 수치 기법

| 항 | 기법 |
|----|------|
| 시간 미분 | <Euler/backward/CrankNicolson> |
| 대류 (U) | <linearUpwind/upwind/linear> |
| Laplacian | <Gauss linear corrected> |
| 보간 | <linear> |

### 2.3 경계 조건

| 패치 | U | p |
|------|---|---|
| 입구 | fixedValue (<U> 0 0) | zeroGradient |
| 출구 | zeroGradient | fixedValue 0 |
| 벽면 | noSlip | zeroGradient |

---

## 3. 메시

### 3.1 메시 통계

| 속성 | 값 |
|------|-----|
| 전체 셀 수 | <N> |
| 전체 면 수 | <N> |
| 전체 점 수 | <N> |
| 최대 비직교성 | <값>° |
| 최대 왜곡도 | <값> |
| 최대 종횡비 | <값> |
| 메시 품질 | <합격/불합격> |

### 3.2 메시 품질 평가

<메시 품질에 대한 설명. 정제가 필요한 영역을 표시합니다.>

---

## 4. 수렴

### 4.1 잔차 이력

![잔차 수렴 플롯](residuals.png)

### 4.2 최종 잔차

| 변수 | 최종 잔차 | 목표 |
|------|-----------|------|
| Ux | <값> | < 1e-5 |
| Uy | <값> | < 1e-5 |
| p  | <값> | < 1e-4 |
| k  | <값> | < 1e-4 |

**수렴 상태:** <수렴 / 미수렴>

### 4.3 실행 시간

| | 값 |
|--|-----|
| 경과 시간 | <t> s |
| CPU 시간 | <t> s |
| 사용 코어 수 | <N> |

---

## 5. 결과

### 5.1 힘 계수

![힘 계수](force_coefficients.png)

| 계수 | 시뮬레이션 | 참조 | 오차 |
|------|-----------|------|------|
| Cd (평균) | <값> | <값> | <X>% |
| Cl (평균) | <값> | <값> | <X>% |
| Cl (진폭) | <값> | <값> | <X>% |
| St (Strouhal) | <값> | <값> | <X>% |

### 5.2 유동장

<주요 유동 특성 설명: 분리점, 후류 구조,
와류 이탈 주파수, 압력 분포 등.>

---

## 6. 검증

### 6.1 참조 데이터와 비교

**참조:** <인용 — 저자, 연도, 출처>

</cfd-validate의 검증 표와 설명>

### 6.2 격자 수렴

<여러 메시 해상도로 실행한 경우 GCI 결과.
해가 격자 수렴되었는지 서술합니다.>

---

## 7. 결론

<주요 결과 요약:>
- <문제>에 대한 시뮬레이션이 성공적으로 완료되었습니다.
- 계산된 항력 계수 Cd = <값>으로, 참조 데이터와 <X>% 이내로 일치합니다.
- <기타 주요 결과>

### 향후 연구 권장 사항

- <더 나은 정확도를 위해 후류 영역의 메시 정제>
- <더 높은 Reynolds 수에서 실행>
- <난류 모델 비교 포함>

---

## 부록 A: 파일 구조

```
<케이스>/
├── 0/                 # 초기 조건
├── constant/          # 메시 및 물성
├── system/            # 솔버 설정
└── postProcessing/    # 결과
```

## 부록 B: 소프트웨어

| 소프트웨어 | 버전 |
|-----------|------|
| OpenFOAM | <버전> |
| OS | <OS> |
| Python | <버전> |
| cfdstack | 1.0.0 |
```

모든 `<플레이스홀더>`를 수집된 데이터로 채워 `simulation_report.md`에 저장합니다.

## 8단계: PDF 변환 (선택 사항)

```bash
if command -v pandoc >/dev/null 2>&1; then
  pandoc simulation_report.md \
    -o simulation_report.pdf \
    --pdf-engine=pdflatex \
    -V geometry:margin=2.5cm \
    -V fontsize=11pt \
    2>&1
  echo "PDF 저장됨: simulation_report.pdf"
else
  echo "참고: pandoc 없음 — Markdown 보고서가 simulation_report.md에 저장되었습니다"
  echo "PDF 출력을 위해 pandoc을 설치하세요: https://pandoc.org/installing.html"
fi
```

## 9단계: 요약

```
보고서 생성 완료
================
  simulation_report.md  — 전체 시뮬레이션 보고서
  simulation_report.pdf — PDF 버전 (pandoc이 있는 경우)

목차:
  1. 문제 설명
  2. 계산 설정 (솔버, BC, 기법)
  3. 메시 통계
  4. 수렴 이력 + 잔차 플롯
  5. 결과 (Cd, Cl, 유동장 설명)
  6. 참조 데이터와 비교 검증
  7. 결론 및 권장 사항

공유 방법: simulation_report.md를 저장소에 커밋하거나
simulation_report.pdf를 직접 전송하세요.
```
