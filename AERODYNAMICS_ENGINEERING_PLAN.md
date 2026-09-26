# Scimitar 공력해석 및 동역학 모델 구축 실행 계획

> 상태: 설계·조사 문서. CFD, VSPAERO, AVL 및 비행 시뮬레이션은 아직 실행하지 않았다.
> 실행 환경: Ubuntu. 아래 수치 범위와 합격 기준은 **초기 엔지니어링 제안**이며 실기체 사양이나 검증 결과가 아니다.

[프로젝트 README](readme.md) · [전체 개발 계획](SCIMITAR_GAZEBO_DIGITAL_TWIN_PLAN.md)

## 1. 이번 계획에서 결정하는 것

공력 작업의 첫 산출물은 그림이 아니라, 출처와 적용 범위를 가진 **힘·모멘트 계수 데이터베이스**다. 이후 같은 데이터를 오프라인 6-DoF 계산과 Gazebo에 넣어 결과가 일치하는지 확인한다.

```text
원본 CFD 자료 감사 ─────────────────────────────┐
CAD → 좌표계·기준 면적·단면·조종면 → VSPAERO / AVL ├→ 계수 DB
                    └→ 단면 점성 추정 / 선택 CFD ┘     ↓
실측 질량·CG·추력·서보 → 트림 계산 → Gazebo 계수 매핑 → PX4 SITL
                                                  ↓
                                      실측 / 비행 식별로 모델 보정
```

권장 경로는 다음과 같다.

1. **VSPAERO를 주 해석 후보**로 삼아 정상 비행 영역의 양력·유도항력·모멘트·조종면 효과를 구한다.
2. **AVL은 단순화된 동일 형상의 교차 확인 및 안정미계수 계산 후보**로 사용한다. 두 도구의 결과를 근거 없이 평균하지 않는다.
3. 단면 점성항력과 천이 민감도는 **XFOIL 및 자료 조사**, 전체 기체 항력·간섭·박리 민감 조건은 **선택적인 3D RANS CFD**로 보완한다.
4. **Gazebo AdvancedLiftDrag**로 정상 비행 범위를 먼저 표현한다. 요구 오차를 만족하지 못하는 비선형 효과가 확인되면 테이블 기반 사용자 플러그인으로 넘어간다.
5. 계수 생성, Gazebo 구현 검증, 실기체 검증을 별도 단계로 관리한다.

VSPAERO는 potential-flow 기반 도구이며 버전별 기능·입력 형식 차이가 있다. Ubuntu에서 버전과 예제 재현성을 확인한 후 solver 및 Python API를 고정한다. [NASA VSPAERO 안내](https://www.nasa.gov/reference/openvsp-vspaero-basics/)

## 2. 원본 자료에서 실제로 확인한 내용

확인 대상 revision: `Oscilous/Adapted-Modular-Scimitar@24a5d5986ae11a63882004382576460844760700`.
이번 확인은 일부 텍스트 파일의 사전 조사다. CAD 조립 확인, 전체 로그·바이너리 결과 해석과 데이터 감사는 아직 남아 있다.

| 파일 | 이번에 확인한 사실 | 계획에 미치는 영향 |
|---|---|---|
| `CFD/project_results.xml` | 89 bytes. `ProjectResults` 아래 빈 `FlowmasterData`만 있음 | 이 파일에서 CL/CD/Cm polar를 추출한다는 전제를 폐기 |
| `CFD/Internal_M5GO.info.json` | `ScimitarV5CFD.SLDASM`, `Project(36)`, `finished: true` | STEP와 동일 configuration인지 추가 확인 |
| 같은 JSON | `cells_total: 249712`, iteration 174, 여섯 개 goal 요약 | 단일 계산 요약으로 취급. 격자 독립성이나 sweep 증거로 취급하지 않음 |
| 같은 JSON | 압력·밀도·Y/Z 평균 속도·Y/Z Normal Force goals | 평균 속도를 freestream, Normal Force를 총 lift/drag로 바로 해석하지 않음 |
| 같은 JSON | external, steady, `laminar and turbulent`, SOLIDWORKS Flow Simulation 2023 | 원본 재현에는 해당 solver의 export가 필요할 수 있음 |
| 같은 JSON | default fluids에 Water와 Air가 함께 기재됨 | 실제 외부 유체 선택과 물성·경계조건을 config에서 확인 |
| `CFD/Internal_M5GO.xmlconfig` | Python 표준 XML parser로 읽을 때 3264행의 attribute 내부 `<` 때문에 실패 | 원본 보존 후 복사본의 해당 문법 문제를 명시적으로 처리하거나 native export 사용 |
| `PX4 Firmware/Parameters.params` | header: PX4 1.17.0 beta, Git Revision `15f5fefa0c000000` | SITL 버전과 parameter migration 검토 필요 |
| 같은 params | `FW_AIRSPD_MIN=15`, `TRIM=17`, `MAX=25`, `STALL=13` | 해석 속도 후보를 잡는 참고 설정. 실제 실속속도·검증된 운용한계가 아님 |
| `PX4 Firmware/readme.txt` | 구성 참고용이며 controller tune은 새로 수행하라고 명시 | 원본 tune의 일괄 재사용 금지 |

근거: [results XML](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/CFD/project_results.xml), [CFD JSON](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/CFD/Internal_M5GO.info.json), [CFD config](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/CFD/Internal_M5GO.xmlconfig), [PX4 params](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/PX4%20Firmware/Parameters.params), [PX4 설명](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/PX4%20Firmware/readme.txt).

### 원본 CFD 복원 작업

- config에서 외부 유속 벡터, 밀도·점성·온도, 길이·힘 단위, 좌표계, 계산 영역, 유체 선택, 물체 선택, 경계조건과 수렴 설정을 추출한다.
- `.stdout`, `.log`에서 goal 이력과 종료 사유를 확인한다. `finished=true`만으로 수렴했다고 판정하지 않는다.
- `.fld` 등 바이너리 파일은 이름만으로 포맷을 추정하지 않는다. 읽을 수 없다면 원 solver에서 force/moment history, surface pressure, CSV를 export하는 작업으로 남긴다.
- configuration, 받음각, 조종면 각도, 프로펠러 상태, 기준점, 전체 힘/모멘트 여부를 연결할 수 있는 case만 비교 자료로 승격한다.
- 동일 단위·좌표계로 변환할 근거가 없으면 raw 자료로 보존하고 계수 fitting에서 제외한다.

**판정:** 현재 확인한 자료만으로 6축 공력계수 DB를 구성할 수 없다. 다른 로그·export의 유효 데이터 복원 가능성을 확인한 뒤 신규 해석 범위를 확정한다.

## 3. CAD에서 해석용 형상을 만드는 방법

### 3.1 기준 configuration 고정

`CAD/001_Scimitar_V2.STEP`와 개별 부품 STEP를 비교해 assembly 포함 범위·중복·placement를 확인한다. CFD의 V5 이름과 CAD의 V2 이름이 다르므로 동일 형상이라고 가정하지 않는다.

첫 구성은 제작 예정 canopy, 센서 grid, 배터리·payload 배치를 포함해 ID를 부여한다. 모터 정지/회전, prop 장착/미장착, elevon 중립각도 configuration에 포함한다.

`geometry_manifest.yaml`에 다음을 기록한다.

- 원본 commit 및 STEP SHA-256
- 사용 부품과 제외 부품, 단위, CAD→해석 좌표 변환
- 외부 형상 변경 여부, canopy/grid/payload configuration
- 공력 외형, 질량 모델, visual/collision mesh 각각의 생성 방법

### 3.2 추출·단순화 작업

Ubuntu에서 OpenCascade 계열 도구(FreeCAD Python 또는 pythonOCC 중 설치·재현성이 좋은 하나)를 선정한다.

1. STEP의 bounding box, assembly transform, solid/shell 구성을 조사한다. 원본이 mm인지 m인지 실제 치수 근거로 확인한다.
2. root·tip 및 taper/sweep 변화 지점에서 단면을 추출한다. 추가 span station은 곡률과 형상 변화에 따라 배치한다.
3. leading/trailing edge, chord, twist, sweep, dihedral, airfoil 좌표를 추출한다. winglet이나 비평면 형상이 있으면 별도 lifting surface로 보존한다.
4. elevon span 범위, chord 비율, hinge 선의 두 끝점과 회전축, 실제 중립각을 추출한다.
5. 냉각 유입구·grid·canopy처럼 외부 유동에 영향을 줄 구조와 내부 나사·전자부품처럼 제거 가능한 구조를 구분한다. 제거 목록을 기록한다.
6. OpenVSP lifting-surface 형상을 재구성하고 CAD와 단면 overlay를 비교한다. STEP를 mesh로 바꿨다는 이유만으로 올바른 VSPAERO 형상이라고 간주하지 않는다.
7. 전기체 RANS용 외피는 watertight 여부, normal, self-intersection, 좁은 gap, trailing-edge 두께를 별도로 검사한다.

### 3.3 공력 기준값

`S_ref`는 양쪽 날개를 포함한 정해진 평면 투영 면적, `b_ref`는 기준 span, `c_ref`는 MAC로 정한다. 중심부 포함 여부와 winglet 처리 규칙을 고정한다. 단면별 wetted area와 혼용하지 않는다.

```text
S_ref = ∫ c(y) dy                # 정의한 기준 평면에서 전 span 적분
MAC   = (1 / S_ref) ∫ c(y)^2 dy
AR    = b_ref^2 / S_ref
Re(y) = rho * V * c(y) / mu
Mach  = V / a
```

MAC 위치도 함께 계산한다. 순항점만이 아니라 root/tip chord에서의 Reynolds 수 범위를 기록한다.

초기 형상 전사 기준 제안: span·S_ref·MAC 오차 각각 0.5% 이하, 주요 단면 RMS 오차 0.5% chord 이하. hinge 위치 오차는 1 mm 또는 0.5% local chord 중 더 엄격한 기준을 목표로 하되 CAD 정밀도에 맞춰 착수 전에 확정한다. 의도적으로 생략한 형상은 수치 오차와 별도로 기록한다.

산출물: `geometry_manifest.yaml`, `reference_geometry.yaml`, `sections/*.dat`, `scimitar.vsp3`, `geometry_comparison.md`.

## 4. 좌표계·힘·모멘트 계약

분석용 표준 body frame은 FRD(x 전방, y 오른쪽, z 아래)로 둔다. 모멘트와 각속도는 오른손 법칙이다. CAD, 각 solver, Gazebo link, PX4 사이의 회전·이동을 `FRAME_CONVENTIONS.md`에 수치 행렬로 기록한다.

분석 DB는 body-frame 힘 계수 `CX,CY,CZ`와 body-frame 모멘트 계수 `Cl,Cm,Cn`을 기본으로 저장한다. `CL,CD`는 명시한 wind frame으로 변환해 함께 저장한다. `CL`은 양력, `Cl`은 roll moment이며 대소문자를 구분한다.

```text
V_air_body = R_world_to_body * (V_vehicle_world - V_wind_world)
V         = |V_air_body|
alpha     = atan2(w, u)
beta      = atan2(v, sqrt(u^2 + w^2))
qbar      = 0.5 * rho * V^2
F_body    = qbar * S_ref * [CX, CY, CZ]
M_ref     = qbar * S_ref * [b_ref*Cl, c_ref*Cm, b_ref*Cn]
p_hat     = p * b_ref / (2*V)
q_hat     = q * c_ref / (2*V)
r_hat     = r * b_ref / (2*V)
M_CG      = M_ref + (r_ref - r_CG) × F_body
```

`q`(pitch rate)와 `qbar`(동압)를 변수명에서도 구별한다. 정지 부근에서는 rate normalization을 직접 나누지 않고 유효 속도 하한과 모델 적용범위를 정의한다.

공력 해석의 moment reference는 고정된 기하 기준점으로 두고, 배터리 위치 등으로 CG가 바뀔 때 위 식으로 이동한다. 동일 힘의 lever-arm moment를 solver와 Gazebo에서 중복 더하지 않는다.

### elevon 부호

좌우 모두 trailing edge down을 공통 양(+)의 기하 편향으로 정의한다.

```text
delta_sym  = (delta_L + delta_R) / 2
delta_diff = (delta_R - delta_L) / 2
delta_L    = delta_sym - delta_diff
delta_R    = delta_sym + delta_diff
```

이 정의 자체가 pitch-up/roll-right 명령을 뜻하지는 않는다. CAD hinge 축, solver control 부호, SDF joint 부호, PX4 allocator 부호를 각각 확인한다. 좌우 1° 단독 입력의 힘·모멘트 방향으로 검증한다.

## 5. 도구별 역할과 채택 기준

| 방법 | 이번 프로젝트에서 얻을 것 | 맡기지 않을 판단 |
|---|---|---|
| CAD / 단면 추출 | 형상·기준값·조종면 축 | 얇은 출력물을 solid 재질로 간주한 실제 질량 |
| VSPAERO | 정상 비행 영역의 3D 하중, 유도항력, 조종면·안정성 영향 | 단독 결과로 실제 박리·실속·표면 거칠기 확정 |
| AVL | 같은 단순 형상의 도함수·트림·모드 교차 확인 | 높은 받음각이나 큰 박리의 실측 대체 |
| XFOIL | 실제 추출 단면의 Re별 점성 polar 및 천이 민감도 | 2D CLmax를 그대로 전기체 CLmax로 사용 |
| 3D RANS / 필요 시 URANS | 점성항력, canopy/grid 간섭, 고받음각·큰 조종면 편향의 추가 조사 | 격자 독립성만으로 실기체 검증 완료 선언 |
| 실측 / 비행 식별 | 질량·CG·추력·서보 및 실제 동역학 보정 | 데이터가 여기하지 못한 계수의 독립 식별 |

AVL의 lifting-surface 및 준정상 해석 범위는 [MIT AVL primer](https://web.mit.edu/drela/Public/web/avl/AVL_User_Primer.pdf)를 기준으로 제한한다. XFOIL의 기능과 제한된 박리 처리는 [MIT XFOIL 문서](https://web.mit.edu/drela/Public/web/xfoil/)를 참고한다.

VSPAERO 설치·API 자동화가 첫 pilot case에서 막히면 AVL로 같은 형상의 정상 비행 모델을 먼저 만든다. 3D CFD는 Ubuntu에서 OpenFOAM을 기본 후보로 하되 배포판과 release를 한 가지로 고정한다. upstream native solver 재현은 라이선스·Windows 환경 확보 여부에 따른 보조 경로다.

## 6. 계산 case matrix

다음은 초기 후보다. 실제 hinge 가동범위·형상·트림 결과를 확인한 후 `cases.csv`로 확정한다. 범위 밖 해석을 수행했다는 사실이 그 영역의 물리적 타당성을 보장하지 않는다.

| 묶음 | 후보 조건 | 목적 / 처리 |
|---|---|---|
| Pilot | V=17 m/s, alpha=0°, beta=0°, 양쪽 elevon=0° | 파이프라인·축·단위 점검. 17은 upstream 설정 참고값 |
| Baseline | alpha=-6°~12° / 2° 간격; V=15,17,21,25 m/s | CL/CD/Cm 및 Re 영향. attached-flow 범위만 V0 fitting에 사용 |
| Symmetric elevon | alpha=-2°,4°,8°; delta_sym=-10,-5,0,5,10° | pitch/lift 효과, 양력·항력 연동 |
| Differential elevon | 같은 alpha; delta_diff=-10,-5,0,5,10° | roll 효과와 adverse yaw, pitch coupling |
| Sideslip | alpha=-2°,4°,8°; beta=-6,-3,0,3,6° | CY_beta, Cl_beta, Cn_beta |
| Mixed controls | alpha=4°; delta_sym와 delta_diff 각각 -5,0,5° | 좌우 조종면 선형 중첩 오차 |
| Rate derivatives | 3개 정상 비행점; p_hat/q_hat/r_hat 각각 ±0.01, ±0.02 단독 | 감쇠미계수와 perturbation 크기 민감도 |
| Envelope extension | 고받음각, beta 최대 ±15°, 편향 최대 ±20°는 필요 조건만 | baseline 이후 CFD/실측 근거가 있는 영역만 확장 |

Baseline 외 묶음은 우선 17 m/s에서 수행하고, 민감한 경우에만 속도 축을 확장한다. 중복 case는 동일 ID로 재사용한다. 모든 축의 Cartesian product를 처음부터 돌리지 않는다. pilot 후 유효 case 수와 CPU 시간·메모리를 측정해 batch 규모를 정한다.

고받음각 탐색은 우선 2° 간격, 하중의 비선형 변화가 나타난 구간은 0.5°~1° 간격으로 좁힌다. potential-flow 결과가 계속 선형으로 증가해도 이를 실속이 없다는 근거로 쓰지 않는다.

### 도함수 계산

solver가 제공하는 analytic/stability derivative와 중앙차분을 비교한다.

```text
C_alpha ≈ [C(alpha0+h) - C(alpha0-h)] / (2*h)  # h: rad
C_delta ≈ [C(delta0+h) - C(delta0-h)] / (2*h)  # h: rad
C_p     ≈ [C(p_hat=+h) - C(p_hat=-h)] / (2*h)
```

각도 h는 0.5°, 1°, 2° 후보를 rad로 변환한다. 반으로 줄였을 때 값이 크게 변하면 비선형성·격자 해상도·수렴 오차를 조사한다. rate derivative는 정적 alpha sweep만으로 얻었다고 주장하지 않는다. solver의 회전율 해석 또는 시간응답 자료가 필요하다.

`Cl_p`, `Cm_q`, `Cn_r` 같은 감쇠 항뿐 아니라 `Cl_r`, `Cn_p`, 조종면의 yaw/pitch coupling을 확인한다. 대칭 형상에서 예상되는 홀/짝 대칭은 수치 점검에 사용하되 비대칭 payload·prop 효과까지 0으로 강제하지 않는다.

## 7. 항력·실속·표면 상태 보완

항력은 먼저 구성 항목을 명시한다.

```text
CD_total = CD_induced + CD_profile + CD_body_and_appendages + CD_interference
```

이는 관리용 분해다. RANS의 total drag에는 여러 항이 이미 포함되므로 위 항을 다시 전부 더하지 않는다. solver 출력이 induced-only인지 total-estimate인지 result metadata에 기록한다.

- 추출 단면별 Re와 Mach에서 XFOIL polar를 구한다. 수렴 실패점은 결측으로 남긴다.
- free-transition과 강제 천이 조건을 비교한다. 출력물 거칠기·이음매·hinge gap은 실물 상태가 확인되기 전까지 불확실성 항목이다.
- strip 적분을 사용하면 local chord, local effective alpha와 section Reynolds 수를 적용한다. 같은 profile drag를 VSPAERO와 후처리 양쪽에 넣지 않는다.
- canopy/grid/payload의 증분은 동일 조건의 형상 A/B 비교로 구한다. 가능하면 공통 mesh 규칙을 사용해 차분 오차를 줄인다.
- `CD0 + k*CL²`는 정상 비행의 제한된 범위에서만 fit한다. elevon drag나 비선형 구간까지 한 포물선으로 강제하지 않는다.
- 실제 CLmax·stall alpha·post-stall Cm가 없으면 실속·착륙 flare 결과에 낮은 신뢰도를 표시한다. 단순 외삽 계수는 ASSUMED로 관리한다.

## 8. 선택적인 3D CFD 실행 절차

### 8.1 추가 CFD를 시작하는 조건

다음 중 하나가 충족되면 해당 조건을 추가 계산한다.

- upstream에서 total force/moment와 기준 조건을 복원할 수 없음
- 정상 비행 trim이 후보 CG·질량·추력 범위에서 성립하지 않음
- VSPAERO/AVL 차이가 기준 통일·panel 수렴 후에도 해결되지 않음
- canopy/grid 또는 큰 elevon 편향의 간섭이 비행 성능을 좌우함
- 예정 mission이 attached-flow 모델의 유효 범위를 벗어남

### 8.2 Pilot case 준비

1. full-aircraft watertight 표면과 patch 목록을 만든다. 모터-off baseline의 prop 상태를 정의한다.
2. 속도·밀도·동점성/점성·온도·Mach/Re를 고정한다. Mach가 충분히 낮은 경우(초기 후보 기준 0.3 미만) 비압축성 steady RANS부터 검토한다.
3. k-omega SST를 초기 난류모델 후보로 두고, 천이/laminar bubble 민감도가 큰 Re 영역에서는 fully turbulent 결과의 한계를 표시하고 천이·trip 민감도를 비교한다.
4. 계산 영역은 기체로부터 upstream 10 MAC, downstream 20 MAC, 측면·상하 max(10 MAC, 2 span)를 초기 후보로 잡고 영역 확장 시험으로 결정한다.
5. 유입 속도 벡터와 far-field 경계조건을 alpha/beta와 일치시킨다. 벽면 no-slip, 출구 압력 기준, 압력의 차원/kinematic 여부를 solver 규약에 맞춘다.
6. 처음에는 beta=0, 대칭 조종면·대칭 형상의 경우만 반기체 symmetry를 검토한다. beta·차동 조종면·비대칭 형상·회전 prop case는 full domain을 사용한다.

### 8.3 격자와 수렴

- leading/trailing edge, tip, hinge/gap, canopy/grid와 wake에 refinement를 둔다.
- 벽 해상형 SST 경로에서는 y+ 약 1 이하를 목표로 첫 층 높이를 추정하고 계산 후 분포를 확인한다. layer 수는 경계층 두께를 덮도록 정하며 초기 growth ratio는 1.2 이하 후보로 둔다.
- wall-function 경로를 쓰면 그 모델의 허용 y+ 영역을 별도로 만족시킨다. 두 경로의 설정과 결과를 섞지 않는다.
- coarse/medium/fine 세 격자를 체계적으로 만든다. 대표 cell 수만 바꾸는 대신 표면·벽층·wake 해상도의 refinement 규칙을 기록한다.
- residual 감소, 연속성·질량 불균형, CL/CD/Cm 이력, 표면 압력과 y+를 함께 본다. 마지막 residual 하나만으로 판정하지 않는다.
- mesh quality 실패나 힘의 추세가 남으면 case를 FAILED/UNCONVERGED로 저장하고 fitting에 사용하지 않는다.

초기 수렴 기준 제안:

| 점검 | Pilot 이후 확정할 후보 기준 |
|---|---|
| 반복 수렴 | residual 최소 3 orders 감소를 진단값으로 사용; 힘·모멘트 최종 두 window 평균 차이 1% 이하 |
| 질량 불균형 | 총 유입 질량 유량 기준 0.1% 이하 목표 |
| 중간→미세 격자 | CL/Cm 변화 2%, CD 변화 5% 이하 목표; 0 근처는 절대 오차 병기 |
| 계산 영역 확대 | 주요 계수 변화 1% 이하 목표 |
| panel solver 해상도 | 정상 영역 주요 계수·도함수 변화 2% 이하 목표 |

상대오차의 분모가 작은 계수는 `max(abs(C), C_floor)`를 사용한다. 초기 floor 후보는 force coefficient 0.05, moment coefficient 0.01이며 raw 절대차도 함께 제출한다. 이 수치는 프로젝트 제안이며 보편적 보증 기준이 아니다. 비단조 수렴에는 단순 Richardson extrapolation을 강제하지 않는다. 가능하면 관측 차수와 GCI를 보고한다.

수렴·격자·시간해상도 확인과 실험 비교의 역할은 [NASA CFD verification/validation 지침](https://www.grc.nasa.gov/www/wind/valid/tutorial/tutorial.html)을 따른다.

### 8.4 정상해가 없을 때

steady RANS가 진동하면 단순히 iteration을 늘려 마지막 값을 채택하지 않는다. mesh/BC 문제와 물리적 비정상성을 구분한 뒤 필요 시 URANS로 전환한다.

- Δt를 바꾸며 force/moment 평균·RMS의 수렴을 확인한다.
- 초기 transient를 제거하고 여러 convective time `MAC/V`와 저주파 주기를 포함하도록 averaging window를 늘린다.
- 시간 평균뿐 아니라 RMS, window 길이, sampling 간격을 저장한다.
- URANS 평균계수로 동적 실속·히스테리시스 전체가 모델링됐다고 간주하지 않는다.

### 8.5 첫 CFD batch

먼저 한 정상 비행 후보점에서 격자·영역·수렴 절차를 검증한다. 그다음 nominal trim 근처, 높은 양력 요구점, 큰 대칭 elevon, 필요 시 canopy/grid A/B 비교의 소수 조건을 계산한다. high-alpha와 prop-on 계산은 그 결과에 따라 추가한다.

`forceCoeffs` 같은 후처리에서는 힘 방향과 reference point를 명시한다. roll/yaw는 span, pitch는 MAC 기준이므로 공통 `lRef`로 나온 계수를 그대로 혼용하지 않고 dimensional moment에서 재정규화한다. [OpenFOAM forceCoeffs source guide](https://cpp.openfoam.org/v12/classFoam_1_1functionObjects_1_1forceCoeffs.html)

## 9. 계수 데이터베이스와 fitting

### 9.1 저장 계약

각 run은 독립 디렉터리에 저장하고 solver가 이전 결과를 덮어쓰지 않게 한다.

```text
simulation/aero/
  geometry/                # 추출 단면 / VSP / AVL 입력
  cases/cases.csv          # 실행할 조건과 configuration
  runs/<case_id>/           # input, log, raw result, convergence
  processed/polars.csv     # 정규화한 힘·모멘트와 품질 flag
  processed/derivatives.csv
  coefficients.yaml       # Gazebo/오프라인 모델로 내보낼 값
  validation/             # 수렴·보간·holdout 비교
```

필수 case fields:

```text
case_id, geometry_hash, configuration_id, solver_name, solver_version,
alpha_deg, beta_deg, delta_L_deg, delta_R_deg, V_mps, rho_kgm3, mu_Pas,
p_hat, q_hat, r_hat, S_ref_m2, b_ref_m, c_ref_m, moment_ref_xyz_m,
force_frame, moment_frame, Fx_N, Fy_N, Fz_N, Mx_Nm, My_Nm, Mz_Nm,
CX, CY, CZ, CL, CD, Cl, Cm, Cn, convergence_status, source_type,
mesh_id, quality_flags, valid_domain, source_ref
```

단위가 없는 수치 열을 만들지 않는다. `source_type`은 UPSTREAM/CAD_DERIVED/SIMULATION_DERIVED/MEASURED/ASSUMED를 사용한다. UNKNOWN은 값의 상태이며 출처 종류와 분리한다. 새로운 solver 결과에는 검증 여부와 불확실성을 별도로 기록한다.

### 9.2 정상 영역 모델

```text
Ci = Ci0 + Ci_alpha*alpha + Ci_beta*beta
   + Ci_p*p_hat + Ci_q*q_hat + Ci_r*r_hat
   + Ci_L*delta_L + Ci_R*delta_R
```

이를 초기 선형 모델로 사용하되 drag에는 필요한 이차항, 조종면/alpha coupling에는 근거가 있는 항만 추가한다. fit의 각도 단위는 rad로 통일한다. fit 원점·trim bias·CG 기준점도 metadata에 둔다.

- train과 holdout case를 분리한다. fitting에 사용하지 않은 alpha/beta/편향 중간점으로 보간 오차를 검사한다.
- 제로 근처 coefficient는 상대오차만 보고하지 않는다. absolute error와 전 운용 범위 scale로 정규화한 오차를 함께 쓴다.
- solver끼리 비교하기 전에 reference area/length, moment reference, 좌표계, viscous drag 포함 범위를 맞춘다.
- 불확실한 계수는 단일값을 확정하지 않고 근거별 범위와 민감도 scenario를 만든다.
- 테이블 보간은 유효 셀 안에서만 수행한다. 영역 밖에서 조용히 외삽하지 않고 `OUT_OF_ENVELOPE`를 기록한다.

필수 그림: CL-alpha, CD-CL, Cm-alpha, Cm-delta_sym, Cl-delta_diff, Cn-delta_diff, CY/Cl/Cn-beta, spanwise load, derivative-step sensitivity, grid convergence, holdout residual.

## 10. Gazebo로 변환할 때의 구현 설계

### 10.1 전기체 계수와 부분별 하중 중 하나를 선택

**V0 기본안은 전기체 6축 계수 한 세트**를 body link에 적용하고, 좌우 elevon joint angle을 입력으로 읽는 방식이다. visual/joint는 분리하되 전기체 계수에 포함된 wing/elevon 힘을 별도 LiftDrag로 또 더하지 않는다.

부분별 공력 방식이 필요하면 각 부분의 reference·force application point·moment를 정의하고 합산 결과가 전기체 데이터와 일치하는지 먼저 검증한다.

### 10.2 AdvancedLiftDrag 매핑 확인

이번 조사에서 확인한 Gazebo 소스는 `gz-sim8@36691ae8c1cfad3218cefb7484baebcf3da68592`의 [AdvancedLiftDrag.cc](https://github.com/gazebosim/gz-sim/blob/36691ae8c1cfad3218cefb7484baebcf3da68592/src/systems/advanced_lift_drag/AdvancedLiftDrag.cc)다. 이는 조사 기준이며 Ubuntu 설치 버전은 아직 결정하지 않았다.

이 소스의 실제 연산에서 확인한 사항:

- joint angle을 rad→deg로 변환한 후 `*_ctrl`과 곱한다. 따라서 내부 DB의 per-rad 조종면 도함수는 exporter에서 per-degree로 변환해야 한다: `C_per_deg = C_per_rad * pi / 180`.
- alpha/beta 관련 항과 control 항의 단위를 일괄 취급하면 안 된다. rate 항도 nondimensional normalization을 확인한다.
- `span`은 `sqrt(area*AR)`로 계산한다. MAC를 명시하지 않으면 `area/span`으로 대체되므로 추출한 MAC를 반드시 넣는다.
- beta와 dynamic pressure 계산이 이 문서의 일반 DB 정의와 다르다. 확인한 소스는 `atan2(v,u)`와 body x-z 평면의 속도를 사용한다. beta가 큰 조건에서는 단순 계수 복사로 동등성을 가정하지 않는다.
- `cp × force`가 torque에 더해진다. link 원점·관성 원점과 moment reference의 일치 여부를 시험해 모멘트 이중 반영을 막는다.

parameter 이름은 [Gazebo AdvancedLiftDrag API](https://gazebosim.org/api/sim/8/classgz_1_1sim_1_1systems_1_1AdvancedLiftDrag.html)를 참고하되, exporter는 실제 고정한 소스를 기준으로 만든다. `Cm`↔`Cem`, `Cl`↔`Cell`, `Cn`↔`Cen` 이름 차이와 SDF control 구조를 명시적으로 매핑한다.

추가로 body-rate 부호, lift/sideforce 축, induced drag 계산, stall blend, control offset을 소스와 force-level 시험으로 확인한다. upstream `advanced_plane`의 수치는 Scimitar에 복사하지 않는다.

### 10.3 사용자 플러그인 전환 조건

다음이 holdout·force-level 시험에서 드러나고 V0 유효범위 축소로 해결할 수 없을 때 전환한다.

- alpha-beta-elevon coupling이나 비대칭이 선형 도함수로 표현되지 않음
- prop-on 효과나 Re/속도 의존성을 mission 범위에서 무시할 수 없음
- 플러그인의 축/동압/drag 모델 때문에 요구 오차를 만족할 수 없음
- 필요한 비선형 polar와 stall 구간에 충분한 데이터가 확보됨

테이블 플러그인은 오프라인 계산과 같은 계수 평가 함수를 사용하고, 실제 wind-relative velocity와 실제 joint angle을 입력으로 받는다. 모델 범위 밖 상태와 저속 처리 정책을 로그에 남긴다.

## 11. 질량·추진·actuator와 결합

### 질량·CG·관성

출력 shell, spar, motor, battery, servo, FC, payload를 component ledger로 관리한다. CAD solid density만으로 출력물 질량을 계산하지 않는다.

```text
m_total = Σ m_i
r_CG    = Σ(m_i*r_i) / m_total
I_CG    = Σ [R_i*I_i*R_i^T + m_i*((d_i·d_i)*Identity - d_i*d_i^T)]
d_i     = r_i - r_CG
```

tensor의 대칭성·양의 고윳값·주관성모멘트 triangle inequality와 단위를 점검한다. CG 변화가 pitch stability와 trim에 미치는 영향을 공력 모델과 함께 sweep한다.

### 추진계

- motor/prop/battery/ESC의 실제 조합을 확정하고 thrust axis 및 CG에 대한 lever arm을 추출한다.
- V0 thrust curve는 출처와 불확실성을 붙인 제한된 근사로 시작한다.
- 실측은 throttle, RPM, voltage, current, thrust와 응답 지연을 기록한다.
- static thrust만으로 전진 비행 thrust를 고정하지 않는다. 확보 가능한 prop map에 대해 `J=V/(nD)`, `T=CT*rho*n²*D⁴`, `Q=CQ*rho*n²*D⁵`를 사용한다(n은 rev/s).
- 추진 손실·전진속도·전압 강하가 미확인인 구간은 민감도 범위로 관리한다.
- pusher의 propwash가 실제 어떤 표면에 작용하는지 geometry로 확인한 뒤 필요 시 actuator-disk 또는 rotating prop CFD를 추가한다. 회전 prop의 힘과 별도 thrust model을 중복 적용하지 않는다.

### actuator

실제 elevon 각도를 기준으로 neutral, travel limit, command sign, rate limit, time constant, deadband를 기록한다. V0의 ideal servo는 명시적인 가정이다. motor-off/prop-on 공력 DB가 섞이지 않도록 configuration을 구분한다.

## 12. 트림·안정성·검증 순서

### 12.1 PX4 없는 오프라인 트림

speed·질량·CG·대기조건을 고정하고 alpha, pitch, 좌우 elevon, throttle을 미지수로 둔다. 힘과 모멘트의 평형 및 gamma=0 조건을 푼다. 대칭 case에서는 alpha, delta_sym, throttle로 축소할 수 있지만 추력 방향과 수직 성분을 포함한다.

```text
F_aero + F_prop + F_gravity = 0
M_aero_CG + M_prop_CG      = 0
```

해가 actuator/thrust 범위를 넘으면 실패로 기록한다. PX4 gain을 바꿔 잘못된 물리 평형을 숨기지 않는다. neutral point, Cm_alpha 및 정적 안정성은 같은 부호·CG 기준에서 평가하며 특정 static margin을 근거 없이 강제하지 않는다.

trim 근처에서 선형화해 short-period, phugoid, roll subsidence, Dutch roll, spiral 모드를 조사한다. 열린 고리의 불안정 모드가 보이면 수치 오류인지 실제 모델 예측인지 구분하고, 제어기가 안정화할 수 있는지 별도 검토한다.

### 12.2 모델 구현 검증

1. **수치·대칭 검사:** 단위, coefficient scale, beta/좌우 조종면 부호, reference 이동.
2. **force-level 검사:** PX4와 중력을 제거한 시험 fixture에서 정해진 velocity/rate/joint angle의 힘·모멘트를 오프라인 evaluator와 비교.
3. **동압 검사:** 동일 계수·자세에서 속도 2배일 때 정적 공력 하중 4배. Re-dependent 모델에서는 이 단순 비례 시험에 계수를 고정.
4. **기준점 검사:** 동일 wrench를 다른 reference로 표현해 CG에서 결과가 같음을 확인.
5. **자유 운동:** 오프라인 trim을 Gazebo 초기조건으로 주고 초기 가속도·각가속도 비교.
6. **입력 응답:** pitch/roll doublet, elevon pulse, throttle step 및 p/q/r 비교.
7. **PX4 통합:** actuator allocation, 센서 좌표, EKF 상태를 확인한 뒤 stabilize → altitude hold → loiter → waypoint → RTL/landing.

초기 합격 기준 제안:

| 검증 | 기준 후보 |
|---|---|
| 공력 fit holdout | 정상 범위 coefficient NRMSE 5% 이하, force scale floor 0.05 / moment floor 0.01; 최대 오차 별도 보고 |
| exporter / evaluator | 동일 모델의 static force/moment 오차 1% 이하; near-zero 절대 오차 별도 기준 |
| 오프라인 trim | 잔여 힘 norm < 1% mg; 모멘트 norm < 1% mg*MAC |
| Gazebo 초기 trim | 초기 5초의 가속도·각가속도 잔차를 force-level 기준과 비교; steady-state drift 원인 분석 |
| SITL 순항 | 60초 구간 airspeed 목표 대비 RMS 5% 이하, 고도 RMS 5 m 이하를 초기 후보로 검토 |
| 실기체 비교 | 동일 configuration·비슷한 대기조건의 독립 로그로 별도 기준 설정 |

NRMSE의 scale은 holdout reference coefficient RMS와 floor 중 큰 값으로 정의한다. 허용오차는 baseline 결과를 보고 몰래 넓히지 않고 시험 전에 versioned criteria로 확정한다. 정상 범위 밖 데이터는 별도 평가한다.

### 12.3 불확실성과 식별

초기 민감도 변수는 mass, CG x, Cm_alpha, Cm_delta, Cl_delta, damping, CD0, thrust, servo lag다. 범위는 원본 편차·solver 차이·실측 오차에 근거해 정하고, 근거가 없으면 exploratory range라고 표시한다.

one-at-a-time 영향도를 먼저 구하고 중요한 조합만 표본화한다. seed, 초기조건, model version을 저장한다. 실비행 식별에는 비슷한 신호를 만드는 CG/Cm0/elevon trim, thrust/drag 사이의 상관성을 확인한다. 학습 로그와 독립 검증 로그를 분리하고 여러 비행조건에서 비교한다.

## 13. Ubuntu에서 실행할 작업 패키지

아래 파일명은 예정 산출물이며 현재 실행 가능한 스크립트가 아니다.

| 순서 | 작업 | 예정 산출물 | 다음 단계로 가는 조건 |
|---|---|---|---|
| A0 | upstream CFD/PX4 전수 감사 | `upstream_data_audit.md`, raw inventory | 복원 가능 데이터와 누락 항목 구분 |
| A1 | CAD 조립·좌표·단면 추출 | geometry manifest, sections, reference geometry | 치수·축·hinge·configuration 검토 |
| A2 | VSPAERO/AVL pilot 및 panel 수렴 | VSP/AVL 입력, pilot report | 기준값 통일·힘 방향·해상도 확인 |
| A3 | baseline/control/beta/rate sweeps | cases CSV, polars, derivatives | 품질 flag와 유효범위 확보 |
| A4 | 점성항력 및 선택 CFD | section polars, CFD convergence report | force/moment와 격자/영역 민감도 보고 |
| A5 | coefficient fit 및 holdout | `coefficients.yaml`, fit report | 오차 기준·불확실성 검토 |
| A6 | mass/prop/actuator 결합·trim | trim map, eigenmode report | 물리적 범위 내 평형 또는 실패 원인 설명 |
| A7 | SDF exporter / 필요 시 plugin | `model.sdf`, mapping report | force-level 동등성 확인 |
| A8 | Gazebo/PX4 응답·mission 검증 | GT/ULog, dynamics/mission reports | 단계별 acceptance criteria 충족 |

자동화 인터페이스 후보: `audit_upstream.py`, `extract_sections.py`, `build_aero_cases.py`, `run_vspaero.py`, `normalize_coefficients.py`, `fit_aero_model.py`, `solve_trim.py`, `export_gazebo_aero.py`, `validate_wrench.py`.
각 실행은 입력 manifest를 받아 case ID, 버전, exit status, 로그 경로를 남기도록 설계한다.

**첫 Ubuntu 세션의 공력 목표는 A0/A1의 자료·형상 계약 확정과 A2 pilot 준비다.** 대량 sweep이나 CFD 배치를 시작하기 전에 pilot의 실제 소요 시간, CPU/RAM, 출력 크기를 측정한다. 기체 제작이 진행되면 실측값을 ledger에 추가해 같은 model revision으로 재계산한다.
