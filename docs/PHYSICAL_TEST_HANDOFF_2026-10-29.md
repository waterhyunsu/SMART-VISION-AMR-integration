# SMART-VISION-AMR 실차 주행 인수인계 — 2026-10-29

## 목적과 저장소

이 문서는 팀원 전체 저장소의 주행 코드를 바탕으로 진행한 웹 기반 실차 점검을 다음 담당자에게 전달합니다.

- 공유용 통합 저장소: https://github.com/waterhyunsu/SMART-VISION-AMR-integration
- 팀원 원본: https://github.com/kiki8105/SMART-VISION-AMR
- 기준 원본 커밋: `3949060` (`upstream/main`)
- 이번 통합의 변경: DDS/실차 런치 수정, 그리고 `motion_mux` 제어 메시지 진단 변경
- 팀원 저장소에는 수정/커밋/푸시하지 않았습니다.

## 원본 코드와 통합 코드의 차이

팀원 저장소는 A*, Nav2, 웹 임무 제어, 하드웨어 런타임, EKF를 포함한 전체 프로젝트입니다. 인수인계에서 제공된 `cdc3f6f → b49850e` 변경에는 주행 정체·반복 선회 감지, 실패 구간 및 진입 방향 기억, A* 재계획, 경로 스무딩 검사, 후진·재진입 확인, 장애물 때문에 3회 실패 시 대기, 장애물 제거 2초 확인 후 자동 재개, 웹 상태 표시, AMCL 갱신 요청, 취소/성공 경합 방지 및 진단·Gazebo 시험 자료가 포함된다고 전달받았습니다. EKF 적용도 전달받았습니다. 팀원은 SITL 검증 완료라고 보고했지만, 이 인수인계 작업에서는 그 전체 검증을 재실행하지 않았습니다.

이번에 원본 기준 커밋 이후 추가한 변경은 아래와 같습니다.

### 1. DDS 프로필 인자와 실차 웹 실행 환경 

수정 파일에는 `scripts/robot_bringup_camera_16.sh`, `scripts/run_hardware_tuning.sh`, `scripts/run_hardware_tuning_controller.sh`, `scripts/run_web_mission.sh`, `scripts/sim_env.bash`가 포함됩니다.

- `source scripts/dds_env.bash` 호출 시 빈 선택 인자를 넘겨 호출자의 `$1`이 DDS 프로필 경로로 잘못 해석되지 않게 했습니다. Bash에서 `source`한 파일은 호출자의 위치 인자를 상속할 수 있기 때문입니다.
- 실차 모드의 `run_web_mission.sh`가 Gazebo용 `sim_env.bash`를 source하지 않고 ROS Humble 및 빌드된 워크스페이스만 설정하게 했습니다. 시뮬레이션 모드는 이전처럼 별도 분기를 사용합니다.
- 셸 진입점 테스트를 추가해 물리 웹 실행, 런치 인자 전달, DDS 기본 프로필 흐름을 모의 환경에서 확인하도록 했습니다.

### 2. 비어 있는 휠 기하 런치 인자 제거 

`robot_bringup_camera_16.sh`에서 `WHEEL_RADIUS`, `WHEEL_SEPARATION`이 비어 있을 때 `wheel_radius:=` / `wheel_separation:=`를 넘기지 않게 했습니다. 환경변수가 실제 값으로 지정된 경우에만 대응 런치 인자를 추가합니다. 잘못된 빈 launch argument가 전달되지 않는지 모의 래퍼 테스트도 추가했습니다.

### 3. `motion_mux` 명령 거부 진단 변경

`motion_mux.py`의 상태에 `control_received_count`와 `control_rejected_reason`을 추가했습니다. payload 형식, 순번, 송신 시각, 폐기된 source, 속도 값, 비상정지 래치, 다른 source의 수동 제어권 등 메시지 거부 원인을 구체적으로 기록합니다. 메시지가 거부돼도 상태를 publish하도록 바꿨습니다. 대응 단위 테스트도 추가했습니다.

이 진단 변경은 이번 통합 저장소에 포함되지만, 이번 인수인계 시점에는 테스트를 실행하지 않았고 실차 검증도 하지 않았습니다. 다음 PC 빌드 후 테스트/웹 메시지 흐름으로 확인해야 합니다.

## 실차 시험 결과

### 이전 주행에서 사용자에게 보고받은 결과

- 짧은 주행과 중거리 주행은 여러 차례 정상 완료했다고 보고됐습니다. 일부 시험에서는 도착 방향 정렬도 정상이라고 보고됐습니다.
- 장애물이 경로상에 있을 때 경로 재계산이 반복되다 A* 경로가 사라지고 로봇이 멈춘 사례가 있었습니다. 소화기 위치를 조금 옮긴 다음 목적지까지 도달한 사례도 있습니다.
- 이 결과는 반복적인 사용자 관찰이며 모든 시점의 로그가 남아 있지는 않습니다. 그러므로 “항상 정상” 또는 장애물 안전 여유 하나가 원인이라고 확정할 수 없습니다.

### 2026-10-29 점검 자료

- 웹 페이지 `http://192.168.0.199:8790/`의 `/api/state`에서 연결됨과 ROS Domain 16을 확인했습니다. 웹 서버 프로세스는 살아 있었습니다.
- 웹 mission은 `PAUSED`, `health.ready=false`, localization 상태는 `critical`, `autonomous_allowed=false`였습니다.
- PC ROS 그래프에서 `/follow_path` action server/client는 0개였고 `/amcl_pose` publisher도 0개였습니다. 점검 당시 `/amcl`, `/map_server`, `/controller_server`, `astar_planner`, `path_follower` 노드가 없었습니다. 따라서 웹에서 실제 목적지 주행 시험을 수행할 수 있는 준비 상태가 아니었습니다.
- `/scan`은 약 10Hz로 수신되어 ROS 통신 전체 단절은 아니었습니다. `/cmd_vel`에는 `motion_mux` publisher가 있었지만 로봇 구동 노드 subscriber는 0개였습니다.
- `/joint_states`, `/imu`, `/battery_state` publisher가 없었습니다. `/odom`의 EKF 추정은 위치 공분산이 약 `7.7e9`까지 커졌고 localization health가 `odometry linear velocity covariance too large` 사유로 반복 거부했습니다.

## 중단 원인

실차 테스트는 **배터리/전원 이슈로 인해 지속할 수 없어 중단**했습니다.

- Pi 로그에서 런타임이 `/dev/ttyACM0`을 열고 초기에는 센서·배터리 관련 publisher 및 diff-drive를 시작한 기록이 있었습니다.
- 이후 16:10경 Dynamixel 통신에서 `There is no status packet!`가 반복되고 `turtlebot3_ros`가 종료 코드 `-6`으로 끝났습니다.
- 뒤이어 배터리 토픽 publisher가 사라졌고 웹에 표시된 마지막 전압 `10.99V`는 오래된 값이었습니다. 직전 사용자가 관찰한 배터리 퍼센트는 약 29%였습니다.
- `/dev/ttyACM0` 자체는 남아 있었고, 명확한 USB 분리 기록도 확인되지 않았습니다.

따라서 운영상의 테스트 중단 사유는 배터리/전원 이슈로 기록합니다. 낮은 배터리 또는 부하 시 전압 강하가 유력한 원인이지만, 종료 시점 전압을 계측하지 않아 OpenCR·Dynamixel·전원 접속 문제까지 배제한 근본 원인은 확정되지 않았습니다. 충전 케이블이 연결된 동안 주행 명령은 보내지 않았습니다.

## 다음 시험 순서

1. 배터리를 충분히 충전하고 주행 케이블을 분리한 뒤 시작합니다.
2. 로봇에서 `/battery_state`가 계속 발행되고 전압/퍼센트가 갱신되는지 확인합니다. `turtlebot3_node`, `/joint_states`, `/imu`가 안정적으로 유지되고 Dynamixel `no status packet` 오류가 없는지 봅니다.
3. PC에서 Domain 16의 Nav2/AMCL/A*/`path_follower`를 실행합니다. `/follow_path` server 1개, `/amcl_pose` publisher 1개, 웹 `ready=true`, localization `autonomous_allowed=true`를 확인하기 전에는 목표를 보내지 않습니다.
4. RViz에서 실제 로봇의 2D Pose Estimate를 지정하고 TF와 센서 시간 갱신을 확인합니다.
5. 웹 UI에서 가까운 빈 공간 목표부터 시험하고, 다음으로 장애물 우회 및 중거리 주행을 진행합니다. 웹 이벤트, A* 경로, `/cmd_vel`, 오도메트리, localization health, 배터리와 로봇 로그를 같은 시간대에 저장합니다.
6. 새 `motion_mux` 진단 변경은 먼저 빌드 후 관련 단위 테스트를 실행하고, 웹 제어 패킷이 수신/승인/거부되는지 상태 메시지로 확인합니다.

## 실행 환경 참고

- Raspberry Pi: `robot@192.168.0.187`, TurtleBot3 bringup 런치
- PC ROS Domain: `16`
- 웹 운용: `http://192.168.0.199:8790/`
- 재고 화면: `http://192.168.0.199:8790/inventory`
- 웹 systemd 사용자 서비스: `smart-vision-classroom-web.service`
- 팀원이 추가한 로봇 실행 래퍼: SSH 후 `./turtlebot16` (실행 전 현재 체크아웃과 권한을 확인)

세부 터미널 명령은 저장소의 `docs/TEAM_QUICKSTART.md`를 기준으로 확인하고, 이전 워크스테이션의 경로를 그대로 가정하지 마세요.
