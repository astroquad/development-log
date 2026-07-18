# Astroquad 차기 팀 실비행 온보딩 가이드

> 대상: Astroquad 프로젝트와 전체 미션은 처음인 차기 팀장 및 신규 팀원
>
> 문서 기준: 2026-07-18 저장소 상태
>
> 목표: 새 로직을 먼저 만드는 것이 아니라, Gazebo에서 동작하는 현재 미션을 **실기체에서 짧고 반복 가능한 시험으로 검증하고 로그 기반으로 조정**한다.

## 1. 먼저 이해할 것

Astroquad의 전체 목표는 다음과 같다.

```text
수직이착륙장 ArUco 위에서 이륙
  -> 기체 방향 정렬 및 격자 진입
  -> 라인과 교차점을 따라 snake 방식으로 격자 탐색
  -> 탐색 중 ArUco marker 위치 기록
  -> 기록한 marker를 지정 순서로 재방문
  -> 격자 원점과 수직이착륙장으로 복귀
  -> marker 위 정렬 후 착륙
```

시스템은 두 프로그램으로 나뉜다.

| 위치 | 프로그램 | 책임 |
|---|---|---|
| Windows 노트북 | `uav-gcs`의 `astroquad-gcs.exe` | 영상·텔레메트리 수신, 라인/교차점/마커 오버레이, 격자와 상태 관찰 |
| 드론 Raspberry Pi 5 | `uav-onboard`의 `astroquad-onboard` 또는 `line_follow_node` | 카메라 인식, 미션 판단, Pixhawk MAVLink 제어, 안전 감시, 비행 로그 저장 |
| Pixhawk 6C Mini | ArduCopter | 자세·모터·EKF·광학 흐름/거리 센서 융합, 비행 모드와 FC failsafe |

GCS는 미션을 판단하거나 시작/중단 명령을 보내는 프로그램이 아니다. 현재 GCS command channel은 구현되지 않았다. 미션은 Pi 터미널에서 시작하며, 비상 개입은 **조종기 모드 전환/takeover와 FC 절차가 최우선**이다.

## 2. 현재 진행 상황과 인수 기준선

현재 코드 기준으로 다음은 구현되어 있다.

- Pi 5 + IMX296 하향 카메라 입력과 라인·교차점·ArUco 인식
- `line_follow_node`의 라인 추종, 마커 접근/호버, 착륙 staging 미션
- `astroquad-onboard`의 격자 진입, snake 탐색, marker 기록/재방문, 원점·수직이착륙장 복귀 상태 머신
- Gazebo/SITL 전체 격자 미션
- 실기체용 ArduPilot serial/GUIDED velocity 경로와 RC·local-estimate preflight gate
- GCS 영상/텔레메트리 오버레이와 LTE 크기의 기본 전송 설정
- 실행별 `meta.json`, `frames.csv`, `events.jsonl` 비행 로그
- 영상 정지, heartbeat 손실, 미션/상태 timeout 등의 안전 감시

앞으로의 중심 업무는 위 기능을 새로 설계하는 것이 아니라 다음 반복이다.

```text
한 가지 검증 질문 설정
  -> 짧고 안전한 실비행
  -> 육안 관찰 + GCS 화면 + onboard 로그 + Pixhawk 로그 수집
  -> 실패가 시작된 정확한 시각/상태 특정
  -> Codex에 증거와 기대 동작 전달
  -> 최소 변경 및 테스트
  -> SITL 회귀 확인
  -> 같은 조건으로 다시 실비행
```

주의할 현재 기준선:

- 현 플랫폼은 Pi 5, Pixhawk 6C Mini, IMX296, MTF-01, 4S/S500 V2다. 예전 Pi 4/Pixhawk 1/IMX519 설정은 사용하지 않는다.
- 지원 제어 경로는 `GUIDED` body-frame velocity다. `ALT_HOLD + RC_CHANNELS_OVERRIDE` 설정은 과거 기체용이며 폐기되었으므로 비행에 사용하지 않는다.
- 2026-07-18에 주요 Markdown을 현재 소스·설정 기준으로 정합화했다. 현재
  시스템 범위는 [`SYSTEM_SPEC.md`](SYSTEM_SPEC.md), 실행 방법은
  [`uav-onboard/README.md`](../uav-onboard/README.md), 실제 판정은 소스·테스트와
  실행별 로그를 우선한다.
- 실기체용 [`runtime.ardupilot_serial.toml`](../uav-onboard/config/runtime.ardupilot_serial.toml)의 serial device에는 현재 `CHANGE-ME`가 남아 있다. 아래 절차로 실제 Pixhawk 경로를 확인해 수정하기 전에는 arm/takeoff 명령을 실행하지 않는다.

## 3. 문서를 언제 볼지

코드 구조를 이 문서에 다시 복제하지 않는다. 상황별로 다음 문서를 찾는다.

| 상황 | 먼저 볼 문서 |
|---|---|
| 전체 목표, 하드웨어, 현재 구현 범위 확인 | [`SYSTEM_SPEC.md`](SYSTEM_SPEC.md) — 현행 기준 문서 |
| Pi 빌드·실행 옵션, 카메라, SITL, 실기체 명령, 로그 경로 | [`uav-onboard/README.md`](../uav-onboard/README.md) |
| onboard에서 한 프레임이 미션·제어 명령으로 가는 흐름 파악 | [`uav-onboard/ARCHITECTURE_OVERVIEW.md`](../uav-onboard/ARCHITECTURE_OVERVIEW.md) |
| onboard 모듈 책임과 설정·알고리즘의 기준 확인 | [`uav-onboard/PROJECT_SPEC.md`](../uav-onboard/PROJECT_SPEC.md) |
| GCS 빌드·실행·방화벽·화면 문제 해결 | [`uav-gcs/README.md`](../uav-gcs/README.md) |
| GCS가 받은 데이터가 화면에 표시되는 흐름 파악 | [`uav-gcs/ARCHITECTURE_OVERVIEW.md`](../uav-gcs/ARCHITECTURE_OVERVIEW.md) |
| onboard↔GCS 필드나 packet 의미 확인 | [`uav-onboard/docs/PROTOCOL.md`](../uav-onboard/docs/PROTOCOL.md) |
| 과거에 겪은 증상과 해결책 검색 | [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md)에서 증상 키워드 검색. 각 항목 당시의 구 플랫폼/명칭에 주의 |
| Gazebo world와 SITL 재현 | [`uav-onboard/sim/gazebo/README.md`](../uav-onboard/sim/gazebo/README.md) |
| 기체 교체 사유와 하드웨어 리스크 확인 | [`HARDWARE_UPGRADE.md`](HARDWARE_UPGRADE.md) — 구매 전 판단을 보존한 역사 문서 |

[`MVP_PLAN.md`](MVP_PLAN.md), [`PLAN.md`](PLAN.md), [`RESEARCH.md`](RESEARCH.md)는
완료된 과거 milestone/조사 기록이다. 현재 할 일 목록이나 실행 명령으로 쓰지
말고, 결정 배경을 추적할 때만 참고한다.

문서끼리 상태가 다르면 최근 Git 이력, 현재 README, 설정, 소스, 테스트 순으로 대조한다. 오래된 문서만 근거로 동작을 단정하지 않는다.

## 4. 최초 1회 온보딩

### 4.1 Windows GCS 준비

PowerShell에서 다음을 실행한다.

```powershell
cd C:\path\to\astroquad\uav-gcs
cmake -S . -B build
cmake --build build --config Release
```

실행 파일이 `build`에 생겼으면:

```powershell
.\build\astroquad-gcs.exe --config config
```

Visual Studio multi-config 빌드로 `build\Release`에 생겼으면:

```powershell
.\build\Release\astroquad-gcs.exe --config config
```

Windows 관리자 PowerShell에서 방화벽 설정을 한 번 적용한다.

```powershell
powershell -ExecutionPolicy Bypass -File scripts\setup_windows_firewall.ps1
```

이 스크립트는 GCS 수신용 UDP 14550(telemetry), 5600(video)을 허용한다.

### 4.2 Tailscale 연결과 SSH

현재 고정 주소는 다음과 같다.

| 장치 | Tailscale 주소 |
|---|---|
| Windows GCS 노트북 | `100.85.239.73` (`gcs-laptop`) |
| Raspberry Pi 5 | `100.101.84.47` (`pi5`) |

두 장치를 같은 tailnet에 연결한 뒤 Windows PowerShell에서 확인한다.

```powershell
tailscale status
tailscale ping 100.101.84.47
ping 100.101.84.47
ssh astroquad@100.101.84.47
```

저장소의 기존 Pi 경로가 `/home/astroquad/astroquad/uav-onboard`이므로 위 명령은 계정 `astroquad`를 기준으로 한다. Pi 계정이 바뀌었다면 실제 계정명만 바꾼다. 연결 후 Pi에서 역방향을 확인한다.

```bash
tailscale status
tailscale ping 100.85.239.73
```

가능하면 `tailscale ping` 결과가 `direct`인지 본다. `via DERP`이거나 손실이 있으면 영상만 보고 코드를 의심하지 말고 링크 문제부터 분리한다.

### 4.3 Pi 코드와 의존성 준비

```bash
cd ~/astroquad/uav-onboard
git status --short
git rev-parse HEAD
bash scripts/setup_rpi_dependencies.sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

현장 직전에는 무조건 `git pull`하지 않는다. 팀이 검증할 commit을 먼저 정하고, 변경 사항이 없는지 확인한 후 그 commit을 Pi와 노트북에 맞춘다. 매 시험 기록에 `git rev-parse HEAD` 결과를 남긴다.

### 4.4 Pixhawk serial 경로 확정 — 현재 필수 작업

Pixhawk를 Pi에 연결하고:

```bash
ls -l /dev/serial/by-id/
```

출력된 Pixhawk 6C Mini 경로를 아래 두 파일의 `device` 값에 반영하고 서로 같게 유지한다.

- [`config/runtime.ardupilot_serial.toml`](../uav-onboard/config/runtime.ardupilot_serial.toml)
- [`config/pixhawk6c_indoor_flow.expected.toml`](../uav-onboard/config/pixhawk6c_indoor_flow.expected.toml)

현재 저장소 값 `usb-ArduPilot_Pixhawk6C_CHANGE-ME-if00`는 자리표시자다. 실제 경로를 확인하지 않은 채 `--allow-arm-takeoff`를 사용하지 않는다.

## 5. 매 실비행일 공통 순서

### 5.1 시험 계획을 먼저 한 줄로 제한한다

좋은 예:

- “이륙 후 10초 동안 흰 라인의 lateral offset이 진동 없이 줄어드는가?”
- “마커가 가려졌다 다시 보일 때 전진 명령이 재개되지 않고 hover로 복귀하는가?”
- “첫 T 교차점에서 정지 후 우회전 상태로 정확히 한 번 전이하는가?”

한 번에 gain, 속도, threshold, 카메라 exposure를 모두 바꾸지 않는다. 독립 변수 하나, 성공 기준 하나, 중단 기준 하나를 기록한다.

### 5.2 하드웨어·안전 gate

최소 다음을 현장에서 확인한다.

- 프로펠러, 모터 고정, 프레임, 카메라 하향 고정과 렌즈 초점
- 배터리 상태와 GCS/Pixhawk battery telemetry
- MTF-01 range/optical-flow 품질과 안정적인 `LOCAL_POSITION_NED`
- 조종기 입력 freshness, 모드 스위치 takeover, 수동 착륙/disarm
- 비행장 통제, 안전거리, spotter, 즉시 중단 기준 공유
- props-off 모터 순서·회전 방향 검증과 수동 hover 완료
- 실제 경기장의 선 색/폭/반사, 조도, 마커 ID·배치 기록
- GCS와 Pi가 같은 검증 commit·config를 사용함

자동 비행 전에 Pi에서 no-arm 검사를 실행한다.

```bash
cd ~/astroquad/uav-onboard

./build/mavlink_probe --config config --target ardupilot_serial \
  --duration-ms 12000 --strict-local-estimate

./build/mavlink_probe --config config --target ardupilot_serial \
  --duration-ms 30000 --strict-rc

./build/line_follow_node --config config --target ardupilot_serial \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

어느 하나라도 실패하면 그날의 자동 arm/takeoff를 진행하지 않고 원인을 분리한다. `--unsafe-assume-rc-present`로 gate를 우회하는 것은 정상 실비행 절차가 아니다.

### 5.3 프로그램 시작 순서

순서는 항상 다음과 같다.

1. Windows에서 `astroquad-gcs.exe`를 먼저 실행한다.
2. GCS가 telemetry 14550, video 5600을 기다리는 상태인지 확인한다.
3. Tailscale 양방향 ping을 확인한다.
4. SSH로 Pi에 접속한다.
5. Pi에서 **이번 시험에 맞는 프로그램 하나만** 실행한다.
6. GCS 영상·오버레이와 현장 spotter를 동시에 관찰한다.

비행 중 `vision_debug_node`, `line_follow_node`, `astroquad-onboard`를 동시에 실행하지 않는다. 같은 telemetry/video source가 섞여 진단을 망칠 수 있다.

## 6. 어떤 프로그램을 실행할지

### 6.1 무장 없이 카메라·인식·GCS만 확인

```bash
cd ~/astroquad/uav-onboard
./build/vision_debug_node --config config \
  --line-mode light_on_dark --video
```

기체를 손으로 안전하게 이동하며 GCS에서 다음을 본다.

- 라인 contour와 tracking point가 실제 선 위에 있는가
- 교차점 종류와 branch ray가 실제 모양과 맞는가
- ArUco box/ID/중심이 안정적인가
- 영상이 없고 log만 오면 `--video` 누락 여부
- `[video-rx]`의 `incomplete`가 빠르게 증가하면 네트워크 포화 여부

### 6.2 첫 실비행: 라인 추종 + 마커 호버만 시험

전체 격자를 만들기 전의 짧은 staging 시험에는 반드시 `line_follow_node`를 쓴다.

```bash
cd ~/astroquad/uav-onboard
./build/line_follow_node --config config --target ardupilot_serial \
  --vision rpicam \
  --line-mode light_on_dark \
  --video \
  --control-backend guided_velocity \
  --allow-arm-takeoff
```

`guided_velocity`는 기본값이므로 해당 옵션은 생략할 수 있지만, 인수 초반에는 실행 기록을 명확히 하기 위해 적어 두는 편이 좋다. 이 프로그램은 라인 추종, 마커 접근/호버, 착륙까지만 검증하기 위한 것이며 전체 격자 미션 결과로 해석하지 않는다.

### 6.3 전체 격자 경기장 시험

격자 진입, snake 탐색, marker 재방문, 복귀까지 시험할 때는 `line_follow_node`가 아니라 `astroquad-onboard`를 쓴다.

```bash
cd ~/astroquad/uav-onboard
./build/astroquad-onboard --config config --target ardupilot_serial \
  --line-mode light_on_dark \
  --marker-count 4 \
  --revisit-order desc \
  --video \
  --allow-arm-takeoff
```

각 값의 의미:

| 값 | 의미 |
|---|---|
| `--target ardupilot_serial` | Gazebo가 아닌 실제 Pixhawk serial 사용 |
| `--line-mode light_on_dark` | 어두운 바닥 위 밝은/흰 라인. 실제 경기장 극성이 다르면 사전 vision smoke 후 변경 |
| `--marker-count 4` | 수직이착륙장 marker를 제외한 격자 marker 예상 개수. 실제 설치 개수와 반드시 일치시킴 |
| `--revisit-order desc` | 발견 marker ID 내림차순 재방문. 현재 기본값도 `desc`지만 기록 명확성을 위해 명시 |
| `--video` | GCS debug video 송출. 미션 로직 자체는 GCS/video 손실에 의존하지 않음 |
| `--allow-arm-takeoff` | 실제 serial target에서 arm/takeoff를 허용하는 명시적 안전 승인 |

Tailscale 실기체에서는 GCS 목적지가 config의 `gcs-laptop`이므로 `--gcs-ip`를 넣지 않는다. 특히 `--gcs-ip pi5` 또는 `100.101.84.47`을 넣으면 Pi가 자기 자신에게 보내므로 GCS에 아무것도 오지 않는다.

LTE 기본 전송값은 600 px, JPEG quality 55, video 12 fps, telemetry 6 fps로 설정되어 있다. 링크를 측정하지 않은 상태에서 무작정 높이지 않는다. 포화되면 미션 로직과 영상 품질을 구분하고 우선 `--fps 8`처럼 video rate만 낮춰 비교한다.

### 6.4 중단과 비상 절차

비정상 이동 시 우선순위는 다음과 같다.

1. 조종기 mode switch/takeover
2. 조종기를 통한 착륙·disarm 또는 사전 합의된 FC 비상 절차
3. 프로세스와 MAVLink가 정상일 때만 SSH 터미널의 `Ctrl+C`

두 실비행 프로그램 모두 `SIGINT/SIGTERM` 종료 시 LAND를 요청하도록 구현되어 있지만, SSH·Pi·MAVLink가 끊긴 상황에는 의존할 수 없다. GCS도 command sender가 아니므로 GCS 창을 닫는 것은 비상 착륙 명령이 아니다.

## 7. 비행이 끝난 뒤 어디를 볼지

### 7.1 onboard 구조화 로그

Pi의 `uav-onboard` 기준으로 자동 저장된다. `--no-flight-log`를 넣지 않는 한 기본 활성이다.

```text
전체 격자 미션:
uav-onboard/logs/flights/run_NNNN_YYYY-MM-DD_HH-MM-SS/

라인/마커 staging:
uav-onboard/logs/flights/line_follow_node/run_NNNN_YYYY-MM-DD_HH-MM-SS/
```

최신 실행을 찾는다.

```bash
cd ~/astroquad/uav-onboard
ls -dt logs/flights/run_* 2>/dev/null | head -1
ls -dt logs/flights/line_follow_node/run_* 2>/dev/null | head -1
```

각 폴더의 의미:

| 파일 | 먼저 확인할 내용 |
|---|---|
| `meta.json` | 실제 argv, target, marker count/order, 주요 mission/video/camera/network 설정, 시작 시각 |
| `events.jsonl` | 상태 전이와 이유, node commit, marker 발견/재방문, safety event, 종료 이유 |
| `frames.csv` | 약 20 Hz 시계열. 인식값, 미션 상태, 위치/고도/yaw/flow/battery, 실제 보낸 속도·yaw-rate 명령 |

첫 분석 순서는 `meta.json` → `events.jsonl` → 문제 시각 주변의 `frames.csv`다. CSV 전체를 눈으로 처음부터 읽지 않는다.

전체 미션 `frames.csv`의 주요 열은 `state`, `intent`, `coord_x/y`, `heading`, `hop_m`, `line_detected`, `line_offset_norm`, `line_angle_err_rad`, `line_conf`, `idec_state/type`, `branch_mask`, `markers_visible`, `marker_err_x/y`, `mode`, `armed`, `agl_m`, `local_x/y`, `vxy_mps`, `yaw_rad`, `flow_q`, `batt_v`, `cmd_vx/vy/vz`, `cmd_yaw_rate_rads`, `vision_latency_ms`, `safety_event`다.

### 7.2 GCS와 Pixhawk 자료

- GCS 화면은 현재 persistent log subsystem이 아니다. 시험 중 화면 녹화 또는 휴대폰 영상으로 overlay와 로그 pane을 남긴다.
- spotter는 “오른쪽으로 약 0.5 m 밀림”, “yaw가 두 번 튐”처럼 방향·크기·시각을 말로 기록한다.
- Pixhawk 자세/EKF/failsafe/모드 문제는 Mission Planner에서 해당 비행의 DataFlash `.BIN` 로그를 내려받아 함께 보관한다.
- onboard의 명령이 잘못됐는지, FC가 올바른 명령을 다르게 수행했는지를 구분하려면 `frames.csv`와 `.BIN`의 같은 시각을 비교한다.

### 7.3 Windows 작업공간으로 로그 복사

raw 로그는 Git에 커밋하지 않고 `uav-onboard/logs/` 아래에 보관하면 현재 ignore 규칙을 따른다. 예:

```powershell
$TestId = "2026-07-18_line_01"
New-Item -ItemType Directory -Force ".\uav-onboard\logs\imported\$TestId" | Out-Null
scp -r astroquad@100.101.84.47:~/astroquad/uav-onboard/logs/flights/line_follow_node/run_0001_YYYY-MM-DD_HH-MM-SS `
  ".\uav-onboard\logs\imported\$TestId\"
```

전체 미션이면 원격 경로에서 `line_follow_node/`만 뺀다. `run_...` 이름은 실제 `ls -dt` 결과로 교체한다.

## 8. 실비행 후 디버깅 파이프라인

### 8.1 먼저 사람이 시험 카드를 작성한다

아래 템플릿을 매 비행마다 채운다.

```markdown
# TEST-ID: YYYY-MM-DD_목적_번호

- Git commit (Pi / GCS):
- 실행 명령 전체:
- 변경한 config와 diff:
- 시험 질문:
- 성공 기준:
- 즉시 중단 기준:
- 장소/바닥/라인 색·폭·재질:
- 조도/날씨/바람:
- marker ID와 실제 배치:
- 배터리 시작/종료 전압:
- 육안 관찰과 발생 시각:
- GCS 관찰과 발생 시각:
- 최종 상태 및 착륙/disarm 방식:
- onboard run 폴더:
- Pixhawk .BIN:
- 영상 파일:
```

“라인 추종이 이상함”만 남기지 말고, 기대 동작과 실제 동작의 첫 차이를 시간과 방향으로 적는다.

### 8.2 증거를 맞춰 원인 층을 분리한다

| 관찰 | 우선 확인할 값 | 의심 층 |
|---|---|---|
| GCS 영상만 끊기고 기체는 정상 | `[video-rx] incomplete/completed`, Tailscale direct/DERP | 네트워크/debug video |
| 실제 라인은 보이는데 overlay가 사라짐 | `line_detected`, `line_conf`, exposure, threshold, 마스크 | camera/vision |
| overlay는 맞는데 기체가 반대로 이동 | `line_offset_norm`, `cmd_vy_mps`, `invert_lateral`, 기체 축 | control mapping |
| 명령은 부드러운데 기체가 흔들림 | `cmd_*` 대 Pixhawk attitude/velocity, EKF, flow quality | FC/EKF/기체 튜닝 |
| 교차점에서 잘못 회전 | `idec_state/type`, `branch_mask`, state transition reason | intersection/mission decision |
| 마커를 보고도 통과 | marker error, detection hold, state, `cmd_vx/vy` | marker stabilizer/mission gate/control |
| 갑자기 LAND | `events.jsonl`, `safety_event`, mode, heartbeat, vision latency, battery | safety/failsafe |
| GCS에 아무것도 안 옴 | GCS 선실행, Windows firewall, 목적지가 `gcs-laptop`인지 | 실행 순서/network |

가장 중요한 질문은 “코드가 어떤 명령을 냈는가?”와 “기체가 그 명령대로 움직였는가?”를 분리하는 것이다.

### 8.3 Codex에 줄 입력 묶음

Codex가 추측하지 않게 최소 다음을 함께 준다.

1. 시험 카드
2. 정확한 commit hash와 실행 명령
3. 해당 run의 `meta.json`, `events.jsonl`
4. 실패 전 5초부터 실패 후 5초까지의 `frames.csv` 구간 또는 전체 CSV
5. GCS/현장 영상에서 문제 시각과 육안 이동 방향
6. 가능하면 같은 구간의 Pixhawk `.BIN`
7. 기대 동작과 변경 허용 범위

권장 요청 예시:

```text
TEST-ID 2026-07-18_line_01을 분석해줘.
기대 동작은 라인 중심으로 수렴하면서 0.20 m/s 전진하는 것이고,
실제 기체는 영상 00:18부터 오른쪽으로 진동했다.

실행 commit과 명령은 시험 카드에 있고, onboard 로그는
uav-onboard/logs/imported/2026-07-18_line_01/ 아래에 있다.
00:13~00:23의 frames.csv와 events.jsonl, 현장 관찰을 서로 대조해
1) 최초 이상 징후,
2) vision/control/FC 중 가장 가능성 높은 층,
3) 근거가 되는 열과 시각,
4) 한 번에 하나만 바꾸는 최소 수정안을 제시해줘.

수정이 필요하면 기존 구조를 유지하고 관련 테스트를 추가/수정한 뒤,
전체 테스트와 SITL 회귀까지 실행해줘. 확인되지 않은 원인은 단정하지 말아줘.
```

Codex에게 곧바로 “gain을 고쳐줘”라고 하지 않는다. 먼저 로그로 vision error, command, 실제 motion 중 어느 단계에서 차이가 시작됐는지 찾게 한다.

### 8.4 수정 후 다시 비행하기 전

- `git diff`로 의도한 한 가지 변경만 있는지 확인
- 관련 unit test와 전체 `ctest` 통과
- 라인/교차점/마커 변경이면 저장 영상 또는 offline tool로 회귀 확인
- 미션/control/safety 변경이면 Gazebo의 해당 구간과 전체 격자 미션 회귀
- 새 commit hash와 config diff 기록
- 이전 시험과 같은 물리 조건, 같은 명령, 같은 성공 기준으로 재시험

결과가 좋아도 한 번으로 확정하지 않는다. 최소 여러 회 반복하여 같은 상태 전이, 오차 감소, 착륙 결과가 재현되는지 본다.

## 9. 차기 팀장 운영 원칙

- 비행마다 책임자 1명, 조종기 담당 1명, 관찰/기록 담당 1명을 정한다.
- “오늘 바꾼 한 가지”와 “오늘 확인할 한 가지”를 모두가 알고 시작한다.
- raw 로그·영상은 보존하고, Git에는 코드/config/짧은 시험 카드와 결론만 의도적으로 반영한다.
- 실패한 비행도 지우지 않는다. 재현 조건과 실패 시각이 가장 가치 있는 데이터다.
- GCS 영상 손실을 미션 실패로, 또는 overlay 성공을 실제 제어 성공으로 혼동하지 않는다.
- 안전 gate를 코드 편의를 위해 제거하지 않는다. gate가 막히면 센서·RC·EKF 조건을 해결한다.
- 실비행에서 발견한 현상은 [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md)에 증상, 원인, 증거, 수정, 검증 상태 순서로 축적한다.
- `README`의 실행 명령이나 실제 로그 구조가 바뀌면 이 문서도 같은 PR/commit에서 갱신한다.

## 10. 현장용 초단축 체크리스트

```text
[ ] 오늘의 commit / config / 시험 질문 / 중단 기준 기록
[ ] 기체·배터리·카메라·MTF-01·RC takeover·수동 hover 확인
[ ] Windows Tailscale 연결, GCS 방화벽, astroquad-gcs 먼저 실행
[ ] Pi SSH 접속, Tailscale 역방향 direct 확인
[ ] mavlink_probe local estimate / RC, line_follow mavlink smoke 통과
[ ] vision_debug_node로 실제 바닥의 라인·교차점·마커 인식 확인 후 종료
[ ] 짧은 라인+마커 시험이면 line_follow_node 하나만 실행
[ ] 전체 격자 시험이면 astroquad-onboard 하나만 실행
[ ] 비정상 이동 시 RC takeover 우선
[ ] 착륙·disarm 확인 후 onboard run 폴더 즉시 식별
[ ] meta → events → 문제 구간 frames 순으로 확인
[ ] 현장 영상·GCS·Pixhawk BIN·시험 카드를 같은 TEST-ID로 묶기
[ ] Codex 분석 → 최소 변경 → test/SITL → 동일 조건 재비행
```
