---
name: cfd-post
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-post.
  Post-process CFD simulation results. Extracts integral quantities (forces,
  mass flow, pressure drop), generates residual convergence plots, computes
  velocity/pressure statistics, and creates Python/matplotlib or ParaView scripts
  for visualization. Use when asked to "post-process results", "plot residuals",
  "extract forces", "compute drag", "visualize", or "get results".
  Proactively suggest after /cfd-run completes successfully.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion

---

# /cfd-post — 시뮬레이션 결과 후처리

## 1단계: 결과 위치 확인

```bash
# 사용 가능한 시간 디렉토리 탐색
echo "=== 시간 디렉토리 ==="
ls -d [0-9]*/ 2>/dev/null | sort -V | tail -20

echo "=== 최신 시간 ==="
ls -d [0-9]*/ 2>/dev/null | sort -V | tail -1

echo "=== 케이스 필드 ==="
ls $(ls -d [0-9]*/ 2>/dev/null | sort -V | tail -1) 2>/dev/null
```

솔버 및 물리 모델 유형을 파악하기 위해 `.cfdstack`이 있으면 읽습니다.

## 2단계: 추출할 결과 확인

사용자에게 묻거나 문맥에서 추론합니다:

> 어떤 결과가 필요하신가요?
> 1. 잔차 수렴 플롯
> 2. 힘 / 항력 / 양력 계수
> 3. 압력 강하 (입구-출구)
> 4. 단면 속도 프로파일
> 5. 위 항목 전부

## 3단계: 솔버 로그에서 잔차 파싱

솔버 로그 파일을 찾습니다:

```bash
# 솔버 로그 탐색
ls *.log log.* /tmp/cfdstack-solver.log 2>/dev/null | head -5
```

잔차 이력을 추출하고 플롯 Python 스크립트를 작성합니다:

```python
# cfd-post-residuals.py — cfdstack /cfd-post가 자동 생성
import re
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

LOG_FILE = "<로그 파일 경로>"

time_vals = []
residuals = {}

with open(LOG_FILE) as f:
    current_time = None
    for line in f:
        m = re.match(r'Time = ([0-9.e+\-]+)', line)
        if m:
            current_time = float(m.group(1))
        m = re.search(r'Solving for (\w+).*Initial residual = ([0-9.e+\-]+)', line)
        if m and current_time is not None:
            field = m.group(1)
            res = float(m.group(2))
            if field not in residuals:
                residuals[field] = ([], [])
            residuals[field][0].append(current_time)
            residuals[field][1].append(res)

fig, ax = plt.subplots(figsize=(10, 6))
for field, (times, vals) in residuals.items():
    ax.semilogy(times, vals, label=field)

ax.set_xlabel('시간 / 반복 횟수')
ax.set_ylabel('잔차')
ax.set_title('수렴 이력')
ax.legend()
ax.grid(True, which='both', alpha=0.3)
plt.tight_layout()
plt.savefig('residuals.png', dpi=150)
print("저장됨: residuals.png")
```

이 스크립트를 `<케이스>/cfd-post-residuals.py`에 저장하고 실행합니다:

```bash
python3 cfd-post-residuals.py 2>&1
```

## 4단계: 힘 계수 추출 (외부 공기역학)

### OpenFOAM의 `forceCoeffs` 함수 오브젝트 사용

힘 계수가 이미 설정되어 있는지 확인합니다:

```bash
grep -r "forceCoeffs\|forces" system/controlDict system/fvSolution 2>/dev/null | head -10
```

설정되지 않은 경우 `system/controlDict`에 `forceCoeffs` 함수 오브젝트를 추가합니다:

```
functions
{
    forceCoeffs
    {
        type            forceCoeffs;
        libs            ("libforces.so");
        writeControl    timeStep;
        writeInterval   1;
        patches         (body cylinder airfoil);   // 형상에 맞게 조정
        rho             rhoInf;
        rhoInf          1.225;       // kg/m^3 (해수면 공기)
        liftDir         (0 1 0);
        dragDir         (1 0 0);
        pitchAxis       (0 0 1);
        magUInf         <유입 속도>;
        lRef            <코드 또는 직경>;
        Aref            <기준 면적>;
        CofR            (0 0 0);
    }
}
```

그런 다음 `postProcess -func forceCoeffs`로 재실행합니다:

```bash
postProcess -func forceCoeffs 2>&1 | tail -20
```

### 힘 계수 결과 파싱

```bash
# 힘 계수 출력 탐색
ls postProcessing/forceCoeffs/0/ 2>/dev/null || ls postProcessing/forceCoeffs/ 2>/dev/null
cat postProcessing/forceCoeffs/0/coefficient.dat 2>/dev/null | tail -20
```

Cd, Cl 대 시간 플롯 스크립트를 작성합니다:

```python
# cfd-post-forces.py — cfdstack /cfd-post가 자동 생성
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

data = np.loadtxt('postProcessing/forceCoeffs/0/coefficient.dat', comments='#')
time = data[:, 0]
Cd   = data[:, 1]
Cl   = data[:, 3]

fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(10, 8), sharex=True)
ax1.plot(time, Cd, 'b-', label='Cd')
ax1.set_ylabel('항력 계수 Cd')
ax1.legend(); ax1.grid(True, alpha=0.3)
ax2.plot(time, Cl, 'r-', label='Cl')
ax2.set_ylabel('양력 계수 Cl')
ax2.set_xlabel('시간 (s)')
ax2.legend(); ax2.grid(True, alpha=0.3)
plt.suptitle('힘 계수')
plt.tight_layout()
plt.savefig('force_coefficients.png', dpi=150)

print(f"평균 Cd: {np.mean(Cd[-100:]):.4f}")
print(f"평균 Cl: {np.mean(Cl[-100:]):.4f}")
print("저장됨: force_coefficients.png")
```

## 5단계: 압력 강하 추출 (내부 유동)

파이프/채널 유동의 경우 입구 및 출구 패치에서 압력을 샘플링합니다:

```bash
postProcess -func patchAverage -patch inlet -fields '(p)' 2>&1 | tail -5
postProcess -func patchAverage -patch outlet -fields '(p)' 2>&1 | tail -5
```

또는 샘플 딕셔너리를 작성합니다:

```
// system/sampleDict
FoamFile { version 2.0; format ascii; class dictionary; object sampleDict; }
type            sets;
fields          (p U);
interpolationScheme cellPoint;
setFormat       raw;
sets
(
    lineX
    {
        type    uniform;
        axis    x;
        start   (0 0 0.5);
        end     (10 0 0.5);
        nPoints 200;
    }
);
```

```bash
postProcess -func sample 2>&1 | tail -10
```

## 6단계: ParaView 스크립트 생성

결과를 ParaView에서 열기 위한 Python 스크립트를 작성합니다 (헤드리스 렌더링):

```python
# cfd-post-paraview.py — cfdstack /cfd-post가 자동 생성
# 실행 방법: pvpython cfd-post-paraview.py
from paraview.simple import *

# OpenFOAM 케이스 로드
reader = OpenFOAMReader(FileName='<케이스>.foam')
reader.MeshRegions = ['internalMesh']
reader.CellArrays = ['U', 'p']

# 최신 시간 스텝으로 이동
animationScene = GetAnimationScene()
animationScene.GoToLast()

# 속도 크기 렌더링
renderView = GetActiveViewOrCreate('RenderView')
display = Show(reader, renderView)
ColorBy(display, ('CELLS', 'U', 'Magnitude'))
display.SetScalarBarVisibility(renderView, True)
renderView.ResetCamera()

# 스크린샷 저장
SaveScreenshot('<케이스>_velocity.png', renderView, ImageResolution=[1920, 1080])
print("저장됨: <케이스>_velocity.png")
```

```bash
# ParaView 리더를 위한 빈 .foam 파일 생성
touch <케이스이름>.foam
python3 cfd-post-paraview.py 2>&1 || echo "참고: pvpython 없음 — ParaView를 직접 열어 <케이스이름>.foam을 로드하세요"
```

## 7단계: 요약

추출된 결과를 출력합니다:

```
후처리 결과
===========
케이스: <경로>
최신 시간: <t>

수렴
  솔버:               <수렴/미수렴>
  Ux 최종 잔차:       <값>
  p  최종 잔차:       <값>
  플롯: residuals.png

힘 (계산된 경우)
  평균 Cd: <값>
  평균 Cl: <값>
  플롯: force_coefficients.png

생성된 파일
  cfd-post-residuals.py   — 잔차 플롯 스크립트
  cfd-post-forces.py      — 힘 계수 플롯 스크립트
  cfd-post-paraview.py    — ParaView 시각화 스크립트
  residuals.png           — 수렴 플롯
  force_coefficients.png  — Cd/Cl 이력
```

결과를 참조 데이터와 비교하려면 `/cfd-validate`를, 전체 보고서를 생성하려면 `/cfd-report`를 제안합니다.
