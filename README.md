# SMART-VISION-AMR

TurtleBot3 Waffle Pi가 저장 지도에서 A* 경로를 만들고 Nav2로 이동하는 프로젝트입니다.
Gazebo에서 먼저 시험하고, 실제 로봇과 웹 배송 관제로 이어서 사용할 수 있습니다.

**처음 사용하는 팀원:** [클론·설치부터 터미널별 실행까지 최종 매뉴얼](docs/TEAM_QUICKSTART.md)을 순서대로 따라 하세요.

경로를 따라가지 못하면 목표를 유지하고 실패한 접근 방향을 잠시 피해서 재계획합니다.
장애물 때문에 기본 3회 실패하면 웹에 대기 안내를 표시하고, 제거가 2초간 확인되면 재개합니다.
후진·재진입 조건과 조절값은 [실패 접근 재계획 안내](docs/FAILURE_REPLANNING.md)에 있습니다.
벽 근처 장애물 판별·차체 탈출 여유·Nav2 입력 확인의 변경 내용은
[팀원 주행 개선 통합 안내](docs/TEAM_NAVIGATION_INTEGRATION.md)에 정리했습니다.

## 윤현수 통합 작업 및 실차 인수인계 (2026-10-29)

이 공개 저장소는 팀원 원본 [`kiki8105/SMART-VISION-AMR`](https://github.com/kiki8105/SMART-VISION-AMR)을 기준으로 실차 웹 주행을 준비하며 확인·수정한 작업을 따로 공유합니다. 팀원 저장소에는 직접 커밋하지 않았습니다. 원본의 A*·Nav2·웹 임무·EKF 등 프로젝트 전체 코드를 포함하며, 아래는 원본 `main`의 기준 커밋 `3949060` 이후 이 통합 작업에서 더해진 변경입니다.

### 원본에서 달라진 코드

- **실차 DDS 환경 전달 수정** : 여러 실행 스크립트에서 `dds_env.bash`를 불러올 때 빈 선택 인자를 명시해, 호출자의 인자(예: 워크스페이스 경로·웹 서버 옵션)가 DDS 프로필 경로로 잘못 해석되는 문제를 막았습니다. 실차 웹 실행은 시뮬레이터/Gazebo 환경을 불필요하게 불러오지 않고 ROS와 빌드된 패키지 오버레이만 설정하도록 분기했습니다. 웹 실행 인자 전달과 실행 환경을 확인하는 셸 진입점 테스트도 추가했습니다.
- **로봇 런타임 휠 치수 인자 수정** : `WHEEL_RADIUS`, `WHEEL_SEPARATION` 값이 설정되지 않은 경우 `wheel_radius:=`, `wheel_separation:=`처럼 값 없는 ROS launch 인자를 넘기지 않도록 했습니다. 값이 설정된 경우에만 해당 인자를 전달합니다. 이를 확인하는 래퍼 테스트를 추가했습니다.
- **주행 제어 수신 진단 추가** (현재 통합 브랜치에 반영): `motion_mux` 상태에 제어 메시지 수신 횟수와 마지막 메시지 거부 사유를 표시하고, 거부된 메시지가 와도 상태를 발행하게 했습니다. 웹 명령이 수신되지 않는지, 형식·시간·순서·제어권·비상정지 래치 중 무엇 때문에 거부됐는지 구분하기 위한 진단입니다. 이 마지막 변경은 아직 테스트 실행이나 실차 검증을 하지 않았습니다.

팀원 원본에서 인수인계된 주행 동작(실패 접근 기억, A* 재계획, 정체 대기 후 장애물 제거 확인 및 재개, EKF 등)은 이 저장소에서 **이어 사용하고 시험하는 기반 코드**입니다. 위 두 커밋에서 새로 구현한 기능으로 오해하지 않도록 구분했습니다.

### 테스트한 내용과 결과

- 팀원은 원본 변경 `cdc3f6f → b49850e`를 Gazebo/SITL로 검증했다고 인수인계했습니다. 이 문서는 전달받은 결과로 기록하며, 이번 실차 점검에서 SITL 전체 회귀를 다시 돌렸다는 뜻은 아닙니다.
- 이전 실차 시험에서 사용자가 짧은 거리와 중거리 목적지까지 정상 주행했다고 여러 차례 보고했습니다. 장애물이 경로에 있을 때 우회 경로를 다시 만들다가 경로가 사라지고 멈춘 사례도 있었고, 소화기 위치를 옮긴 뒤 목적지에 도착한 사례도 있었습니다. 각 주행 로그가 모두 보존되어 있지는 않아 일반적인 성공률이나 단일 원인은 확정하지 않습니다.
- 2026-10-29에는 로봇 bringup·PC ROS 그래프·웹 상태를 점검했습니다. `/scan`은 약 10Hz로 들어왔지만, 시험 시점에는 `/follow_path` 서버와 `/amcl_pose` 발행자가 없고 웹 임무는 `PAUSED`, `health.ready=false`, localization `critical`이었습니다. 따라서 그 상태에서는 웹 목적지부터 실제 자율주행까지 이어지는 시험을 완료하지 못했습니다.
- TurtleBot3 런타임은 처음에 OpenCR(`/dev/ttyACM0`)을 열고 센서·배터리 관련 publisher와 diff-drive를 시작했으나, 이후 Dynamixel에서 `There is no status packet!`가 반복된 뒤 `turtlebot3_ros`가 종료 코드 `-6`으로 끝났습니다. 이후 배터리 토픽 publisher가 사라졌고 마지막 웹 전압은 오래된 값이었습니다. 직전 사용자가 배터리 약 29%라고 확인했습니다.
- 운영상 테스트 중단 사유는 **배터리/전원 이슈로 인한 테스트 지속 불가**입니다. 낮은 배터리 또는 부하 시 전압 강하가 유력하지만, 종료 순간 전압을 계측하지 못했으므로 OpenCR·Dynamixel·전원 접속 문제를 배제한 근본 원인 확정은 아닙니다. 충전선 연결 상태에서는 주행 명령을 보내지 않았습니다.

### 다음 담당자가 확인할 순서

1. 충분히 충전한 뒤 `/battery_state` 전압·퍼센트가 계속 갱신되는지, `turtlebot3_node`가 유지되는지, `/joint_states`, `/imu`, `/battery_state` 발행자가 있는지 확인합니다.
2. PC의 Domain 16에서 Nav2·AMCL·A*·`path_follower`가 실행 중인지 확인합니다. `/follow_path` 서버와 `/amcl_pose` 발행자, 웹 `ready`/localization 상태를 확인하기 전에는 목적지를 보내지 않습니다.
3. 충전선을 분리하고 안전한 빈 공간에서 RViz 초기 위치를 설정한 뒤 짧은 웹 목표부터 시험합니다. 이후 장애물 우회·중거리 시험으로 진행하며 웹 이벤트, A* 경로, localization health, `/cmd_vel`, 배터리, 로봇 런타임 로그를 함께 기록합니다.
4. `motion_mux`의 새 거부 사유 진단은 아직 미검증이므로, 다음 빌드에서 관련 단위 테스트와 실제 웹 제어 메시지 흐름을 확인합니다.

상세 장비 로그와 인수인계 기록은 [실차 주행 인수인계](docs/PHYSICAL_TEST_HANDOFF_2026-10-29.md)를 참고하세요.

---

## 처음 설치하기 (PC)

아래 명령은 **Ubuntu 22.04와 ROS 2 Humble이 설치된 PC**에서 실행합니다.
예시 작업공간은 `~/Team2_workspace`입니다. 저장소를 `src` 안에 두어야
제공된 스크립트가 작업공간을 올바르게 찾습니다.

**1. 필요한 프로그램 설치**

```bash
sudo apt update
sudo apt install -y git build-essential cmake python3-colcon-common-extensions gazebo \
  ros-humble-desktop ros-humble-gazebo-ros-pkgs ros-humble-nav2-bringup \
  ros-humble-xacro ros-humble-slam-toolbox ros-humble-robot-localization \
  ros-humble-compressed-image-transport \
  ros-humble-teleop-twist-keyboard \
  python3-opencv python3-pyqt5 python3-numpy python3-yaml python3-pil
```

**2. GitHub에서 코드 받기**

```bash
mkdir -p ~/Team2_workspace/src
cd ~/Team2_workspace/src
git clone https://github.com/kiki8105/SMART-VISION-AMR.git
cd SMART-VISION-AMR
```

**3. TurtleBot3 시뮬레이션 파일 준비 및 빌드**

```bash
./scripts/setup_sim.sh
```

이 스크립트가 필요한 ROBOTIS 저장소를 가져오고 `~/Team2_workspace`에서
`colcon build`를 실행합니다. 처음에는 다운로드와 빌드에 시간이 걸립니다.

## Gazebo에서 주행 시험하기

**터미널 1 — 시뮬레이터 실행**

```bash
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/run_sim.sh
```

Gazebo와 RViz가 열리고, 저장소에 포함된 예제 지도·A*·Nav2가 실행됩니다.

**터미널 2 — 자동 주행 시험**

```bash
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/check_sim.sh
```

`check_sim.sh`는 **실제로 시뮬레이션 로봇을 움직여** 경로 생성부터 도착까지
검사합니다. 종료할 때는 각 실행 터미널에서 `Ctrl+C`를 누르세요.
스크립트가 시뮬레이션용 `ROS_DOMAIN_ID=76`을 설정하므로 별도 `export`는 필요 없습니다.
Gazebo 기본 위치 추정은 기존 오도메트리(`baseline`)를 사용합니다.
시뮬레이션은 로컬 통신으로 격리합니다. 실행 스크립트의 DDS 프로필 기본값과
이전 설정으로 복귀하는 방법은 [운용 보완 안내](docs/UPGRADE_20261029.md)에 있습니다.

카메라·지도·파라미터를 PyQt 조종기로 보려면 별도 터미널에서 실행합니다.
조종기 소스는 저장소의 `tools/`에 포함되어 있습니다.

```bash
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/run_sim_controller.sh
```

자동주행의 속도 중재기(`motion_mux`)가 실행 중이면 수동 조작은 웹 조이스틱을
사용합니다. 아래 수동 SLAM에서는 PyQt 조종기로 직접 운전할 수 있습니다.

## ArUco 미로와 웹 조종기

미로 월드·마커 4개·저장 지도·웹 파일이 모두 포함되어 있습니다.
설치·빌드 후 각각 다른 터미널에서 실행하세요.

```bash
# 터미널 1: Gazebo 미로 + 저장 지도 + 주행 노드
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/run_web_mission_sim.sh
```

```bash
# 터미널 2: 웹 관제·재고 요청 사이트
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/run_web_mission.sh
```

PC는 `http://localhost:8790/`, 같은 Wi-Fi의 휴대폰은 `http://PC의_IP:8790/`으로
접속합니다. 재고 요청 화면은 주소 뒤에 `/inventory`를 붙입니다.

## 새 지도 만들기

`SLAM` 모드에서는 경로 계획을 끄고 직접 조종하며 지도를 만듭니다.

**터미널 1 — Gazebo와 SLAM**

```bash
cd ~/Team2_workspace/src/SMART-VISION-AMR
./scripts/run_sim.sh slam:=true use_motion_mux:=false
```

**터미널 2 — 키보드로 조종**

```bash
cd ~/Team2_workspace/src/SMART-VISION-AMR
source scripts/sim_env.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

키보드 조종 터미널을 활성화해 로봇을 움직입니다. 매핑을 마친 뒤
**터미널 1에서 `Ctrl+C`**를 누르면
`~/Team2_workspace/maps/slam_날짜_시간_실행번호/map.yaml`과 `map.pgm`이 저장됩니다.
저장한 지도로 다시 주행할 때는 실제 출력된 경로를 넣으세요.

```bash
./scripts/run_sim.sh map:=/home/사용자명/Team2_workspace/maps/slam_날짜_시간_실행번호/map.yaml
```

매핑 조종 방법과 ArUco 미로 준비는 [Gazebo 실습 안내](docs/GAZEBO.md)에 있습니다.
키보드 대신 같은 터미널에서 `./scripts/run_sim_controller.sh`로 PyQt 조종기를
사용할 수도 있습니다.

## 실제 로봇·웹 관제

실제 로봇은 Raspberry Pi에서 센서·모터 bringup·독립 EKF·정지 감시·속도 중재를,
PC에서 AMCL·경로 계획·웹을 실행합니다. 두 장치 모두 같은 Wi-Fi와 `ROS_DOMAIN_ID=16`을 사용합니다.
시뮬레이션의 76번 도메인과 혼용하지 마세요.
실차의 기본 실행 스크립트는 **독립 바퀴·IMU EKF**를 켭니다.
처음에는 로봇에서 저장소를 받고 `./scripts/setup_robot_runtime.sh`로 로봇용 런타임을
빌드하세요. 자세한 명령은 아래 실차 시작 안내에 있습니다.
이전 PC EKF 구성으로 복귀할 때는 양쪽에서 `SMART_VISION_ESTIMATION_LOCATION=pc`를 사용합니다.
EKF 자체를 끄는 기존 오도메트리 구성은 양쪽의 `_baseline.sh`로 선택합니다.

- [팀원용 설치·실행 순서](docs/collaboration_handoff.md)
- [실차 터미널별 실행 순서](docs/classroom_navigation_startup.md)
- [재고 요청·배송 웹 사용법](docs/inventory_delivery.md)
- [ArUco 마커 준비](docs/ARUCO.md)
- [실차 기본 EKF 실행과 기존 구성으로 복귀](docs/hardware_ekf.md)
- [조종기 DEB 설치와 실습 ZIP 생성](distribution/README.md)
- [주행·화면·안전·기록 보완 내용](docs/UPGRADE_20261029.md)
- [튜닝 전후 동일 경로 반복 시험](docs/NAVIGATION_BENCHMARK.md)

진행 흐름은 `목표 → A* 경로 → Nav2 FollowPath → /cmd_vel → 로봇`입니다.

## 출처

A* 격자 탐색, 장애물 관측/기억 및 FollowPath 연결은
[`resilient-amr-astar`](https://github.com/waterhyunsu/resilient-amr-astar)를 참고했습니다.
