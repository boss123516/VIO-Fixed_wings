# VIO-Fixed_wings

Scimitar 기반 고정익 기체의 제작 자료와 Gazebo / PX4 SITL 시뮬레이션을 연결하고, 이후 Visual-Inertial Odometry(VIO)를 이용한 항법을 연구하기 위한 개발 저장소입니다.

**현재 단계: GitHub 저장소와 개발 계획 준비. 실제 개발·시뮬레이션 실행은 Ubuntu 컴퓨터에서 진행합니다.**

## 목표

- 동일한 CAD를 기준으로 제작 형상과 시뮬레이션 모델을 관리합니다.
- 기체의 질량·CG·관성·공력·추진·조종면 특성을 출처와 함께 모델링합니다.
- PX4 SITL에서 기본 비행과 waypoint mission을 검증합니다.
- 기본 기체 모델 검증 후 카메라/IMU, VIO, GPS-denied 시나리오로 확장합니다.
- 실제 제작·비행 측정값을 반영해 모델을 단계적으로 보정합니다.

## 현재 상태

| 항목 | 상태 |
|---|---|
| upstream fork 및 `dev` 브랜치 | 완료 |
| 프로젝트 README 및 개발 계획 | 정리됨 |
| Ubuntu 개발 환경 / 의존성 버전 확정 | 예정 |
| CAD / CFD / PX4 원본 데이터 감사 | 예정 |
| Scimitar Gazebo 모델 / PX4 SITL 연결 | 미구현 |
| 트림·동역학·자율비행 검증 | 미실시 |
| VIO / GPS-denied 시험 | 후속 단계 |

현재 실행 가능한 Scimitar 시뮬레이터와 setup/run 스크립트는 제공되지 않습니다. 원본 저장소의 실험 결과는 이 fork의 시뮬레이션 검증 결과가 아닙니다.

## 개발 계획

공력해석의 도구 선택, 계산 조건, CFD 격자·수렴, 계수 DB, Gazebo 매핑과 트림 검증은 [공력·동역학 상세 실행 계획](AERODYNAMICS_ENGINEERING_PLAN.md)에 정리했습니다. 원본 CFD/PX4 텍스트 일부를 사전 조사했으며 전체 데이터 감사와 실제 해석은 아직 진행 전입니다.

전체 범위와 단계별 검증 기준은 [Scimitar Gazebo Digital Twin 개발 계획](SCIMITAR_GAZEBO_DIGITAL_TWIN_PLAN.md)을 참고하세요.

첫 Ubuntu 작업은 **환경 확인 → 기본 PX4 예제 실행 → 원본 데이터 감사 → Scimitar 모델 설계** 순서로 진행합니다. 환경 준비와 자료 감사는 병행할 수 있습니다.

목표 스택은 Gazebo Harmonic, PX4 SITL, SDF, QGroundControl, Python입니다. Ubuntu/PX4/Gazebo의 정확한 버전 조합은 개발 환경에서 확인한 뒤 고정합니다. ROS 2와 VIO는 후속 단계입니다.

## 브랜치 운영

| 브랜치 | 용도 |
|---|---|
| `main` | upstream 추적 및 원본 보존 |
| `dev` | 우리 프로젝트의 문서·제작·시뮬레이션 변경 통합 |
| `feat/gazebo-digital-twin` | 시뮬레이션 작업 시작 시 생성 |
| `feat/print-prep` | 제작 작업 시작 시 생성 |

작업 브랜치는 `dev`에서 만들고 이 저장소의 `dev`를 대상으로 PR을 작성합니다. 제작과 시뮬레이션의 공통 CAD·실측 결과도 `dev`를 통해 공유합니다. `dev`와 `dev/...` 이름은 Git ref 경로가 충돌하므로 함께 사용하지 않습니다.

## Ubuntu에서 저장소 준비

다음은 새 작업 폴더에서 사용하는 Git 명령입니다. 시뮬레이터 설치 명령은 환경 확인 후 추가합니다.

```bash
git clone --branch dev https://github.com/boss123516/VIO-Fixed_wings.git
cd VIO-Fixed_wings
git remote add upstream https://github.com/Oscilous/Adapted-Modular-Scimitar.git
git fetch upstream
git switch -c feat/gazebo-digital-twin
```

PX4는 추후 별도 의존성으로 clone하고 commit을 고정합니다. PX4 전체 소스를 이 저장소에 복사하지 않습니다.

## 저장소 구성

| 경로 | 내용 |
|---|---|
| `CAD/` | upstream STEP 형상 및 부품 모델 |
| `3D_Printing/` | upstream STL / 3MF / 출력 설정 자료 |
| `CFD/` | upstream 공력 해석 파일 |
| `PX4 Firmware/` | upstream PX4 parameter 및 설명 |
| `M5 Software/` | upstream M5Stack 관련 파일 |
| `Media/` | upstream 조립 이미지 및 영상 |
| `SCIMITAR_GAZEBO_DIGITAL_TWIN_PLAN.md` | 이 fork의 전체 개발 계획 |
| `AERODYNAMICS_ENGINEERING_PLAN.md` | 공력해석·계수 구축·CFD·동역학 검증의 상세 실행 계획 |

`simulation/`은 Ubuntu 개발 단계에서 추가할 예정입니다. 계획 문서의 파일 트리는 목표 구조이며 현재 구현 상태를 뜻하지 않습니다.

## 모델링 및 검증 원칙

- 수치마다 출처·단위·적용 조건을 기록하고 확인값과 추정값을 구분합니다.
- CAD/CFD/PX4 원본 자료를 감사한 뒤 부족한 데이터를 보완합니다.
- 원본 PX4 parameter는 분석 후 SITL에 적합한 항목만 선별해 옮깁니다.
- 첫 비행 시험부터 Gazebo ground truth와 PX4 추정 상태를 함께 기록합니다.
- 트림과 기본 응답을 확인한 뒤 자율비행, 이후 VIO로 진행합니다.
- 미션 성공만으로 실기체 동역학이 검증됐다고 판단하지 않습니다.

모델의 성숙도는 **Physics Model V0 → Measured Airframe V1 → Flight-identified Digital Twin V2**로 구분합니다.

## 원본 프로젝트와 출처

이 저장소는 [Oscilous/Adapted-Modular-Scimitar](https://github.com/Oscilous/Adapted-Modular-Scimitar)의 fork입니다.
원본은 University of Southern Denmark에서 개발한 다음 논문의 자료 저장소입니다.

> Field-tested workflow for adapting open-source 3D-printed fixed-wing UAVs for modular payload integration

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.18744058.svg)](https://doi.org/10.5281/zenodo.18744058)

- [원본 README 및 실험 설명](https://github.com/Oscilous/Adapted-Modular-Scimitar/blob/24a5d5986ae11a63882004382576460844760700/readme.md)
- 시작 upstream revision: `24a5d5986ae11a63882004382576460844760700`
- 이 fork의 초기 변경: 프로젝트 README 재구성 및 Gazebo/PX4/VIO 개발 계획 추가

원본 프로젝트는 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)으로 공개되어 있습니다. 원본 자료의 저작자 표시와 출처를 유지합니다. 추후 도입하는 외부 코드·모델에는 각 원본의 라이선스와 고지를 유지합니다.
