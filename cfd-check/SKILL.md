---
name: cfd-check
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-check.
  Validate mesh quality and CFD case setup before running a simulation.
  Runs checkMesh, validates boundary conditions, checks field dimensions,
  verifies solver settings, and flags common configuration errors.
  Use when asked to "check my case", "validate mesh", "is my setup correct",
  "check boundary conditions", or "run checkMesh".
  Proactively suggest after /cfd-new or when the user is about to run a solver.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion

---

# /cfd-check — 메시 및 케이스 설정 검증

## 1단계: CFD 케이스 위치 확인

현재 디렉토리가 CFD 케이스인지 확인하거나 사용자에게 물어봅니다:

```bash
# 케이스 유형 감지
if [ -f ".cfdstack" ]; then
  cat .cfdstack
  echo "CFDSTACK_CONFIG: 발견"
elif [ -d "0" ] && [ -d "constant" ] && [ -d "system" ]; then
  echo "OPENFOAM_CASE: 감지됨 (표준 레이아웃)"
elif [ -f "*.cfg" ] || ls *.cfg 2>/dev/null | head -1; then
  echo "SU2_CASE: 감지됨 (.cfg 파일 발견)"
else
  echo "NO_CASE_DETECTED"
fi
```

`NO_CASE_DETECTED`이면 케이스 디렉토리 경로를 지정하거나 케이스 폴더 안에서 실행하도록 안내합니다.

## 2단계: 메시 확인 (OpenFOAM)

### 2a. `checkMesh` 실행

```bash
# checkMesh 실행 후 출력 저장
checkMesh 2>&1 | tee /tmp/cfdstack-checkmesh.txt
echo "종료 코드: $?"
```

`checkMesh`를 찾을 수 없으면 OpenFOAM 환경을 확인합니다:

```bash
echo "FOAM_APPBIN: $FOAM_APPBIN"
ls "$FOAM_APPBIN/checkMesh" 2>/dev/null && echo "발견" || echo "없음"
```

OpenFOAM이 소싱되지 않은 경우 사용자에게 안내합니다:
> `source /opt/openfoam*/etc/bashrc`를 실행(또는 설치 환경에 맞는 경로)한 후 `/cfd-check`를 다시 실행하세요.

### 2b. checkMesh 출력 파싱

`/tmp/cfdstack-checkmesh.txt`를 읽어 주요 지표를 추출합니다:

```bash
grep -E "(Mesh OK|FAILED|cells:|faces:|points:|non-orthogonality|skewness|aspect ratio|ERROR|WARNING)" /tmp/cfdstack-checkmesh.txt
```

다음 지표를 표로 보고합니다:

| 지표 | 값 | 상태 |
|------|-----|------|
| 전체 셀 수 | - | - |
| 비직교성 최대값 | - | OK: < 70, 경고: 70-85, 불합격: > 85 |
| 최대 왜곡도 | - | OK: < 4, 경고: 4-8, 불합격: > 8 |
| 최대 종횡비 | - | OK: < 100, 경고: 100-1000, 불합격: > 1000 |
| 전체 메시 품질 | 합격/불합격 | - |

**임계값:**
- 비직교성 > 85: 압력-속도 연성에 강한 영향; `nNonOrthogonalCorrectors`로 완화
- 왜곡도 > 4: 수치 확산 유발 가능; 메시 정제 필요할 수 있음
- 종횡비 > 1000: 경계층에서 솔버 경직성 유발 가능

## 3단계: 디렉토리 구조 검증

```bash
# 필수 OpenFOAM 파일 존재 확인
echo "=== 디렉토리 구조 ==="
for dir in 0 constant system; do
  [ -d "$dir" ] && echo "  $dir/: OK" || echo "  $dir/: 없음"
done

echo "=== system/ 파일 ==="
for f in controlDict fvSchemes fvSolution; do
  [ -f "system/$f" ] && echo "  $f: OK" || echo "  $f: 없음"
done

echo "=== constant/ 파일 ==="
[ -f "constant/transportProperties" ] && echo "  transportProperties: OK" || echo "  transportProperties: 없음 (비압축성에 필수)"
[ -f "constant/thermophysicalProperties" ] && echo "  thermophysicalProperties: OK" || true
[ -d "constant/polyMesh" ] && echo "  polyMesh/: OK" || echo "  polyMesh/: 없음 (먼저 blockMesh 또는 snappyHexMesh 실행 필요)"
```

## 4단계: 경계 조건 검증

초기 조건 파일을 읽어 일관성을 확인합니다:

```bash
echo "=== 0/ 경계 필드 ==="
ls 0/ 2>/dev/null
echo ""
echo "=== 메시에 정의된 경계 패치 ==="
if [ -f "constant/polyMesh/boundary" ]; then
  grep -A2 "^[a-zA-Z]" constant/polyMesh/boundary | grep -E "(type|nFaces)" | head -40
fi
```

`0/`의 각 필드 파일에 대해 다음을 확인합니다:
1. 모든 메시 경계 패치가 `boundaryField`에 해당 항목이 있는지
2. 패치 이름에 오타 없는지 (흔한 오류: `inlet` vs `Inlet`)
3. 필드 유형에 맞는 차원 괄호 확인:
   - 속도 U: `[0 1 -1 0 0 0 0]`
   - 동압력 p: `[0 2 -2 0 0 0 0]`
   - 난류 운동 에너지 k: `[0 2 -2 0 0 0 0]`
   - 난류 소산율 epsilon: `[0 2 -3 0 0 0 0]`
   - 비소산율 omega: `[0 0 -1 0 0 0 0]`
   - 온도 T: `[0 0 0 1 0 0 0]`

불일치 사항은 ERROR로 보고합니다.

## 5단계: `controlDict` 검증

```bash
grep -E "(application|startTime|endTime|deltaT|writeInterval)" system/controlDict
```

다음을 확인합니다:
- `application` 필드가 설치된 솔버와 일치하는지
- `endTime > startTime`인지
- `deltaT`가 적절한지 (비정상 케이스에서 > 0.1이면 경고)
- `writeInterval`이 설정되어 있는지 (없으면 경고)

비정상 케이스의 경우 CFL 수를 추정합니다:
> CFL = U × Δt / Δx
>
> 메시 통계에서 일반적인 셀 크기 Δx를 추정합니다. 명시적 솔버에서 CFL > 1이면 경고합니다.

## 6단계: 난류 모델 확인 (해당 시)

```bash
# 난류 모델 설정 확인
if [ -f "constant/turbulenceProperties" ]; then
  cat constant/turbulenceProperties
  echo "난류: 설정됨"
elif grep -r "turbulenceModel\|RASModel\|LESModel" constant/ 2>/dev/null; then
  echo "난류: constant/에서 발견"
else
  echo "난류: 설정 없음 (층류인 경우 OK)"
fi
```

난류가 활성화된 경우 `0/`에 `k`, `epsilon`/`omega`, `nut` 필드가 존재하는지 확인합니다.

## 7단계: 요약 보고서

합격/불합격 구조의 요약을 출력합니다:

```
CFD 케이스 검사 보고서
=====================
케이스: <경로>
솔버: <solver>

메시
  전체 셀 수:       <N>
  비직교성:         <최대> (평균 <avg>)  [<OK/경고/불합격>]
  최대 왜곡도:      <값>                [<OK/경고/불합격>]
  최대 종횡비:      <값>                [<OK/경고/불합격>]
  전체 평가:        <합격/불합격>

케이스 설정
  디렉토리 구조:    <합격/불합격>
  경계 조건:        <합격/불합격> (<N>개 패치 매칭)
  필드 차원:        <합격/불합격>
  controlDict:      <합격/불합격>
  난류:             <설정됨/층류>

발견된 문제
  오류:   <오류 목록 — 실행 전 반드시 수정>
  경고:   <경고 목록 — 문제가 발생할 수 있음>

판정: 실행 준비 완료 / 오류 먼저 수정 필요
```

합격 시: 다음 단계로 `/cfd-run`을 제안합니다.
불합격 시: 가능한 경우 오류를 자동으로 수정해 드릴 것을 제안합니다 (누락된 필드, 잘못된 차원, 패치 이름 불일치 등).
