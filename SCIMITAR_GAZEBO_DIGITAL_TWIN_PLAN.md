# Scimitar Gazebo Digital Twin Development Plan

> 상태: GitHub 저장소 및 문서 준비 단계. 실제 개발·실행은 Ubuntu 컴퓨터에서 진행한다.
> 이 문서는 구현 계획이며, 실행 가능한 시뮬레이터나 검증 완료 보고서가 아니다.

- 저장소: [boss123516/VIO-Fixed_wings](https://github.com/boss123516/VIO-Fixed_wings)
- 개발 통합 브랜치: `dev`
- 원본 보존 브랜치: `main`
- 시작 upstream revision: `24a5d5986ae11a63882004382576460844760700`
- [프로젝트 README](readme.md)

## 현재 진행 상태

- [x] Scimitar 저장소 포크
- [x] `main` 기준 `dev` 브랜치 생성
- [x] 프로젝트 README 및 개발 계획 정리
- [ ] Ubuntu 개발 환경과 의존성 버전 확정
- [ ] upstream CAD / CFD / PX4 데이터 감사
- [ ] Gazebo 모델 및 PX4 SITL 연동 구현
- [ ] 비행 동역학·미션 검증

아래 단계의 수치, 파일 구조, 실행 스크립트는 별도 완료 표시가 없는 한 계획 또는 예시다.
개발 일정은 데이터 감사 결과를 바탕으로 정한다.

## 0. 결론

**개발은 원본 `Oscilous/Adapted-Modular-Scimitar`를 Fork해서 진행한다.**

`main`은 upstream 추적용으로 유지하고, `dev`에서 우리 프로젝트의 변경을 통합한다.

```text
VIO-Fixed_wings
├── main                         # upstream 추적 / 원본 보존
└── dev                          # 개발 통합 (현재 생성됨)
    ├── feat/print-prep           # 제작 작업 시 생성 예정
    └── feat/gazebo-digital-twin   # 시뮬레이션 작업 시 생성 예정
```

작업 브랜치는 `dev`에서 만들고 PR의 base를 이 저장소의 `dev`로 지정한다.
제작 결과와 시뮬레이션 변경은 `dev`를 통해 공유하며, 필요한 CAD revision과 실측 데이터 revision을 기록한다.
`dev`와 `dev/...` 이름은 Git ref 경로가 충돌하므로 함께 사용하지 않는다.

초기에는 **하나의 Fork 안에서 CAD → Simulation → Manufacturing의 traceability를 유지하는 것**이 가장 좋다.

PX4 자체는 당장 Fork하지 않는다.

- `PX4-Autopilot`은 별도 dependency로 clone
- 필요하면 custom airframe config / parameter patch만 우리 repo에 관리
- PX4 core를 실제로 수정해야 할 때만 PX4 fork를 만든다

---

# 1. 프로젝트 목표

Scimitar의 단순 3D mesh를 Gazebo에 띄우는 것이 목표가 아니다.

최종 목표는:

> **실제 제작되는 Scimitar와 기하학적·질량적·공력적·추진계·actuator 특성이 대응되는 6-DoF 동역학 모델을 구축하고, PX4 SITL에서 실제 비행 전 Mission을 검증하는 것**

이다.

즉 다음 구조를 만든다.

```text
                  SAME SOURCE CAD
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
       Manufacturing           Simulation
       STL / 3MF / G-code      Gazebo SDF
             │                     │
             ▼                     ▼
        Physical UAV          Digital Twin
             │                     │
             └──────────┬──────────┘
                        ▼
                 Flight Comparison
```

---

# 2. 기술 스택

## Simulation

- Gazebo Harmonic (목표 버전; Ubuntu/PX4 호환성 확인 후 고정)
- SDF
- PX4 SITL
- QGroundControl
- ROS 2: 후속 VIO 단계에서 사용
- Python: parameter extraction / validation / plotting

## Aerodynamics

초기:

- upstream CFD 결과 분석
- CAD geometry extraction
- PX4/Gazebo Advanced Plane model 구조를 template로 사용

필요 시:

- OpenVSP / VSPAERO
- XFLR5
- additional CFD

## Real Vehicle

후속:

- Pixhawk
- PX4
- Camera + IMU
- Jetson
- RTK-GNSS Ground Truth

---

# 3. 왜 Gazebo `advanced_plane`을 출발점으로 사용하는가

PX4의 현재 Gazebo fixed-wing simulation에는:

```text
gz_rc_cessna
gz_advanced_plane
```

이 존재한다.

`advanced_plane`은 일반 plane보다 공력 모델을 더 세밀하게 조절하기 위한 기반으로 사용한다.

Scimitar는 flying-wing이므로 최종 model은 그대로 복사하지 않고:

```text
advanced_plane physics concept
            +
Scimitar geometry
            +
Scimitar elevons
            +
Scimitar pusher motor
            +
Scimitar aerodynamic coefficients
```

로 구성한다.

---

# 4. Git Workflow

Fork 및 `dev` 생성은 완료되었다. 이후 Ubuntu에서 다음과 같이 시작한다.

```bash
git clone --branch dev https://github.com/boss123516/VIO-Fixed_wings.git
cd VIO-Fixed_wings
git remote add upstream https://github.com/Oscilous/Adapted-Modular-Scimitar.git
git fetch upstream
git switch -c feat/gazebo-digital-twin
```

제작 작업은 `dev`에서 별도 `feat/print-prep` 브랜치를 만든다.
작업 결과는 `dev` 대상으로 검토·통합한다. `main`에는 프로젝트 전용 변경을 넣지 않는다.
동시에 로컬 작업할 때는 별도 checkout 또는 `git worktree`로 작업 폴더를 분리한다.

원본 자료의 출처와 CC BY 4.0 표시를 유지한다. 복사하는 PX4/Gazebo 코드·모델은 해당 파일의 라이선스를 별도로 확인하고 유지한다.

---

# 5. Repository 확장 구조

`feat/gazebo-digital-twin`에서 단계별로 다음 구조를 추가할 계획이다. 현재 이 구조와 스크립트는 구현되지 않았다.

```text
VIO-Fixed_wings/
│
├── CAD/
├── CFD/
├── PX4 Firmware/
├── Media/
│
├── simulation/
│   ├── README.md
│   │
│   ├── gazebo/
│   │   ├── models/
│   │   │   └── scimitar/
│   │   │       ├── model.config
│   │   │       ├── model.sdf
│   │   │       └── meshes/
│   │   │           ├── fuselage.dae
│   │   │           ├── left_wing.dae
│   │   │           ├── right_wing.dae
│   │   │           ├── left_elevon.dae
│   │   │           ├── right_elevon.dae
│   │   │           └── propeller.dae
│   │   │
│   │   └── worlds/
│   │       ├── scimitar_empty.sdf
│   │       ├── scimitar_mission.sdf
│   │       └── scimitar_wind.sdf
│   │
│   ├── aero/
│   │   ├── raw/
│   │   ├── processed/
│   │   ├── coefficients.yaml
│   │   └── README.md
│   │
│   ├── mass_properties/
│   │   ├── components.yaml
│   │   ├── mass_properties.yaml
│   │   └── README.md
│   │
│   ├── propulsion/
│   │   ├── motor_prop.yaml
│   │   └── README.md
│   │
│   ├── px4/
│   │   ├── parameters/
│   │   ├── airframe/
│   │   └── scripts/
│   │
│   ├── missions/
│   │   ├── nominal/
│   │   ├── gps_denied/
│   │   └── failure_cases/
│   │
│   ├── scripts/
│   │   ├── extract_geometry.py
│   │   ├── compute_mass_properties.py
│   │   ├── validate_model.py
│   │   └── compare_trajectories.py
│   │
│   └── reports/
│       ├── geometry_report.md
│       ├── aero_report.md
│       ├── mass_report.md
│       └── validation_report.md
│
└── docs/
    └── digital_twin.md
```

---

# 6. Phase 0 — Upstream Data Audit

가장 먼저 **추측을 금지**하고 upstream에서 실제 데이터를 추출한다.

확인 대상:

```text
CAD/
CFD/
PX4 Firmware/Parameters.params
Media/Assembly/
readme.md
```

## 반드시 답해야 할 질문

1. 실제 airframe geometry는 어떤 STEP 파일 조합인가?
2. 좌/우 elevon geometry와 hinge axis는 정확히 어디인가?
3. motor/propeller 위치와 thrust axis는 어디인가?
4. payload/battery/FC 위치 자료가 있는가?
5. upstream에서 사용한 total mass는 정확히 얼마인가?
6. CG 위치가 수치로 제공되는가?
7. inertia tensor 자료가 있는가?
8. CFD `project_results.xml` 안에 어떤 coefficient가 들어 있는가?
9. CFD 결과가 전체 aircraft인지 특정 configuration인지?
10. `Parameters.params`의 airframe/control allocation은 무엇인가?
11. elevon mapping은 어떻게 되어 있는가?
12. 실제 flight-test parameter를 simulation에 재사용할 수 있는가?

### 산출물

```text
simulation/reports/upstream_data_audit.md
```

---

# 7. Phase 1 — CAD → Simulation Geometry

## 목표

실제 CAD와 시뮬레이터 visual/collision geometry의 기준을 동일하게 만든다.

### STEP에서 추출

```text
Wingspan
Reference wing area
MAC
Sweep
Dihedral
Airfoil geometry
Fuselage dimensions
Left elevon area
Right elevon area
Elevon hinge locations
Motor position
Thrust axis
Sensor mounting locations
```

---

## Visual mesh와 Collision mesh 분리

### Visual

CAD 형상에 가깝게:

```text
visual/
high-resolution mesh
```

### Collision

physics 계산 안정성을 위해 단순화:

```text
collision/
fuselage primitive / simplified mesh
wing simplified geometry
```

**고해상도 CAD mesh를 collision mesh로 그대로 사용하지 않는다.**

---

# 8. Coordinate Frame Convention

가장 먼저 좌표계를 확정한다.

PX4 / aircraft convention을 명확하게 mapping한다.

문서:

```text
simulation/FRAME_CONVENTIONS.md
```

반드시 기록:

```text
CAD frame
Gazebo model frame
Gazebo world frame
PX4 body frame
PX4 NED frame
ROS ENU frame
Camera optical frame
IMU frame
```

좌표계 변환을 문서화하지 않은 상태에서는 센서와 VIO 개발을 진행하지 않는다.

---

# 9. Phase 2 — Mass / CG / Inertia Model

## V0: 제작 전

CAD와 upstream 자료를 이용해 추정한다.

필요 값:

```text
mass

CG:
x_cg
y_cg
z_cg

Inertia:
Ixx
Iyy
Izz
Ixy
Ixz
Iyz
```

SDF:

```xml
<inertial>
    <mass>...</mass>
    <inertia>
        <ixx>...</ixx>
        ...
    </inertia>
</inertial>
```

---

## 주의

3D printed shell을 solid PLA로 가정하지 않는다.

실제:

```text
LW-PLA thin shell
+
carbon spar
+
motor
+
battery
+
servo
+
Pixhawk
+
payload
```

이기 때문이다.

따라서 V0에서는 estimated 값을 쓰되 모든 값에 출처, 단위, 적용 조건, 불확실성을 붙인다.

예:

```yaml
mass:
  value: null  # 감사 후 입력; 아직 확인되지 않은 값
  unit: kg
  source_type: null
  source_ref: null
  status: UNKNOWN

Ixx:
  value: null
  unit: kg*m^2
  source_type: null
  source_ref: null
  status: UNKNOWN
```

---

# 10. 실제 기체 제작 후 Mass Model V1

3D printing branch의 `weight_log.csv`와 연결한다.

각 component:

```text
part mass
part CG
part orientation
```

을 CAD에 배치해 전체 CG/inertia를 다시 계산한다.

즉:

```text
Print Result
    ↓
Actual part mass
    ↓
simulation/mass_properties/components.yaml
    ↓
recompute
    ↓
model.sdf update
```

를 자동화한다.

---

# 11. Phase 3 — Aerodynamic Model

## 목표

단순히 "날개가 있으니 LiftDrag plugin" 수준으로 끝내지 않는다.

Scimitar에 대해 최소 다음 계수가 필요하다.

```text
CL(alpha)
CD(alpha)
Cm(alpha)

control derivatives:
dCL / d(delta_elevon)
dCm / d(delta_elevon)

lateral-directional:
CY(beta)
Cl(beta)
Cn(beta)

rate derivatives, if obtainable:
Cl_p
Cm_q
Cn_r
...
```

---

# 12. Upstream CFD 먼저 해석

`CFD/` 아래 파일을 먼저 분석한다.

특히:

```text
project_results.xml
*.fld
other result files
```

에서 다음을 찾는다.

```text
AoA
Velocity
Lift
Drag
Moment
CL
CD
Cm
L/D
reference area
reference length
density
```

### 원칙

upstream CFD 값을 확인하기 전에 새 CFD를 돌리지 않는다.

---

# 13. CFD가 충분하지 않은 경우

다음 순서로 부족한 데이터를 채운다.

## Level 1

OpenVSP / VSPAERO

장점:

- fixed-wing coefficient sweep에 적합
- alpha/beta/control surface sweep 가능
- 반복 자동화가 쉬움

## Level 2

필요한 영역만 추가 CFD

예:

```text
near stall
large elevon deflection
fuselage interaction
```

처음부터 모든 operating point를 고비용 CFD로 계산하지 않는다.

---

# 14. Aerodynamic Sweep Matrix

최소 후보:

```text
AoA:
-10 to +20 deg

Beta:
-15 to +15 deg

Elevon symmetric:
-20 to +20 deg

Elevon differential:
-20 to +20 deg
```

실제 range는 CAD/PX4/servo geometry 분석 후 확정한다.

---

# 15. Gazebo Aerodynamic Implementation

V0에서는 PX4의 Gazebo plane model 구조를 참고해 aerodynamic system을 구성한다.

예상 구조:

```text
Left wing lift/drag
Right wing lift/drag
Left elevon contribution
Right elevon contribution
Fuselage drag
```

Scimitar는 flying wing이므로:

```text
left elevon
right elevon
```

이 pitch와 roll에 동시에 영향을 준다.

---

# 16. Advanced Model로 확장

기본 LiftDrag parameterization이 Scimitar의 coefficient table을 충분히 표현하지 못하면:

```text
Custom Gazebo aerodynamic system plugin
```

으로 넘어간다.

입력:

```text
airspeed
alpha
beta
p q r
left elevon
right elevon
air density
```

출력:

```text
Fx Fy Fz
Mx My Mz
```

즉:

```text
F_aero = f(state, controls)
M_aero = f(state, controls)
```

를 직접 계산한다.

---

# 17. Phase 4 — Propulsion Model

Scimitar pusher configuration을 모델링한다.

필요 값:

```text
Motor KV
Battery voltage
Prop diameter
Prop pitch
ESC behavior
Motor location
Thrust axis
```

V0:

```text
throttle → approximate thrust
```

V1:

실제 motor/propeller bench test 결과:

```text
throttle
RPM
current
voltage
thrust
```

를 lookup table로 사용.

---

# 18. Propwash

초기 Mission validation에서는 propwash를 단순화할 수 있다.

필요성이 확인되면 후속으로:

```text
pusher propwash → elevon/wing interaction
```

을 추가한다.

초기 Digital Twin 완성을 방해하지 않도록 우선순위를 낮게 둔다.

---

# 19. Phase 5 — Actuator Model

Left / Right elevon joint:

```text
angle limit
neutral angle
rate limit
delay
```

Motor:

```text
throttle response time
RPM lag
```

Servo V0:

```text
ideal position controller
```

Servo V1:

실제 서보 측정:

```text
command → measured angle
time response
deadband
```

---

# 20. Phase 6 — PX4 Integration

가장 먼저 upstream의:

```text
PX4 Firmware/Parameters.params
```

를 분석한다.

Scimitar가 실제로 사용한:

```text
airframe
actuator mapping
control parameters
airspeed settings
TECS
FW attitude controller
```

을 파악한다.

**원 기체 parameter를 먼저 분석하고, SITL에 적합한 항목만 선별해 이식한다.**

원본 parameter 파일은 그대로 보존한다. 센서 보정값, 하드웨어 출력 설정, 펌웨어 버전별 parameter 차이를 검토하고 SITL용 airframe/control allocation을 별도로 관리한다. 실기체 설정 전체를 일괄 import하지 않는다.

PX4 자체는 현재 flying-wing airframe을 지원하므로 Scimitar control allocation에 활용한다.

---

# 21. PX4 dependency 관리

우리 Fork에 PX4 전체 source를 복사하지 않는다.

권장:

```text
external/
└── PX4-Autopilot
```

또는 bootstrap script에서 별도로 clone. 일반 clone 방식이면 `external/`을 Git 추적에서 제외하며, PX4와 모델 의존성의 commit을 고정한다.

예:

```bash
git clone https://github.com/PX4/PX4-Autopilot.git external/PX4-Autopilot
```

revision 고정:

```text
simulation/px4/PX4_VERSION
```

---

# 22. Custom Gazebo Model 실행 목표

최종적으로 다음과 같은 실행 형태를 만든다.

예상:

```bash
./simulation/scripts/setup.sh
./simulation/scripts/run_scimitar.sh
```

내부적으로:

```text
Gazebo Harmonic
+
scimitar model
+
PX4 SITL
+
QGroundControl
```

을 시작한다.

가능하면 사용자에게 복잡한 환경변수를 직접 입력시키지 않는다.

---

# 23. Phase 7 — Sensor Model

V0 센서:

```text
IMU
barometer
magnetometer
GPS
airspeed
```

후속 VIO용:

```text
downward/forward camera
camera noise
IMU noise
timestamp
camera-IMU extrinsic
```

---

# 24. Sensor Extrinsic Source

센서 위치를 대충 놓지 않는다.

CAD에서 실제 장착 예정 위치를 사용한다.

예:

```text
body → IMU
body → camera
body → GPS
```

transform을 파일로 관리:

```yaml
camera:
  xyz: [...]
  rpy: [...]

imu:
  xyz: [...]
  rpy: [...]
```

---

# 25. Phase 8 — Ground Truth

Gazebo world pose는 simulation GT로 별도 logging한다.
GT와 PX4 추정 상태의 timestamp·좌표계를 맞추는 로깅은 첫 트림/비행 시험 전에 구현한다.
이 장의 번호가 구현을 늦추는 의미는 아니다.

반드시 구분:

```text
Ground Truth
≠
PX4 estimated state
≠
VIO state
```

logging:

```text
GT position
GT orientation
GT velocity
GT angular velocity

PX4 estimator
VIO estimator
```

---

# 26. Phase 9 — 가장 먼저 해야 하는 Flight Test

처음부터 autonomous mission을 하지 않는다.

## Test 1 — Static

```text
gravity
mass
CG
sensor orientation
servo direction
motor thrust direction
```

검증.

---

## Test 2 — Control Surface

```text
Pitch command
→ both elevons correct direction

Roll right
→ differential elevon correct direction
```

검증.

---

## Test 3 — Motor

```text
Throttle ↑
→ forward acceleration
```

확인.

---

## Test 4 — Simple Launch

초기 속도를 주거나 simple launch condition을 사용해:

```text
trimmed straight flight
```

가능 여부 확인.

---

# 27. Phase 10 — Trim Validation

중요.

정상 순항에서:

```text
constant altitude
constant airspeed
near-zero angular rate
```

가 되는 trim point를 찾는다.

기록:

```text
airspeed
throttle
pitch
elevon trim
AoA
```

이 값이 말이 안 되면 Mission 단계로 넘어가지 않는다.

---

# 28. Phase 11 — Basic Dynamics Validation

다음 입력에 대한 response를 기록한다.

```text
Elevon pulse
Roll doublet
Pitch doublet
Throttle step
```

측정:

```text
p
q
r
roll
pitch
airspeed
altitude
```

목적:

- unstable physics 확인
- sign error 확인
- aerodynamic coefficient 문제 확인
- actuator mapping 확인

---

# 29. Phase 12 — PX4 Autonomous Flight

다음 순서로 진행한다.

```text
1. Stabilized flight
2. Altitude hold
3. Loiter
4. Waypoint
5. Multi-waypoint
6. Return
7. Auto landing
```

각 단계가 PASS해야 다음 단계로 넘어간다.

---

# 30. Mission V0

첫 mission은 단순해야 한다.

```text
Takeoff / launch
     ↓
WP1
     ↓
WP2
     ↓
WP3
     ↓
Loiter
     ↓
Return
     ↓
Land
```

목표는 mission logic과 aircraft model 검증이지 VIO가 아니다.

---

# 31. Mission V1 — GPS Denied

Nominal mission이 안정된 후:

```text
GPS ON
  ↓
Launch
  ↓
WP1
  ↓
GPS DENIED ZONE
  ↓
WP2
  ↓
Target / Loiter
  ↓
WP3
  ↓
GPS RECOVERY
  ↓
Return
```

을 만든다.

---

# 32. VIO Integration은 그 이후

순서:

```text
Perfect GT navigation
       ↓
GPS navigation
       ↓
VIO logging only
       ↓
VIO vs GT
       ↓
VIO fusion
       ↓
GPS denied
```

**VIO를 model validation과 동시에 넣지 않는다.**

기체 동역학 문제와 VIO 문제를 분리해야 디버깅이 가능하다.

---

# 33. Phase 13 — Failure Injection

최소:

```text
wind
wind gust
GPS loss
camera dropout
VIO dropout
telemetry loss
airspeed sensor fault
barometer bias
IMU noise increase
```

를 scenario로 만든다.

---

# 34. Wind Scenario

기본:

```text
still air
```

PASS 이후:

```text
steady headwind
crosswind
gust
```

순서.

Digital Twin V0부터 극단적인 turbulence를 넣지 않는다.

---

# 35. Simulation Acceptance Criteria

Digital Twin V0 완료 조건:

- [ ] CAD-based geometry
- [ ] correct scale
- [ ] correct control surface geometry
- [ ] pusher motor axis correct
- [ ] estimated mass/CG/inertia
- [ ] aerodynamic coefficient source documented
- [ ] stable trimmed flight
- [ ] correct elevon control
- [ ] PX4 SITL connected
- [ ] QGroundControl connected
- [ ] waypoint mission success
- [ ] GT logging
- [ ] 고정 버전·초기조건·seed로 재실행 가능하며, 합의한 수치 허용오차 내 결과 재현

---

# 36. Digital Twin V1 완료 조건

실제 airframe 제작 후:

- [ ] actual mass
- [ ] measured CG
- [ ] component weight distribution
- [ ] updated inertia
- [ ] actual motor/prop configuration
- [ ] measured/verified servo deflection
- [ ] simulation model updated

---

# 37. Digital Twin V2 완료 조건

실비행 이후 System Identification 수행.

비교:

```text
Simulation
vs
Real PX4 log / RTK
```

identification 대상:

```text
CL_alpha
CD
Cm_alpha
control effectiveness
roll damping
pitch damping
yaw stability
motor thrust curve
servo dynamics
```

완료 후에야 "실기체와 동역학적으로 보정된 Digital Twin"으로 간주한다.

---

# 38. Simulation ↔ Real 검증 Metrics

공통 mission을 simulation과 real aircraft에 실행한다.

비교:

```text
trajectory
airspeed
altitude
roll
pitch
yaw
angular rates
throttle
elevon commands
energy usage
```

metrics:

```text
position RMSE
altitude RMSE
airspeed RMSE
attitude RMSE
rate RMSE
cross-track RMSE
mission completion
landing error
```

---

# 39. 개발 단계와 완료 기준

각 단계의 기간은 아직 미정이다. 물리 모델 수치와 오차 허용 기준은 데이터 감사 이후 정하고 시험 전에 문서화한다.

| 단계 | 작업 | 완료 기준 |
|---|---|---|
| 준비 (현재) | GitHub fork, dev, README, 계획 | Ubuntu에서 작업을 이어갈 저장소·문서 준비 |
| Ubuntu 기반 | OS/CPU/GPU 확인, PX4/Gazebo/모델 revision 고정 | 기본 fixed-wing 예제 실행 및 QGC 연결 |
| 원본 감사 | CAD/CFD/PX4 및 제작 자료 분석 | geometry inventory, known/unknown 표, 좌표계 설계 |
| Model bootstrap | CAD 기반 visual/collision, elevon, 센서 및 bridge | 스케일·축·조종면 방향 확인, Scimitar/PX4 연결 |
| Physics V0 | 질량·CG·관성·공력·추진·actuator, GT/PX4 로깅 | 정적 시험, 트림, 입력 응답 검증 |
| Autonomous | Stabilized, altitude hold, loiter, waypoint, RTL, landing | 단계별 로그와 합의한 허용오차 기준 충족 |
| Sensor / VIO | 카메라·IMU 모델, ROS 2, VIO 기록과 비교 | GT 대비 평가 후 fusion 및 GPS-denied 시험 |
| Real calibration | 제작 실측 V1, 실비행 식별 V2 | 실측 반영 및 sim/real 오차 보고서 |

실제 제작과 질량 측정은 개발과 병행할 수 있다. VIO 착수는 기본 기체 모델 검증을 전제로 하며, 실비행 자료 확보 시 V2 보정을 진행한다.

---

# 40. Ubuntu에서 시작할 첫 개발 작업

1. 4장의 명령으로 `dev`를 clone하고 작업 브랜치를 생성한다.
2. Ubuntu 버전·아키텍처·그래픽 환경을 기록하고 PX4/Gazebo 호환 조합을 결정한다.
3. 기본 fixed-wing 예제의 실행·QGroundControl 연결을 확인한다. 예제 성공은 Scimitar 검증으로 취급하지 않는다.
4. `CAD/`, `CFD/`, `PX4 Firmware/`, `3D_Printing/`, `Media/` 및 upstream README를 감사한다.
5. 특히 `CFD/project_results.xml`과 `PX4 Firmware/Parameters.params`를 분석한다.
6. `simulation/reports/upstream_data_audit.md`를 작성한 뒤 Scimitar 물리 모델 설계를 진행한다.

감사 보고서 필수 항목:

- Known parameters: 원본 경로·revision·단위·조건 포함
- Unknown parameters
- Assumed parameters: 추정 근거와 불확실성 포함
- Parameters requiring physical measurement
- Parameters requiring aerodynamic analysis
- Geometry inventory 및 조립 좌표계
- PX4 parameter 호환성 및 재사용 가능 항목

환경 준비와 자료 감사는 병행할 수 있다. 지금의 GitHub 문서 작업에는 설치·감사·시뮬레이터 구현이 포함되지 않는다.

---

# 41. Codex가 하지 말아야 하는 것

다음은 금지.

```text
CAD와 관계없는 임의 aircraft geometry 생성
임의 mass 사용 후 기록하지 않기
임의 CG 사용
임의 aerodynamic coefficients 사용 후 source 미표기
Cessna coefficient를 그대로 Scimitar coefficient로 사용
Mission이 된다는 이유만으로 model을 validated라 부르기
VIO와 aircraft physics를 동시에 디버깅하기
```

---

# 42. Parameter Provenance

확인·산출된 수치의 출처는 다음 네 종류 중 하나로 표시한다. 확인되지 않은 값은 `value: null`, `status: UNKNOWN`으로 남긴다.

```text
UPSTREAM
CAD_DERIVED
SIMULATION_DERIVED
MEASURED
```

추정값은:

```text
ASSUMED
```

로 표시한다.

예:

```yaml
wing_area:
  value: ...
  source_type: CAD_DERIVED

mass:
  value: ...
  source_type: UPSTREAM

Ixx:
  value: ...
  source_type: ASSUMED
```

Digital Twin 정확도를 높일수록 `ASSUMED` 항목이 줄어야 한다.

---

# 43. Versioning

simulation 결과마다 다음을 기록한다.

```text
Scimitar upstream commit
Our fork commit
PX4 commit
Gazebo version
aero coefficient version
mass-property version
mission version
```

예:

```yaml
scimitar_commit: ...
px4_commit: ...
gazebo: harmonic
aero_model: v0.2
mass_model: v0.1
mission: gps_nominal_v1
```

---

# 44. 최종 프로젝트 흐름

```text
upstream main → fork main → dev
                            ├─ feat/print-prep → 제작 / 실측
                            └─ feat/gazebo-digital-twin → Physics Model V0
                                      ↓ dev에서 CAD·실측 데이터 통합
                              Measured Airframe V1
                                      ↓ 실비행 로그 / System Identification
                              Flight-identified Digital Twin V2
```

기본 모델과 nominal mission 검증 후 VIO 평가·fusion·GPS-denied 시험으로 확장한다.
실측 및 비행 식별 결과를 얻을 때마다 같은 CAD 구성과 모델 revision에 연결한다.

---

# 45. 현재 세션의 완료 기준과 다음 작업

현재 범위는 GitHub 준비다.

- [x] Fork 확인: `boss123516/VIO-Fixed_wings`
- [x] `main`에서 `dev` 생성
- [x] 프로젝트 README 및 개발 계획을 `dev`에 정리

Ubuntu에서 수행할 다음 작업:

- [ ] 개발용 clone 및 `feat/gazebo-digital-twin` 생성
- [ ] 의존성 버전 고정 및 기본 예제 실행
- [ ] 필요한 `simulation/` 구조 생성
- [ ] upstream CAD/CFD/PX4 data audit
- [ ] geometry inventory 및 known/unknown dynamics parameter table
- [ ] advanced_plane 구조 분석 및 최초 model.sdf 설계

**자료 감사와 모델 설계 전에 임의 coefficient로 Scimitar 비행 검증을 주장하지 않는다.**

---

# 46. 참고 upstream

Scimitar:

```text
https://github.com/Oscilous/Adapted-Modular-Scimitar
```

PX4 Gazebo Models:

```text
https://github.com/PX4/PX4-gazebo-models
```

PX4 Autopilot:

```text
https://github.com/PX4/PX4-Autopilot
```

PX4 Documentation:

```text
https://docs.px4.io/
```

---

# 47. 핵심 원칙

이 프로젝트에서 가장 중요한 원칙:

> **처음에는 "정확한 Digital Twin"을 주장하지 않는다.**

진행 단계:

```text
Physics Model V0
    ↓
Measured Airframe V1
    ↓
Flight-identified Digital Twin V2
```

그리고 모든 Mission은:

```text
Simulation PASS
      ↓
Ground Test PASS
      ↓
Real Flight
```

순서로 진행한다.
