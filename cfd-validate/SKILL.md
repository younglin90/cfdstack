---
name: cfd-validate
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-validate.
  Validate CFD simulation results against reference data, analytical solutions,
  or experimental benchmarks. Computes error norms (L2, Linf), compares integral
  quantities (Cd, Cl, Nusselt number, friction factor), and generates a
  pass/fail validation report. Use when asked to "validate results", "check against
  reference", "compare with benchmark", "is my Cd correct", or "verify simulation".
  Proactively suggest after /cfd-post extracts results.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
  - WebSearch

---

# /cfd-validate — 벤치마크 대비 결과 검증

## 1단계: 검증 대상 확인

사용자에게 묻습니다:

> 무엇을 검증하시나요?
> 1. **항력/양력 계수** — 실험 또는 발표된 데이터 대비
> 2. **속도/압력 프로파일** — 해석해 대비
> 3. **마찰 계수 / 압력 강하** (파이프/채널 유동)
> 4. **열전달** (Nusselt 수)
> 5. **사용자 지정** — 참조 데이터 파일이 있음

또한: 참조 데이터 파일이 있는지, 아니면 발표된 벤치마크를 찾아볼지 확인합니다.

## 2단계: 참조 벤치마크 조회 (필요 시)

일반적인 벤치마크 케이스에 대해 알려진 참조값을 사용합니다:

### 실린더 주위 유동 (Re에 따라 다름)

| Re | Cd (참조) | Cl 진폭 (참조) | St (참조) | 출처 |
|----|----------|----------------|-----------|------|
| 20 | 2.05 ± 0.05 | — (정상) | — | Tritton 1959 |
| 40 | 1.52 ± 0.05 | — (정상) | — | Tritton 1959 |
| 100 | 1.33 ± 0.05 | 0.33 ± 0.05 | 0.166 ± 0.005 | Braza 1986 |
| 200 | 1.34 ± 0.05 | 0.68 ± 0.05 | 0.197 ± 0.005 | Braza 1986 |

### 리드 구동 공동 (벤치마크)

| Re | x=0.5에서 u_max | y=0.5에서 v_max | 출처 |
|----|----------------|----------------|------|
| 100 | y=0.4560에서 0.2166 | x=0.8080에서 0.1791 | Ghia 1982 |
| 400 | y=0.5000에서 0.4054 | x=0.8860에서 0.3050 | Ghia 1982 |
| 1000 | y=0.1736에서 0.3886 | x=0.8421에서 0.3767 | Ghia 1982 |

### 난류 파이프 유동 (Moody 선도)

완전 발달된 난류 파이프 유동:
- Blasius 마찰 계수: f = 0.316 × Re^(-0.25)  (Re < 100,000)
- Colebrook-White: 1/√f = -2.0 × log(ε/(3.7D) + 2.51/(Re√f))

### 층류 파이프 유동 (Poiseuille)

- 정확해: f = 64/Re (Darcy 마찰 계수)
- 포물선형 속도 프로파일: u(r) = 2U_mean × (1 - (r/R)²)

### NACA 0012 익형 (박익 이론)

- Cl ≈ 2π × sin(α) (작은 받음각)
- Cd (영양력) ≈ NACA 실험 데이터에서 0.008

## 3단계: 시뮬레이션 결과 추출

검증 대상에 따라:

### 힘 계수

```bash
# forceCoeffs 후처리에서 데이터 추출
DATA_FILE=$(ls postProcessing/forceCoeffs/*/coefficient.dat 2>/dev/null | head -1)
if [ -n "$DATA_FILE" ]; then
  echo "힘 계수 데이터: $DATA_FILE"
  echo "마지막 5 시간 스텝:"
  tail -5 "$DATA_FILE"
  echo ""
  echo "시간 평균 (시뮬레이션 마지막 20%):"
  python3 -c "
import numpy as np
data = np.loadtxt('$DATA_FILE', comments='#')
n = len(data)
start = int(0.8*n)
print(f'  평균 Cd = {np.mean(data[start:,1]):.4f}')
print(f'  평균 Cl = {np.mean(data[start:,3]):.4f}')
print(f'  표준편차 Cl = {np.std(data[start:,3]):.4f}')
" 2>/dev/null || echo "python3 사용 불가 — 수동으로 파싱하세요"
fi
```

### 속도 프로파일

```bash
# 단면(예: 중심선)에서 속도 샘플링
postProcess -func 'sample' 2>&1 | tail -10
ls postProcessing/sample/ 2>/dev/null
```

### 압력 강하

```bash
postProcess -func patchAverage -patch inlet  -fields '(p)' 2>&1 | grep "areaAverage"
postProcess -func patchAverage -patch outlet -fields '(p)' 2>&1 | grep "areaAverage"
```

## 4단계: 참조값과 비교

Python 검증 스크립트를 작성합니다:

```python
# cfd-validate.py — cfdstack /cfd-validate가 자동 생성
import numpy as np

# ====== 사용자: 이 값들을 수정하세요 ======
sim_Cd   = <시뮬레이션_Cd>
sim_Cl   = <시뮬레이션_Cl>
ref_Cd   = <참조_Cd>
ref_Cl   = <참조_Cl>
tol_pct  = 5.0   # 합격 허용 오차 (%)
# ==========================================

def check(label, sim, ref, tol_pct):
    err_pct = abs(sim - ref) / abs(ref) * 100
    status = "합격" if err_pct <= tol_pct else "불합격"
    print(f"  {label:20s}  시뮬레이션={sim:.4f}  참조={ref:.4f}  오차={err_pct:.1f}%  [{status}]")
    return status == "합격"

print("검증 결과")
print("=" * 60)
results = []
results.append(check("항력 계수 Cd", sim_Cd, ref_Cd, tol_pct))
results.append(check("양력 계수 Cl", sim_Cl, ref_Cl, tol_pct))

print()
if all(results):
    print("판정: 합격 — 모든 값이 허용 오차 이내")
else:
    n_fail = sum(1 for r in results if not r)
    print(f"판정: 불합격 — {n_fail}/{len(results)} 값이 허용 오차 초과")
    print("  고려 사항: 메시 정제, 실행 시간 연장, 또는 BC 수정")
```

속도 프로파일 검증의 경우 L2 및 L-infinity 오차를 계산합니다:

```python
# 속도 프로파일과 해석해 비교
sim_data = np.loadtxt('<샘플 파일>')
y_sim = sim_data[:, 1]
u_sim = sim_data[:, 3]  # 열 인덱스 조정 필요

# 해석해 (예: Poiseuille)
R = <반경>
U_mean = <평균 속도>
u_analytical = 2 * U_mean * (1 - (y_sim / R)**2)

L2_error = np.sqrt(np.mean((u_sim - u_analytical)**2)) / np.max(u_analytical)
Linf_error = np.max(np.abs(u_sim - u_analytical)) / np.max(u_analytical)
print(f"  L2 오차:   {L2_error:.4f} ({L2_error*100:.2f}%)")
print(f"  Linf 오차: {Linf_error:.4f} ({Linf_error*100:.2f}%)")
```

```bash
python3 cfd-validate.py 2>&1
```

## 5단계: 격자 수렴 연구 (선택 사항)

사용자가 여러 메시 해상도의 결과를 가지고 있는 경우 Richardson 외삽을 수행합니다:

```python
# 격자 수렴 지수 (GCI) — Celik et al. 2008
import numpy as np

# 메시 크기 (특성 셀 크기 h)
h = [<조대 h>, <중간 h>, <세밀 h>]  # 예: [0.1, 0.05, 0.025]
f = [<조대 Cd>, <중간 Cd>, <세밀 Cd>]

r21 = h[1]/h[0]
r32 = h[2]/h[1]

# 외관 차수
p_app = np.log((f[2]-f[1])/(f[1]-f[0])) / np.log(r21)

# 외삽값
f_ext = (r21**p_app * f[2] - f[1]) / (r21**p_app - 1)

# GCI (세밀 격자)
Fs = 1.25  # 안전 계수
GCI_fine = Fs * abs((f[2] - f[1]) / f[1]) / (r21**p_app - 1) * 100

print(f"외관 차수 p = {p_app:.2f}")
print(f"외삽값 = {f_ext:.4f}")
print(f"GCI (세밀 격자) = {GCI_fine:.2f}%")
print(f"격자 수렴: {'예' if GCI_fine < 5 else '아니오 — 메시 정제 필요'}")
```

## 6단계: 검증 보고서

```
검증 보고서
===========
케이스: <경로>
솔버: <solver>  물리 모델: <physics>
참조: <출처>

스칼라 값
  항력 계수 Cd:  시뮬레이션=<값>  참조=<값>  오차=<X>%  [합격/불합격]
  양력 계수 Cl:  시뮬레이션=<값>  참조=<값>  오차=<X>%  [합격/불합격]
  Strouhal 수 St: 시뮬레이션=<값>  참조=<값>  오차=<X>%  [합격/불합격]

프로파일 오차 (계산된 경우)
  속도 L2:    <X>%  [합격/불합격]
  속도 Linf:  <X>%  [합격/불합격]

격자 수렴
  외관 차수: <p>  (2차 기법에서 ~2 예상)
  세밀 격자 GCI: <X>%  [격자 수렴 / 메시 정제 필요]

판정: 검증됨 / 정제 필요 / 불합격

권장 사항
  <검증되지 않은 경우 실행 가능한 다음 단계>
```

전체 보고서 작성을 위해 `/cfd-report`를 제안합니다.
