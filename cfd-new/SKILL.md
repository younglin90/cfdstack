---
name: cfd-new
version: 1.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /cfd-new.
  Scaffold a new CFD simulation case from scratch. Asks what solver (OpenFOAM,
  SU2, etc.), physics (incompressible, compressible, heat transfer, turbulent),
  geometry, and boundary conditions — then generates the full directory structure
  with all required input files. Use when asked to "create a new CFD case",
  "set up a simulation", "start a new OpenFOAM case", or "scaffold a CFD project".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion

---

# /cfd-new — 새 CFD 케이스 스캐폴딩

## 1단계: 시뮬레이션 요구사항 수집

AskUserQuestion을 사용해 사용자에게 아래 질문을 한 번에 모두 물어봅니다:

> **새 CFD 케이스 설정**
>
> 시뮬레이션을 설명해 주세요:
> 1. **솔버** — OpenFOAM, SU2, 또는 기타? (기본값: OpenFOAM)
> 2. **물리 모델** — 비압축성 층류, 비압축성 난류(RANS/LES), 압축성, 열전달 중 선택?
> 3. **문제** — 무엇을 시뮬레이션하나요? (예: "Re=100 실린더 유동", "관 유동", "받음각 5° NACA0012 익형")
> 4. **차원** — 2D 또는 3D?
> 5. **메시** — 기존 메시가 있나요, 아니면 blockMesh / snappyHexMesh 설정을 스캐폴딩할까요?

## 2단계: 설치된 솔버 감지

```bash
# 사용 가능한 CFD 솔버 확인
echo "=== OpenFOAM ==="
if command -v icoFoam >/dev/null 2>&1; then
  echo "OpenFOAM: 발견 ($(icoFoam --version 2>&1 | head -1 || echo '버전 불명'))"
elif [ -n "$FOAM_APPBIN" ]; then
  echo "OpenFOAM: FOAM_APPBIN 경로에서 발견"
else
  echo "OpenFOAM: 없음"
fi

echo "=== SU2 ==="
if command -v SU2_CFD >/dev/null 2>&1; then
  echo "SU2: 발견 ($(SU2_CFD --version 2>&1 | head -1 || echo '버전 불명'))"
else
  echo "SU2: 없음"
fi
```

원하는 솔버를 찾을 수 없으면 사용자에게 경고하고, 솔버를 설치하지 않으면 파일은 생성되지만 실행할 수 없음을 안내합니다.

## 3단계: 케이스 이름 및 디렉토리 결정

설명에서 명확하지 않으면 사용자에게 케이스 이름을 물어봅니다. 소문자와 하이픈을 사용합니다 (예: `cylinder-re100`, `pipe-flow`, `naca0012-aoa5`).

디렉토리가 이미 존재하는지 확인합니다:

```bash
CASE_NAME="<1단계에서 얻은 케이스 이름>"
if [ -d "$CASE_NAME" ]; then
  echo "EXISTS"
else
  echo "OK"
fi
```

이미 존재하면 덮어쓸지 또는 다른 이름을 사용할지 사용자에게 물어봅니다.

## 4단계: OpenFOAM 케이스 구조 생성

OpenFOAM 케이스의 표준 디렉토리 구조를 만듭니다:

```bash
CASE="<케이스 이름>"
mkdir -p "$CASE/0" "$CASE/constant" "$CASE/system"
echo "생성됨: $CASE/0/, $CASE/constant/, $CASE/system/"
```

### 4a. 적절한 솔버 선택

물리 모델에 따라 솔버를 선택합니다:

| 물리 모델 | 권장 솔버 |
|-----------|-----------|
| 비압축성 층류 (Re < ~2000) | `icoFoam` |
| 비압축성 정상 RANS | `simpleFoam` |
| 비압축성 비정상 RANS/LES | `pimpleFoam` |
| 압축성 초음속 | `rhoCentralFoam` |
| 압축성 아음속/천음속 | `rhoSimpleFoam` |
| 부력 구동 / 열전달 | `buoyantSimpleFoam` |
| 다상 유동 (자유 표면) | `interFoam` |

### 4b. `system/controlDict` 작성

솔버 유형에 따라 적절한 기본값으로 시간 제어 파일을 작성합니다:
- 비정상 솔버: `deltaT 0.001`, `endTime 10`, `writeInterval 0.1`
- 정상 솔버: `deltaT 1`, `endTime 2000`, `writeInterval 100`

```
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      controlDict;
}

application     <솔버>;
startFrom       startTime;
startTime       0;
stopAt          endTime;
endTime         <endTime>;
deltaT          <deltaT>;
writeControl    timeStep;
writeInterval   <writeInterval>;
purgeWrite      0;
writeFormat     ascii;
writePrecision  6;
writeCompression off;
timeFormat      general;
timePrecision   6;
runTimeModifiable true;
```

### 4c. `system/fvSchemes` 작성

솔버에 맞는 이산화 기법을 선택합니다:

비압축성 (icoFoam / simpleFoam)의 경우:
```
FoamFile { version 2.0; format ascii; class dictionary; location "system"; object fvSchemes; }

ddtSchemes      { default Euler; }
gradSchemes     { default Gauss linear; grad(p) Gauss linear; }
divSchemes
{
    default         none;
    div(phi,U)      Gauss linearUpwind grad(U);
    div((nuEff*dev(T(grad(U))))) Gauss linear;
}
laplacianSchemes { default Gauss linear corrected; }
interpolationSchemes { default linear; }
snGradSchemes   { default corrected; }
```

압축성/난류 유동의 경우, 더 강건한 upwind 기법을 사용하고 난류 수송항을 추가합니다.

### 4d. `system/fvSolution` 작성

```
FoamFile { version 2.0; format ascii; class dictionary; location "system"; object fvSolution; }

solvers
{
    p
    {
        solver          GAMG;
        tolerance       1e-06;
        relTol          0.1;
        smoother        GaussSeidel;
    }
    pFinal
    {
        $p;
        relTol          0;
    }
    U
    {
        solver          smoothSolver;
        smoother        symGaussSeidel;
        tolerance       1e-05;
        relTol          0.1;
    }
    UFinal { $U; relTol 0; }
}

PISO  { nCorrectors 2; nNonOrthogonalCorrectors 0; pRefCell 0; pRefValue 0; }
SIMPLE { nNonOrthogonalCorrectors 0; }
```

### 4e. `constant/transportProperties` 작성

비압축성 뉴턴 유체의 경우:
```
FoamFile { version 2.0; format ascii; class dictionary; location "constant"; object transportProperties; }
transportModel  Newtonian;
nu              <동점성계수>;  // m^2/s (공기: 1e-5, 물: 1e-6)
```

사용자가 Reynolds 수를 제공한 경우 동점성계수를 계산합니다:
- ν = U·L / Re  (U = 유입 속도, L = 특성 길이)

### 4f. `0/` 초기 조건 파일 작성

**`0/U`** (속도장):
```
FoamFile { version 2.0; format ascii; class volVectorField; location "0"; object U; }
dimensions      [0 1 -1 0 0 0 0];
internalField   uniform (0 0 0);
boundaryField
{
    inlet   { type fixedValue; value uniform (<Ux> 0 0); }
    outlet  { type zeroGradient; }
    walls   { type noSlip; }
    // 2D 케이스:
    frontAndBack { type empty; }
}
```

**`0/p`** (압력장):
```
FoamFile { version 2.0; format ascii; class volScalarField; location "0"; object p; }
dimensions      [0 2 -2 0 0 0 0];
internalField   uniform 0;
boundaryField
{
    inlet   { type zeroGradient; }
    outlet  { type fixedValue; value uniform 0; }
    walls   { type zeroGradient; }
    frontAndBack { type empty; }
}
```

사용자가 설명한 형상에 맞게 경계 조건 이름을 조정합니다.

### 4g. `constant/polyMesh/blockMeshDict` 작성 (기존 메시가 없는 경우)

단순 박스/채널 도메인의 경우 (사용자 문제에 맞게 크기 조정):

```
FoamFile { version 2.0; format ascii; class dictionary; location "constant/polyMesh"; object blockMeshDict; }
scale 1;
vertices
(
    (0  -0.5  0)  // 0
    (10 -0.5  0)  // 1
    (10  0.5  0)  // 2
    (0   0.5  0)  // 3
    (0  -0.5  1)  // 4 (2D: 얇은 슬랩)
    (10 -0.5  1)  // 5
    (10  0.5  1)  // 6
    (0   0.5  1)  // 7
);
blocks
(
    hex (0 1 2 3 4 5 6 7) (<nx> <ny> 1) simpleGrading (1 1 1)
);
boundary
(
    inlet   { type patch; faces ((0 4 7 3)); }
    outlet  { type patch; faces ((1 2 6 5)); }
    walls   { type wall;  faces ((0 1 5 4) (3 7 6 2)); }
    frontAndBack { type empty; faces ((0 3 2 1) (4 5 6 7)); }
);
```

실린더/바디 피팅 케이스의 경우 snappyHexMesh 설정을 스캐폴딩하고, STL 형상 파일이 필요함을 안내합니다.

## 5단계: `.cfdstack` 설정 파일 작성

다른 cfdstack 스킬이 읽을 수 있도록 케이스 설정을 저장합니다:

```bash
cat > "$CASE/.cfdstack" << 'EOF'
solver=openfoam
physics=<물리모델>
dimensions=<2d|3d>
case_type=<laminar|rans|les|compressible>
EOF
echo "$CASE/.cfdstack 작성 완료"
```

## 6단계: 요약

생성된 내용을 명확하게 요약합니다:

```
케이스 생성 완료: <케이스 이름>/
├── 0/
│   ├── U            (속도: 유입=<Ux> m/s, 벽면=noSlip)
│   └── p            (압력: 유출=0 Pa)
├── constant/
│   ├── transportProperties  (nu=<값>)
│   └── polyMesh/
│       └── blockMeshDict    (도메인: <크기>, 메시: <nx>×<ny>)
└── system/
    ├── controlDict  (솔버=<solver>, endTime=<t>)
    ├── fvSchemes
    └── fvSolution
```

**다음 단계:**
1. `/cfd-check` — 메시 및 케이스 설정 검증
2. `blockMesh`로 메시를 생성한 후 `/cfd-run`으로 계산

다음 단계로 `/cfd-check`를 제안합니다.
