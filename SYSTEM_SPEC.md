# Astroquad System Spec

최종 업데이트: 2026-07-07 (실기 전환 리팩터: LTE-sized 전송 기본값,
vision-stale 워치독 + per-state 행 가드, per-run 비행 로깅, 배터리 ingest)

이 문서는 `uav-onboard`와 `uav-gcs`를 모두 포함하는 전체 시스템 기준
문서다. Repo별 세부 책임과 실행 방법은 각 repo의 `PROJECT_SPEC.md`와
`README.md`를 따른다.

## 1. 시스템 목표

Astroquad는 GPS 없는 실내/운동장 격자 환경에서 UAV가 하향 카메라 기반
라인 인식, 교차점 판단, ArUco marker 인식을 사용해 탐색 임무를 수행하는
C++ 기반 시스템이다.

최종 목표:

```text
이륙
  -> grid 진입
  -> snake 방식 grid 탐색
  -> 교차점별 ArUco marker 기록
  -> marker 번호 역순 재방문
  -> 출발점 복귀
  -> 자동 착륙
```

현재 구현 기준:

- Vision/GCS path는 재사용 가능한 라이브러리로 분리되어 있다.
- `line_follow_node`는 SITL 및 guarded ArduPilot serial line-follow staging을
  담당한다 (RC-override 백엔드는 폐기됨 — 축소/은퇴 예정).
- `astroquad-onboard`는 현재 grid arena snake mission의 온보드 메인
  runtime이다.
- `astroquad-gcs`는 현재 GCS 메인 UI다.

## 2. 하드웨어 기준

2026-07 업그레이드 플랫폼 (1차 예선 통과 지원금으로 전면 교체, 이륙중량 약
2.1 kg). FC 파라미터 타깃: `uav-onboard/config/pixhawk6c_indoor_flow.params`.
구 플랫폼(Pixhawk 1 / Pi 4 / IMX519 / 3S / 1.5 kg)은 퇴역.

| 모델/장치 | 연결 위치 | 역할 |
|---|---|---|
| Holybro Pixhawk 6C Mini (ArduCopter) | 기체 중앙 | 비행 제어기. 자세 안정화, 모터 출력, EKF3 센서 융합, 모드 관리 |
| MicoAir MTF-01 Optical Flow & Range Sensor | FC TELEM1 (MAVLink) | GPS 없이 수평 속도 추정 + 바닥 거리 측정 (내장 TOF rangefinder 겸용) |
| Raspberry Pi 5 (4 GB, Active Cooler) | FC USB 또는 UART | OpenCV 비전, mission state, MAVLink command sender |
| Raspberry Pi Global Shutter Camera (IMX296 mono) | Raspberry Pi CSI | 하향 영상 입력. 모노 글로벌셔터, 고정초점 CS-mount (렌즈 FOV 실측 필요) |
| Holybro S500 V2 frame kit + companion plate | — | 기체 프레임 |
| Sunnysky X2212 980KV CW/CCW + Tarot 9450 folding props | FC MAIN OUT → ESC | 추진계. 2.1 kg에서 호버 스로틀 약 55–62% 예상 (호버 로그로 확인) |
| GT-Drone 35A BLHeli_S OPTO ESC (2S–5S) | PDB | 모터 구동 (OPTO: ESC 텔레메트리 없음 → 노치필터는 스로틀 기반) |
| 4S LiPo (Pollyttronics PT-B8000-FX35, XT90S) | PDB/Power Module | 주 전원. 저전압 기준 14.0V (BATT_LOW_VOLT), 아밍 최소 14.8V |
| Matek BEC12S-PRO | 배터리 → Pi 5 | 컴패니언 5V 전원. Pi 5 + 카메라 피크 부하 검증 필요 (미검증 리스크) |
| ELRS RC Receiver (BETAFPV Nano 2400, CRSF) | FC TELEM2 (SERIAL2_PROTOCOL=23) | 수동 takeover, 비상 개입. RC_CHANNELS 네이티브 인식 필수 |
| External Compass | 미장착 (권장: 마스트 장착 검토) | 내장 컴퍼스는 4S PDB/ESC 인접 — COMPASS_MOT 캘리브레이션 + 경기장 현지 캘리브레이션 필수 |

## 3. Repo 역할

### uav-onboard

담당:

- Raspberry Pi camera capture
- Fake/Gazebo/rpicam frame source abstraction
- ArUco/line/intersection detection and stabilization
- Intersection decision and local grid-node telemetry
- Grid mission state machine and snake planner
- Marker registry/stability gate
- Guidance/control mapping to body velocity setpoints
- MAVLink UDP and serial transport
- Pixhawk bench tools and line-follow staging
- Safety/failsafe and telemetry/debug video sender

Current executables:

| Executable | Current role |
|---|---|
| `astroquad-onboard` | Current full grid-arena snake mission runtime. |
| `uav-onboard-telem` | Basic telemetry sender / development probe. |
| `vision_debug_node` | Vision/GCS bring-up and tuning. |
| `line_follow_node` | Short line-follow SITL/guarded ArduPilot serial staging (RC-override backend retired). |
| `mavlink_probe` | No-arm Pixhawk/MAVLink/local-estimate probe. |
| `mavlink_motor_test` | Props-removed low-throttle motor command check. |
| `video_streamer` | Raw MJPEG transport smoke tool. |

Current grid mission command (real-hardware/Tailscale — `--gcs-ip` no
longer needs a literal IP; `config/network.toml`'s `gcs.ip = "gcs-laptop"`
resolves through `src/common/KnownHosts.hpp`):

```bash
./build/astroquad-onboard --config config --target ardupilot_serial \
  --line-mode light_on_dark --marker-count 4 --revisit-order asc \
  --video --allow-arm-takeoff
```

SITL still needs an explicit destination (the WSL host IP is not the
Tailscale address):

```bash
./build/astroquad-onboard --config config --target sitl --vision gazebo \
  --world grid --line-mode light_on_dark --marker-count 4 \
  --video --gcs-ip <windows-gcs-ip>
```

Current entrypoint structure:

- `src/main.cpp` is a thin launcher for `onboard::app::AstroquadOnboardApp`.
- `src/app/AstroquadOnboardApp.*` owns the grid mission composition root.
- `src/app/GcsTelemetryPublisher.*` owns GCS telemetry/debug-video publishing.

### uav-gcs

담당:

- UDP telemetry receive/parse/display
- UDP MJPEG debug video receive/display
- GCS discovery beacon
- GCS-side marker/line/intersection overlay from onboard metadata
- Vision/system/camera/video logs
- Local committed-node grid map display
- Future mission dashboard and command sender

Current executables:

| Executable | Current role |
|---|---|
| `astroquad-gcs` | Current primary GCS UI: telemetry, video, overlays, grid/mission logs. |
| `uav-gcs-telem` | Telemetry-only console receiver / development probe. |
| `uav-gcs-video` | Raw MJPEG viewer. |
| `mock_onboard`, `log_replayer` | Development tools. |

Windows Ninja example:

```powershell
.\build\astroquad-gcs.exe --config config
```

Current entrypoint structure:

- `src/main.cpp` is a thin launcher for `gcs::app::AstroquadGcsApp`.
- `src/app/AstroquadGcsApp.*` owns GCS UI/log orchestration.
- `src/app/TelemetryWorker.*` and `src/app/VideoReceiveWorker.*` own receive
  worker threads.

## 4. 모듈 경계

ROS를 직접 쓰지는 않지만, typed input/output만 공유하는 node graph식 구조를
유지한다.

```text
FrameSource
  -> VisionProcessor
  -> IntersectionDecisionEngine
  -> GridCoordinateTracker
  -> GridMission / LineFollowMission
  -> GridControlMapper / GuidedVelocityController
  -> AutopilotMavlinkAdapter

Support:
  SafetyMonitor observes Vision/Mission/Autopilot/GCS state
  GcsTelemetryPublisher sends telemetry/debug video
  GCS observes telemetry/video only
```

경계 규칙:

- Vision code does not know flight mode or MAVLink.
- Mission code produces state/control intent; it does not encode MAVLink packets.
- Control mapping converts intent to `ControlSetpoint`.
- Autopilot adapter owns MAVLink send/receive only.
- GCS does not promote vision candidates into mission decisions.
- Debug video is not mission-critical.

## 5. 현재 알고리즘 기준

### Vision

- `LineMaskBuilder` creates filled bright/dark/local-contrast masks on a resized ROI.
- `LineDetector` scores candidate contours and measures tracking X from an
  anchor/lookahead projection band.
- `LineStabilizer` applies EMA/hold/reacquire/jump rejection.
- `IntersectionDetector` scores branch rays and classifies `straight`, `L`, `T`, `+`.
- `IntersectionDecisionEngine` aggregates branch evidence over a short window
  and emits `node_record`, `turn_confirm`, `turn_ready`, cooldown, and
  overshoot-risk telemetry.
- ArUco detector/stabilizer reports current-frame markers; grid mission applies
  a separate sliding marker window before committing marker IDs.

### Grid mission

Current grid mission is designed for the Gazebo grid arena:

- 3m x 3m vertiport at origin with ArUco ID 23.
- No line from vertiport to grid.
- 5 x 8 grid cells, 3m cell size, white 10cm lines on grass (matches the
  real-field polarity; the legacy black-line edge-case world was retired
  2026-07).
- Four grid ArUco markers expected by default, laid directly on grass as
  printed cards with a thin 5cm-wide white ArUco quiet zone (0.60m card) but
  no separate white mounting platform (matches the physical printed marker
  sample; real-field geometry).

State flow:

```text
ARM_TAKEOFF
  -> MARKER_LOCK_YAW
  -> ENTRY_FORWARD
  -> ENTRY_CENTER_ORIGIN
  -> SNAKE_LAUNCH_ALIGN
  -> SNAKE_FORWARD
  -> SNAKE_RECORD_NODE
  -> SNAKE_STOP_AT_CENTER
  -> SNAKE_TURN_90
  -> SNAKE_ADVANCE_ONE_CELL
  -> SNAKE_TURN_90_AGAIN
  -> SNAKE_COMPLETE
  -> LAND
```

Important rules:

- Mission does not use Gazebo ground-truth pose.
- `ENTRY_FORWARD` flies yaw-locked blind from vertiport toward the first grid node.
- The first grid node becomes local `(0,0)` only after `ENTRY_CENTER_ORIGIN`
  centers the intersection under the camera and passes velocity/stability gates.
- `GridCoordinateTracker::update()` is peek-only; only mission-approved
  `commitAdvance()` changes the grid coordinate.
- Between nodes, line following opens only in a mid-cell align window; otherwise
  yaw remains locked and forward velocity is blind.
- Boundary turns use strict snake alternation. Missing expected branch completes
  the snake rather than silently backtracking.
- Current mission lands after expected markers are found or snake completes.
  Marker reverse revisit and return-home remain future work.

## 6. 제어 전략

Primary:

- ArduPilot `GUIDED`
- MAVLink `SET_POSITION_TARGET_LOCAL_NED`
- Body-frame velocity setpoint

Implemented transports:

- SITL/Gazebo: UDP MAVLink.
- Real Pixhawk 6C Mini bench/flight: serial/USB serial MAVLink.

Current safety boundary:

- `line_follow_node` has guarded ArduPilot serial support with no-arm smoke,
  RC/local-estimate gates, and explicit `--allow-arm-takeoff`.
- `mavlink_probe` and `mavlink_motor_test` support bench verification.
- `astroquad-onboard --target ardupilot_serial --allow-arm-takeoff` has guarded
  serial support and uses RC/local-estimate preflight gates.

`ALT_HOLD + RC_CHANNELS_OVERRIDE`는 2026-07 아키텍처 결정으로 **공식 폐기**
(EKF 위치 안정화 우회, RC 페일세이프 충돌, 구 기체 전용 PWM 상수 의존).
유일한 지원 제어 경로는 GUIDED body-frame velocity setpoint다. 상세 근거는
`uav-onboard/ARCHITECTURE_OVERVIEW.md`의 아키텍처 결정 기록 참조.

## 7. 통신 구조

공통 protocol 문서:

- `uav-onboard/docs/PROTOCOL.md`
- `uav-gcs/docs/PROTOCOL.md`

현재 protocol document version은 **v1.11**이고 JSON top-level
`protocol_version`은 integer `1`이다.

| Channel | Direction | Transport | Default port | Status |
|---|---|---|---:|---|
| Telemetry | onboard -> GCS | UDP JSON (>1200B는 AQT1 청킹) | 14550 | implemented |
| Command | GCS -> onboard | TCP JSON | 14551 | documented only (미구현, 향후 과제) |
| Video stream | onboard -> GCS | UDP MJPEG chunks | 5600 | implemented |
| GCS discovery | GCS -> LAN broadcast | UDP text beacon | 5601 | implemented |

**목적지 해석 (Known Hosts)**: Tailscale IP는 사실상 고정이므로 양
저장소 `src/common/KnownHosts.hpp`(byte-identical 유지)가 심볼릭 이름을
IP로 해석한다 — `gcs-laptop`=100.85.239.73(노트북),
`pi5`=100.101.84.47(이 Pi), `broadcast`=255.255.255.255(LAN/WSL 비컨
디스커버리). 온보드 `config/network.toml`의 기본 `gcs.ip`가
`"gcs-laptop"`이라 `--gcs-ip` 없이 실행해도 노트북으로 전송된다.
`--gcs-ip <ip|name>`는 알려진 이름과 리터럴 IP를 모두 받는다. **온보드
자신의 Tailscale IP(`pi5`)를 `--gcs-ip`에 넣는 실수를 주의** — GCS가
아니라 자기 자신에게 전송되어 "송신은 성공하는데 GCS엔 안 잡히는"
증상으로 나타난다.

**MTU 안전 청킹 (AQT1, v1.11)**: 텔레메트리 JSON이 1200바이트를
넘으면(전형적으로 2~5.5KB) 비디오와 동일한 28바이트 헤더 레이아웃을
`AQT1` 매직으로 재사용해 청크 분할 전송한다 — 1280바이트 MTU인
Tailscale/WireGuard 터널에서 IP 단편화에 의존하지 않기 위함.
1200바이트 이하는 기존과 동일한 단일 bare-JSON 데이터그램(하위 호환).

**Best-effort 보장**: `GcsTelemetryPublisher`의 송신 실패는 5초
rate-limit 경고로만 표시되고 다음 프레임에서 무조건 재시도한다(과거엔
한 번 실패하면 미션 종료까지 영구 중단되는 latch 버그가 있었음, 2026-07
수정). 텔레메트리 소켓은 논블로킹이라 송신 큐 포화도 미션 루프를 막지
않는다. 비디오는 다운스케일/재인코딩/청크 전송이 모두 별도 워커
스레드에서 수행되어 미션/비전 스레드는 latest-wins 포인터 스왑 비용만
낸다.

**디버그 비디오/텔레메트리 기본값 (LTE-sized, 2026-07)**: 기본값이 LAN이
아니라 실제 대회 LG U+ LTE 업링크(측정된 ~2.4 Mbit/s 청정 구간)에 맞춰져
있다. `send_width=600, send_height=0`(=종횡비 유지), `jpeg_quality=55`,
`send_fps=12`, `fec_group_size=4`이고 프레임 텔레메트리는 `send_fps=6`이다.
FEC·텔레메트리 포함 ~2.1~2.2 Mbit/s로 실측된다. 따라서 `--video`만 붙여
실행해도 링크-세이프하며, `--fps/--video-width/--video-quality/--telemetry-fps`는
경로를 재측정한 뒤에만 재조정(상향/하향)한다. `--fps <n>`은 상한(cap)이며
camera fps로 클램프된다. GCS 오버레이 좌표는 카메라 픽셀 공간(1456x1088)
기준으로 전송되고 GCS가 수신 프레임 크기에 맞춰 스케일링한다.

**비행 로깅 (per-run, 2026-07)**: `astroquad-onboard`는 SITL/실기 모두 매
실행마다 `logs/flights/run_NNNN_<timestamp>/` 폴더를 생성한다(기본 활성).
`meta.json`(argv/설정/시작 시각), `frames.csv`(~20Hz: 미션 상태·제어
intent·grid 좌표/heading·라인/교차점 결정·ArUco·제어 명령·MAVLink 센서),
`events.jsonl`(상태 전이·노드 커밋·마커 발견/재방문·안전 이벤트)를 남긴다.
전용 writer 스레드 + 바운드 큐라서 20Hz 제어 루프를 절대 막지 않는다.
설정은 `config/logging.toml`(base_dir/frame_log_hz/flush_interval_s),
CLI는 `--no-flight-log`, `--flight-log-dir <dir>`. `logs/`는 gitignore.

Important current telemetry:

- `vision.line`
- `vision.intersection`
- `vision.intersection_decision`
- `vision.grid_node`
- `vision.drone_position`
- `vision.markers`
- `debug.note`

`vision.grid_node` means the latest committed onboard grid node. In
`astroquad-onboard`, it is resent every frame for UDP loss
tolerance.

## 8. Gazebo/SITL 기준

Launchers:

```bash
bash ~/astroquad/uav-onboard/scripts/line_tracing_test.sh
bash ~/astroquad/uav-onboard/scripts/grid_arena_test.sh
```

Grid mission loop:

```bash
# Windows
.\build\astroquad-gcs.exe --config config

# WSL
WINDOWS_GCS_IP="$(ip route | awk '/default/ {print $3; exit}')"
cd ~/astroquad/uav-onboard
./build/astroquad-onboard --config config --target sitl --vision gazebo \
  --world grid --line-mode light_on_dark --marker-count 4 \
  --video --gcs-ip "$WINDOWS_GCS_IP"
```

`grid_arena_test_world` uses `astroquad_grid_course` and
`iris_with_downward_camera`. The grid runtime profile is
`config/runtime.sitl.grid.toml`.

## 9. 현재 구현 기준선

Implemented:

- Pi 5 + IMX296 mono `rpicam-vid` MJPEG capture (grayscale end-to-end).
- Fake/Gazebo/rpicam frame sources.
- ArUco, line, intersection, marker stabilization.
- Shared `VisionProcessor`.
- UDP JSON telemetry and UDP MJPEG debug video.
- GCS discovery and video unicast.
- GCS marker/line/intersection overlay.
- GCS vision/grid logs and committed-node ASCII map.
- MAVLink UDP and serial transports.
- Pixhawk no-arm probe and props-removed motor test tools.
- `line_follow_node` startup/flight/landing video path.
- Grid arena Gazebo world and `astroquad-onboard` SITL mission runtime.
- Grid mission entry centering, hop-distance gating, strict snake alternation,
  and sliding marker commit window.
- MTU-safe telemetry chunking (AQT1), best-effort non-latching GCS publish,
  central Tailscale known-host mapping, downscaled/worker-thread debug video,
  credit-based send-fps throttle, GCS overlay camera-space scaling (2026-07,
  see `TROUBLESHOOTING.md` #85).
- LTE-sized video/telemetry defaults (600px/q55/12fps + 6fps telemetry),
  vision-stale flyaway watchdog (hold→LAND), per-state turn/stop hang guards,
  per-run flight logging (`logs/flights/run_*`), background system telemetry,
  SYS_STATUS battery ingest (2026-07).

Staging / not final:

- Full GCS dashboard framework and command sender.
- Structured GCS mission-state telemetry from `astroquad-onboard`.
- GCS command channel.
- Marker reverse revisit.
- Return-home and official coordinate conversion.
- Replay tooling that consumes the per-run flight logs (logging itself is
  implemented; a dedicated replayer is future work).

## 10. 실기체 전환 gate

Before real autonomous flight:

- MTF-01 optical flow/range must feed ArduPilot EKF local estimate stably.
- `LOCAL_POSITION_NED`, rangefinder, optical-flow quality, heartbeat, battery,
  and RC/takeover path must be verified.
- Props-off motor order and low-throttle checks must pass.
- Manual hover and RC takeover must be confirmed.
- `line_follow_node --mavlink-smoke` and `mavlink_probe --strict-local-estimate`
  must pass.
- `astroquad-onboard` real arm/takeoff requires `--allow-arm-takeoff` and must
  remain behind the same RC/local-estimate/bench verification gates.

## 11. 문서 역할

- `SYSTEM_SPEC.md`: 전체 시스템 목적, module boundary, current baseline.
- `uav-onboard/PROJECT_SPEC.md`: onboard repo responsibilities, algorithms,
  targets, build/test.
- `uav-onboard/README.md`: current onboard runbook.
- `uav-onboard/sim/gazebo/README.md`: Gazebo worlds/models/run commands.
- `uav-gcs/PROJECT_SPEC.md`: GCS responsibilities, modules, status.
- `uav-gcs/README.md`: current GCS build/run/test guide.
- `docs/PROTOCOL.md`: telemetry/video/discovery wire-format spec.
- `TROUBLESHOOTING.md`: historical issue/decision log.

`RESEARCH.md` and `PLAN.md` are scratchpads for active work and are not the
long-term source of truth.
