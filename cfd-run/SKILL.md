---
name: cfd-run
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-run.
  Run a CFD solver and monitor convergence. Detects the correct solver from
  controlDict, runs it, streams residuals in real time, and alerts when
  convergence is achieved or divergence begins. Supports parallel execution
  with decomposePar. Use when asked to "run the solver", "run OpenFOAM",
  "start the simulation", "run simpleFoam", or "solve the case".
  Proactively suggest after /cfd-check passes.
allowed-tools:
  - Bash
  - Read
  - Glob
  - AskUserQuestion

---

# /cfd-run — 솔버 실행 및 수렴 모니터링

## 1단계: 실행 전 점검

```bash
# OpenFOAM 케이스 디렉토리인지 확인
if [ ! -f "system/controlDict" ]; then
  echo "오류: OpenFOAM 케이스 디렉토리가 아닙니다 (system/controlDict 없음)"
  exit 1
fi

# controlDict에서 솔버 감지
SOLVER=$(grep -oP '(?<=application\s{1,20})\w+' system/controlDict | head -1)
echo "솔버: $SOLVER"

# 메시 존재 확인
if [ ! -d "constant/polyMesh/faces" ] && [ ! -d "constant/polyMesh" ]; then
  echo "메시: 없음 — 먼저 blockMesh 또는 snappyHexMesh를 실행하세요"
else
  echo "메시: OK"
fi

# 초기 조건 존재 확인
ls 0/ 2>/dev/null && echo "초기 조건: OK" || echo "초기 조건: 없음"

# 솔버 바이너리 확인
command -v "$SOLVER" >/dev/null 2>&1 && echo "솔버 바이너리: 발견" || echo "솔버 바이너리: 없음"
```

메시 또는 초기 조건이 없으면 중단하고 먼저 `/cfd-new` 또는 `/cfd-check`를 실행하도록 안내합니다.

## 2단계: 필요 시 메시 생성

`constant/polyMesh/faces`가 없지만 `blockMeshDict`가 있는 경우:

```bash
if [ ! -f "constant/polyMesh/faces" ] && [ -f "constant/polyMesh/blockMeshDict" ]; then
  echo "blockMesh 실행 중..."
  blockMesh 2>&1 | tail -20
  echo "blockMesh 종료 코드: $?"
fi
```

snappyHexMesh 케이스의 경우, 형상 파일이 필요하므로 수동 또는 설정 스크립트로 실행해야 함을 안내합니다.

## 3단계: 병렬 실행 여부 확인

사용자에게 물어봅니다:
- 사용 가능한 / 원하는 CPU 코어 수는?
- 1 초과: `decomposePar` + `mpirun -np N <solver> -parallel` 사용
- 1 (기본값): 직렬 실행

```bash
# 사용 가능한 CPU 코어 감지
nproc 2>/dev/null || sysctl -n hw.ncpu 2>/dev/null || echo "알 수 없음"
```

병렬 실행을 원하면 `system/decomposeParDict`를 생성합니다:

```
FoamFile { version 2.0; format ascii; class dictionary; location "system"; object decomposeParDict; }
numberOfSubdomains  <N>;
method              scotch;
```

그런 다음 실행합니다:
```bash
decomposePar 2>&1 | tail -10
echo "decomposePar 종료 코드: $?"
```

## 4단계: 솔버 실행

### 직렬 실행

```bash
SOLVER=$(grep -oP '(?<=application\s{1,20})\w+' system/controlDict | head -1)
echo "$SOLVER 시작 (직렬)..."
$SOLVER 2>&1 | tee /tmp/cfdstack-solver.log
echo "솔버 종료 코드: $?"
```

### 병렬 실행

```bash
SOLVER=$(grep -oP '(?<=application\s{1,20})\w+' system/controlDict | head -1)
N_CORES=<N>
echo "$SOLVER를 $N_CORES 코어로 시작..."
mpirun -np $N_CORES $SOLVER -parallel 2>&1 | tee /tmp/cfdstack-solver.log
echo "솔버 종료 코드: $?"
```

## 5단계: 실시간 잔차 모니터링

솔버 실행 중 주기적으로 로그에서 수렴 상태를 확인합니다:

```bash
# 로그에서 최신 잔차 추출
tail -100 /tmp/cfdstack-solver.log | grep -E "(Time =|Solving for|SIMPLE|PISO|Final residual|converged|DICPCG|GAMG)" | tail -30
```

각 확인 시 수렴 표를 보고합니다:

```
Time = <t>
  Ux:  초기=<r0>  최종=<rf>  반복=<N>
  Uy:  초기=<r0>  최종=<rf>  반복=<N>
  p:   초기=<r0>  최종=<rf>  반복=<N>
  k:   초기=<r0>  최종=<rf>  반복=<N>    (난류인 경우)
  eps: 초기=<r0>  최종=<rf>  반복=<N>    (난류인 경우)
```

### 수렴 기준

| 잔차 | 수렴 | 허용 | 문제 |
|------|------|------|------|
| 속도 (U) | < 1e-5 | < 1e-4 | > 1e-3 |
| 압력 (p) | < 1e-4 | < 1e-3 | > 1e-2 |
| k, epsilon | < 1e-4 | < 1e-3 | > 1e-2 |

다음 경우 즉시 사용자에게 경고합니다:
- 어떤 잔차가 10회 이상 단조 증가하는 경우 (발산)
- 잔차가 1.0을 초과하는 경우 (폭발)
- 솔버가 `FOAM FATAL ERROR` 또는 `Floating point exception`을 출력하는 경우

## 6단계: 완료 확인

솔버 종료 후:

```bash
# 최종 시간 디렉토리 확인
ENDTIME=$(grep -oP '(?<=endTime\s{1,10})[0-9.]+' system/controlDict | head -1)
ls -d "$ENDTIME" 2>/dev/null && echo "최종 시간 디렉토리: OK" || echo "최종 시간 디렉토리: 없음"

# 최종 잔차 표시
grep "Final residual" /tmp/cfdstack-solver.log | tail -10

# 오류 확인
grep -i "error\|fatal\|floating point" /tmp/cfdstack-solver.log | tail -5
```

## 7단계: 실행 후 요약

```
실행 요약
=========
솔버:      <solver>
모드:      직렬 / 병렬 (<N> 코어)
종료 시간: <t>
종료 코드: <0=OK, 0 아님=오류>

수렴 상태
  Ux 최종 잔차:  <값>  [수렴 / 미수렴]
  p  최종 잔차:  <값>  [수렴 / 미수렴]
  전체:          수렴 / 발산 / 실행 중

다음 단계
```

수렴한 경우: `/cfd-post` 및 `/cfd-validate`를 제안합니다.
발산하거나 미수렴한 경우: `/cfd-debug`를 제안합니다.
아직 실행 중인 경우 (백그라운드): `/tmp/cfdstack-solver.log`에서 로그 파일을 확인하도록 안내합니다.

## 8단계: 병렬 케이스 재조합 (병렬인 경우)

병렬로 실행하고 수렴한 경우:

```bash
reconstructPar 2>&1 | tail -10
echo "reconstructPar 종료 코드: $?"
```
