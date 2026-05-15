# Astroquad Research Snapshot

최종 업데이트: 2026-05-15 KST, Pixhawk serial bench 이후

이 문서는 에이전트가 **이 파일만 읽고도** 현재 Astroquad 프로젝트의 구조, 진행상황, 실기체 연결 상태, Pixhawk 설정, 다음 구현 지점을 빠르게 파악하도록 만든 작업용 요약이다. 기준 문서는 `SYSTEM_SPEC.md`, `MVP_PLAN.md`, `uav-onboard/PROJECT_SPEC.md`, `uav-gcs/PROJECT_SPEC.md`를 따른다.

## 1. 현재 결론

현재 우선순위는 mission logic 확장이 아니라 **Raspberry Pi 4와 Pixhawk1 사이의 실제 MAVLink 연결을 안전하게 여는 레이어**다.

확인된 핵심 사실:

- SITL/Gazebo에서는 `line_follow_node`가 heartbeat, GUIDED, arm, takeoff, line-follow, marker hover, land, complete까지 수행한다.
- 실기체 USB MAVLink 연결은 Pi에서 stable path `/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00`로 열린다.
- `SerialMavlinkTransport`, no-arm `mavlink_probe`, low-throttle `mavlink_motor_test`가 `uav-onboard` commit `030a476`에 구현되어 push/pull/build/test까지 완료됐다.
- `line_follow_node --target pixhawk1 --mavlink-smoke`는 실제 Pixhawk serial endpoint를 사용하되 mode/arm/takeoff/velocity command를 보내지 않고 heartbeat/state/range/local velocity만 확인한다.
- Pixhawk parameter 파일 `config/pixhawk1_usb.params`가 repo에 추가됐고, Pi에서 `EK3_SRC1_POSXY=0`, `EK3_SRC1_VELZ=0`를 적용했다. 이후 해당 파일의 target 값은 모두 `ok`로 확인됐다.
- 현재 no-arm probe에서 `LOCAL_POSITION_NED`, `DISTANCE_SENSOR`, `RANGEFINDER`, `OPTICAL_FLOW`, `EKF_STATUS_REPORT`가 확인됐다. MTF-01/EKF local estimate gate는 no-arm bench 기준 통과했다.
- 아직 `RC_CHANNELS count=0`, `BATT_MONITOR=0`, battery telemetry unavailable 상태다. RC takeover와 battery/failsafe는 다음 blocker다.

## 2. 프로젝트 목표와 MVP 범위

최종 목표:

```text
이륙
  -> 라인 추종 및 격자 진입
  -> snake 방식 grid 탐색
  -> 교차점별 ArUco marker 기록
  -> marker 번호 역순 재방문
  -> 출발점 복귀
  -> 자동 착륙
```

현재 72시간 실기체 MVP:

```text
MTF-01 bring-up
  -> 자동 이륙
  -> 짧은 직선 line follow
  -> 종료 marker 또는 line end/lost/timeout/operator abort 중 하나에서 안전 착륙
```

MVP에서 제외된 것:

- full snake grid exploration
- ArUco marker revisit
- official coordinate conversion
- 교차점 회전/분기 의사결정
- GCS full command UI
- marker count command
- backend switching UI
- command ACK/retry UI

## 3. Repo 역할

### uav-onboard

담당:

- Raspberry Pi camera capture
- ArUco/line/intersection detection
- marker/line/intersection stabilization
- mission state machine
- guidance/control backend
- MAVLink adapter
- safety/failsafe
- telemetry/debug video sender

주요 실행 파일:

- `uav_onboard`: 최종 onboard composition root. 현재는 basic telemetry sender에 가깝고, 최종적으로 vision/mission/control/safety/telemetry/MAVLink를 조립해야 한다.
- `vision_debug_node`: vision bring-up/debug 전용. camera, detector, telemetry, optional debug video만 담당한다.
- `line_follow_node`: SITL/MVP staging executable. 현재 SITL에서는 자동 이륙, line-follow, marker hover, land까지 검증됐다.
- `video_streamer`, `line_detector_tuner`, `aruco_detector_tester`, `grid_image_smoke`, `marker_grid_replay`: smoke/tuning 도구.

### uav-gcs

담당:

- UDP JSON telemetry 수신/파싱/표시
- UDP MJPEG debug video 수신/표시
- GCS-side marker/line/intersection overlay
- vision log and grid-map log display
- future command sender and mission dashboard

현재 MVP에서 GCS는 command UI가 아니라 관제와 기록에 집중한다. `uav_gcs_vision_debug`가 핵심 관제 도구다.

## 4. 현재 구현 기준선

이미 구현/검증된 것:

- Pi 4 + IMX519 `rpicam-vid` MJPEG capture
- Gazebo `FrameSource`와 SITL runtime profile
- Astroquad Gazebo vision world/fixtures
- UDP JSON telemetry send/receive
- opt-in UDP MJPEG debug video send/receive
- GCS discovery beacon and video unicast switch
- onboard ArUco detection
- onboard line tracing and line stabilizer
- onboard intersection classifier/stabilizer
- marker-aware line/intersection mask 처리
- `MarkerStabilizer`
- `IntersectionDecisionEngine`
- `GridCoordinateTracker`
- GCS marker/line/intersection overlay
- SITL MAVLink UDP heartbeat/mode/arm/takeoff/body-velocity/land staging path
- MAVLink UDP command peer pinning
- `line_follow_node` startup video와 landing video streaming
- native POSIX serial MAVLink transport
- no-arm `mavlink_probe` bench tool
- `line_follow_node --mavlink-smoke` real Pixhawk serial smoke path
- explicit `--allow-arm-takeoff` guard for real serial `line_follow_node`
- low-throttle `mavlink_motor_test` command tool with `--props-removed` safety acknowledgement
- Pixhawk parameter target file `config/pixhawk1_usb.params`

미구현 또는 staging:

- safety monitor expansion
- GCS command channel
- full snake mission policy
- marker revisit policy
- official coordinate conversion
- file logging/persistent replay system
- RC takeover verification
- battery monitor/failsafe setup
- props-off arm/disarm/motor command bench

## 5. 현재 line_follow_node 상태

중요 코드:

- `uav-onboard/tools/line_follow_node.cpp`
- `uav-onboard/src/autopilot/MavlinkTransport.hpp`
- `uav-onboard/src/autopilot/UdpMavlinkTransport.*`
- `uav-onboard/src/autopilot/AutopilotMavlinkAdapter.*`
- `uav-onboard/src/control/GuidedVelocityController.*`
- `uav-onboard/src/mission/LineFollowMission.*`
- `uav-onboard/src/safety/SafetyMonitor.*`

현재 흐름:

```text
FrameSource(fake/gazebo/rpicam)
  -> VisionProcessor
  -> VisionDebugPublisher(GCS telemetry/video)
  -> GuidedVelocityController
  -> AutopilotMavlinkAdapter
  -> SET_POSITION_TARGET_LOCAL_NED / MAV_FRAME_BODY_NED
```

현재 mission state:

```text
IDLE -> TAKEOFF -> LINE_FOLLOW -> MARKER_APPROACH -> MARKER_HOVER -> LAND -> COMPLETE
any state -> ABORT
```

주의점:

- `--target pixhawk1`는 이제 `SerialMavlinkTransport`를 선택한다.
- 실제 serial target에서는 `--mavlink-smoke`, `--no-arm`, `--dry-run`, 또는 `--allow-arm-takeoff` 중 하나가 없으면 실행을 거부한다.
- `--mavlink-smoke`는 heartbeat/stream/state만 확인하고 mode/arm/takeoff/velocity command를 보내지 않는다.
- `--allow-arm-takeoff`를 붙이면 기존 자동 mission path가 열릴 수 있으므로 props-off command bench와 RC/battery/failsafe 확인 전에는 사용하지 않는다.

## 6. Config 상태

`uav-onboard/config/autopilot.toml`:

```toml
[transport]
kind = "udp"
listen_host = "0.0.0.0"
listen_port = 14550

[serial]
device = "/dev/serial0"
baudrate = 115200

[mavlink]
system_id = 191
component_id = 191
target_system = 1
target_component = 1
setpoint_rate_hz = 20
```

`uav-onboard/config/runtime.pixhawk1.toml`:

```toml
[runtime]
target = "pixhawk1"
vision = "rpicam"

[transport]
kind = "serial"

[serial]
device = "/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00"
baudrate = 115200
```

현재 실제 연결은 Pixhawk USB이므로 `runtime.pixhawk1.toml`은 stable by-id path를 사용한다. 현재 Pi의 `/dev/serial0`은 UART console과 충돌한다.

## 7. Raspberry Pi 4 조사 결과

접속:

- SSH: `astroquad@astroquad.local`
- IP: `192.168.0.138`
- password: `astroquad`

시스템:

- Hostname: `astroquad`
- OS: Debian GNU/Linux 13 trixie
- Kernel: `Linux 6.12.75+rpt-rpi-v8`
- Architecture: `arm64`
- Board: Raspberry Pi 4 Model B Rev 1.5
- Timezone/current observed time: KST
- Throttling: `throttled=0x0`

사용자 권한:

- User: `astroquad`
- Groups include `dialout`, `sudo`, `video`, `plugdev`, `gpio`, `i2c`, `spi`, `render`, `input`
- `/dev/ttyACM0` 접근 권한은 `root:dialout`, mode `660`; 현재 사용자 권한으로 접근 가능.

Network:

- `wlan0`: `192.168.0.138/24`
- default gateway: `192.168.0.1`
- `eth0`: down

Camera:

- `rpicam-vid` installed.
- IMX519 detected:

```text
0 : imx519 [4656x3496 10-bit RGGB]
Modes include 1280x720, 1920x1080, 2328x1748, 3840x2160, 4656x3496
```

Remote repo/build:

- Remote onboard repo path: `/home/astroquad/astroquad/uav-onboard`
- Remote onboard HEAD observed: `b337d02`
- Local onboard HEAD observed: `23214a4`
- Remote build directory exists and contains `line_follow_node`, `vision_debug_node`, `uav_onboard`, tests, static libs.
- Remote git status printed no dirty entries during this investigation.

주의: remote onboard repo revision이 local과 다르다. 다음 구현/배포 전에는 어떤 revision을 기준으로 빌드할지 맞춰야 한다.

## 8. Pi serial/USB 포트 상태

Pixhawk USB:

```text
/dev/ttyACM0
/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00 -> ../../ttyACM0
/dev/serial/by-path/platform-fd500000.pcie-pci-0000:01:00.0-usb-0:1.2:1.0 -> ../../ttyACM0
```

USB identification:

```text
Bus 001 Device 003: ID 1209:5741 Generic fmuv2
Manufacturer: ArduPilot
Product: fmuv2
SerialNumber: 260034001451373037353835
Driver: cdc_acm
```

Port busy check:

- `fuser /dev/ttyACM0` showed no process using it.
- No `mavproxy`, `mavlink-router`, `line_follow_node`, `vision_debug_node` process was running.

Installed MAVLink helper tools:

- `python3`: installed
- `rpicam-vid`: installed
- `pyserial`: not installed
- `pymavlink`: not installed
- `mavproxy.py`: not found
- `mavlink-routerd`: not found

UART state:

```text
/dev/serial0 -> ttyS0
```

`/proc/cmdline` includes:

```text
console=ttyS0,115200 console=tty1
```

`serial-getty@ttyS0.service`:

```text
enabled-runtime
active
```

Boot config relevant lines:

```text
dtoverlay=imx519
dtoverlay=dwc2,dr_mode=host
enable_uart=1
```

Implication:

- Current USB wiring should use `/dev/ttyACM0` or `/dev/serial/by-id/...`.
- Do not use `/dev/serial0` for Pixhawk right now. It is tied to Linux console/getty.
- If later using TELEM UART, first disable serial console/getty and then choose the correct UART device deliberately.

## 9. Pixhawk MAVLink 조사 결과

조사 방식:

- Device: `/dev/ttyACM0`
- Sent only companion/GCS heartbeat, stream request, `PARAM_REQUEST_LIST`, selected message requests.
- Did not send arm, disarm, mode change, takeoff, land, RC override, or velocity command.

Heartbeat:

```text
target system/component: 1/1
type: 2
autopilot: 3
custom_mode: 0
mode: STABILIZE
armed: false
base_mode: 0x51
system_status: 3
mavlink_version: 3
```

Firmware/statustext observed:

```text
2M flash - use fmuv3 firmware
ArduCopter V4.6.3 (92b0cd78)
ChibiOS: 88b84600
Frame: QUAD/X
```

Parameter list:

- Total parameter count observed: `918`
- Parameter list read succeeded over `/dev/ttyACM0`.

Key parameters:

```text
SYSID_THISMAV=1
SYSID_MYGCS=255
FRAME_CLASS=1
FRAME_TYPE=1
ARMING_CHECK=0
ARMING_NEED_LOC=0
BATT_MONITOR=0
FS_THR_ENABLE=1
FS_GCS_ENABLE=0
AHRS_EKF_TYPE=3
EK3_ENABLE=1
EK3_SRC1_POSXY=3
EK3_SRC1_VELXY=5
EK3_SRC1_POSZ=1
EK3_SRC1_VELZ=3
EK3_SRC1_YAW=1
EK3_FLOW_USE=1
FLOW_TYPE=5
RNGFND1_TYPE=10
RNGFND1_ORIENT=25
RNGFND1_MIN_CM=1
RNGFND1_MAX_CM=800
RNGFND1_GNDCLEAR=10
WPNAV_RFND_USE=1
GPS1_TYPE=1
GPS2_TYPE=0
COMPASS_ENABLE=1
COMPASS_USE=1
BATT_MONITOR=0
MOT_PWM_TYPE=0
MOT_SPIN_ARM=0.1
MOT_SPIN_MIN=0.15
```

Notable missing/renamed parameters:

- `SERIAL0_PROTOCOL`, `SERIAL1_PROTOCOL`, etc. were not present in the observed parameter list.
- `GPS_TYPE`/`GPS_TYPE2` were not present; current firmware exposes `GPS1_TYPE`/`GPS2_TYPE`.
- EK2 parameters were mostly absent; current setup appears EK3-based.

## 10. Pixhawk sensor/estimate telemetry observations

Observed messages included:

- `HEARTBEAT`
- `SYS_STATUS`
- `SYSTEM_TIME`
- `GPS_RAW_INT`
- `RAW_IMU`
- `SCALED_PRESSURE`
- `ATTITUDE`
- `GLOBAL_POSITION_INT`
- `SERVO_OUTPUT_RAW`
- `RC_CHANNELS`
- `VFR_HUD`
- `EKF_STATUS_REPORT`
- several ArduPilot/status messages

Not observed during MAVLink probe:

- `LOCAL_POSITION_NED` (`msgid 32`)
- `DISTANCE_SENSOR` (`msgid 132`)
- `OPTICAL_FLOW` (`msgid 100`)
- `OPTICAL_FLOW_RAD` (`msgid 106`)

Representative decoded values:

```text
GPS_RAW_INT:
  fix_type=0
  satellites=0
  lat=0
  lon=0

GLOBAL_POSITION_INT:
  lat=0
  lon=0
  alt_m=-1.790
  relative_alt_m approximately -95m
  velocity approximately 0

RC_CHANNELS:
  count=0
  rssi=255

SYS_STATUS:
  voltage_battery_mV=0
  current_cA=-1
  battery_remaining_pct=0

VFR_HUD:
  airspeed=0
  groundspeed approximately 0
  alt approximately -1.79
  throttle=0

EKF_STATUS_REPORT:
  flags=0x00a7
```

Interpretation:

- Pixhawk USB MAVLink transport is alive.
- Current Pixhawk is disarmed in STABILIZE.
- Battery monitor is disabled or not configured for telemetry.
- RC input was not visible in `RC_CHANNELS` at probe time.
- GPS is absent/no fix.
- EKF is running, but local horizontal position was not available through `LOCAL_POSITION_NED`.
- MTF-01-related params are configured (`FLOW_TYPE=5`, `RNGFND1_TYPE=10`), but actual optical flow/range telemetry was not confirmed.
- The current state is not ready for GUIDED velocity line-follow gate.

Possible explanations for missing flow/range/local estimate:

- MTF-01 is not powered, not connected to the configured Pixhawk serial port, or not transmitting.
- Pixhawk receives range/flow internally but is not streaming the expected messages over USB.
- Current EKF source setup does not produce local XY position without GPS or valid optical-flow/range input.
- The vehicle is stationary on bench and some messages remain zero/truncated, but absence of `LOCAL_POSITION_NED`, `DISTANCE_SENSOR`, and flow messages still needs explicit verification.

## 11. 2026-05-15 Pixhawk serial bench 결과

구현 commit:

```text
uav-onboard 030a476 Add Pixhawk serial MAVLink bench tools
```

Pi 배포/빌드:

- Pi repo `/home/astroquad/astroquad/uav-onboard`를 `030a476`로 fast-forward pull.
- Pi에서 `cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON`.
- Pi build 성공.
- Pi CTest `13/13` 통과.

추가된 주요 파일:

- `src/autopilot/SerialMavlinkTransport.*`
- `tools/mavlink_probe.cpp`
- `tools/mavlink_motor_test.cpp`
- `tests/test_serial_mavlink_transport.cpp`
- `config/pixhawk1_usb.params`
- `config/pixhawk1_usb.expected.toml`

No-arm probe command:

```bash
cd /home/astroquad/astroquad/uav-onboard
./build/mavlink_probe --config config --target pixhawk1 --duration-ms 12000
```

대표 결과:

```text
heartbeat: system=1 component=1 mode=STABILIZE(0) armed=false
local_position_ned: x≈-1.97 y≈-2.79 z≈101.7 vx≈0 vy≈0 vz≈0.005
range: distance_sensor_m≈1.54 rangefinder_m≈1.54
optical_flow: quality≈139 ground_distance_m≈1.54
rc: channels=0 rssi=255
ekf: flags=0x16f
```

적용한 Pixhawk parameter:

```text
EK3_SRC1_POSXY: 3 -> 0
EK3_SRC1_VELZ: 3 -> 0
```

적용 명령:

```bash
./build/mavlink_probe --config config --target pixhawk1 --duration-ms 12000 \
  --param-file config/pixhawk1_usb.params \
  --apply-params --i-understand-this-writes-pixhawk-params
```

적용 후 `config/pixhawk1_usb.params`의 모든 target 항목은 `status=ok`로 확인됐다.

Strict local estimate gate:

```bash
./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 12000 --strict-local-estimate
```

결과:

- exit status `0`
- `LOCAL_POSITION_NED`, range, optical flow 모두 확인됨.
- no-arm bench 기준 MTF-01/EKF local estimate gate 통과.

`line_follow_node` serial smoke:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

결과:

- exit status `0`
- 실제 Pixhawk serial endpoint 사용.
- mode/arm/takeoff/velocity command 없음.
- heartbeat, local XY, velocity, range가 로그에 출력됨.

아직 미실행:

- `mavlink_motor_test` 실제 모터 회전은 물리 안전 확인이 필요해 자동 실행하지 않았다.
- 사용자가 기체 주변을 정리하고 손/케이블을 치운 상태에서 직접 실행하는 것이 안전하다.

## 12. Safety-critical findings

Do not run current mission path against real Pixhawk yet:

- `line_follow_node` now refuses real serial automatic mission unless `--allow-arm-takeoff` is explicitly provided, but that flag must not be used before props-off command bench is complete.
- `mavlink_probe` and `line_follow_node --mavlink-smoke` are the approved no-arm paths.
- `ARMING_CHECK=0` is currently set on Pixhawk, so pre-arm protection is weakened.
- `BATT_MONITOR=0`, so low battery telemetry/failsafe cannot be trusted from current MAVLink data.
- `RC_CHANNELS count=0`, so RC takeover path is not verified.
- Local estimate and range/flow are verified in no-arm bench, but not yet verified under hover/flight dynamics.

Physical safety status provided by user:

- Pixhawk1, motors, MTF-01 are connected.
- Motors have power.
- Propellers are removed.
- This is correct for the current bench stage, but software must still avoid arm/takeoff until no-arm probes exist.

## 13. Recommended next implementation plan

Priority order:

1. Add a no-arm MAVLink probe path.
2. Implement `SerialMavlinkTransport`.
3. Run props-off Pixhawk bench probe over `/dev/ttyACM0` or stable by-id path.
4. Confirm MTF-01 flow/range and EKF local estimate.
5. Expand safety gates.
6. Only then test props-off mode/arm/land command paths.
7. Only after all above, try short real line-follow.

Minimum `mavlink_probe` behavior:

- Open serial endpoint.
- Receive heartbeat.
- Send companion heartbeat.
- Request message intervals for:
  - `SYS_STATUS`
  - `LOCAL_POSITION_NED`
  - `GLOBAL_POSITION_INT`
  - `DISTANCE_SENSOR`
  - `OPTICAL_FLOW`
  - `OPTICAL_FLOW_RAD`
  - `EKF_STATUS_REPORT`
  - `RC_CHANNELS`
- Read and print selected parameters.
- Never call `setGuidedMode`, `arm`, `takeoff`, `land`, `sendBodyVelocity`, or `RC_CHANNELS_OVERRIDE`.
- Exit nonzero if local estimate/range/flow are missing when the user asks for a strict gate.

Minimum `SerialMavlinkTransport` requirements:

- POSIX `termios` raw mode.
- Support USB CDC `/dev/ttyACM0`, stable `/dev/serial/by-id/...`, and later UART paths.
- Configurable baudrate, but tolerate USB CDC where baudrate is mostly nominal.
- `recvMessage(message, timeout_ms)` with `select`/timeout.
- `sendMessage(message)` with partial write handling.
- MAVLink parse channel separate from UDP if both coexist in tests.
- Clear errors for permission denied, missing device, busy device, timeout.
- RAII restore/close behavior.

Suggested real-device config default for current wiring:

```toml
[serial]
device = "/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00"
baudrate = 115200
```

Do not default current Pixhawk USB testing to `/dev/serial0`.

## 14. Bench verification checklist

Props off, no arm:

- Pi can open Pixhawk USB serial.
- Heartbeat shows expected sysid/component.
- Mode and armed state are decoded.
- `SYS_STATUS` battery data is either valid or explicitly marked unavailable.
- `RC_CHANNELS` shows receiver input before relying on RC takeover.
- `LOCAL_POSITION_NED` appears and updates.
- `DISTANCE_SENSOR` or equivalent range telemetry appears and responds to height changes.
- `OPTICAL_FLOW` or `OPTICAL_FLOW_RAD` appears and responds to horizontal motion/texture.
- EKF flags indicate usable attitude, height, horizontal velocity, and local position as required.

Props off, command path later:

- Mode switch is tested only after no-arm probe is stable.
- Arm command is tested only with props off and operator ready.
- Land command is tested as a command path, not after actual takeoff.
- RC mode switch/takeover is verified.
- Failsafe behavior is documented.

Props on is out of scope until all above gates pass.

## 15. Current blockers

Hard blockers:

- RC input not visible in current MAVLink probe.
- Battery monitor/failsafe not configured in current MAVLink data.
- Props-off motor command bench has not been executed with operator physically observing motors.
- Props-off arm/disarm/mode command bench has not been executed.

Resolved blockers:

- Native serial MAVLink transport is implemented.
- No-arm MAVLink probe is implemented.
- Pixhawk local estimate/range/flow is confirmed in no-arm bench.

Soft blockers:

- Pixhawk `ARMING_CHECK=0` and `BATT_MONITOR=0` reduce safety observability until deliberately configured.
- Remote Pi lacks `pymavlink`, `pyserial`, `mavproxy.py`, `mavlink-routerd`; future tooling should either be C++ native or explicitly install dependencies.
- UART `/dev/serial0` is currently Linux console/getty, not a safe Pixhawk UART endpoint.

## 16. Do not forget

- Do not mix `vision_debug_node` and `line_follow_node` as simultaneous GCS video/telemetry senders.
- GCS overlay uses onboard metadata only; GCS should not rerun detection.
- Debug video is not mission-critical.
- Full snake/revisit remains postponed until first real line-follow MVP is safe.
- The next coding step should be narrow: no-arm MAVLink probe and serial transport, not mission expansion.
