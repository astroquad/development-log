# Astroquad Research Snapshot

Last updated: 2026-05-16 KST

Scope: facts needed for the first real-aircraft line-follow MVP:

```text
auto takeoff -> short straight line follow -> ArUco marker detected
  -> 3 second marker hover -> land
```

Out of scope for this step: full snake/grid exploration, marker revisit, GCS
command UI, RC override backend, battery monitor parameter automation, and
Pixhawk parameter write automation.

## Implemented In This Step

- `line_follow_node` now accepts
  `--line-mode auto|light_on_dark|dark_on_light`.
- `line_follow_node` prints selected line mode and RC safety policy at startup.
- `AutopilotMavlinkAdapter` decodes MAVLink `RC_CHANNELS` into
  `AutopilotState`.
- `SafetyMonitor` handles:
  - missing RC input when RC is required
  - stale RC input
  - operator takeover when mode changes away from `GUIDED`
- SITL/runtime default assumes RC is present so Gazebo line-follow can still
  run without a real receiver.
- Pixhawk1 runtime requires fresh RC before command-producing mission flight.
- Real serial mission still requires explicit `--allow-arm-takeoff`; with that
  flag, RC gate failure still exits before `GUIDED`, arm, takeoff, or velocity
  commands.
- Unit coverage was added for RC MAVLink decode and safety-monitor decisions.

## Relevant Files

- `uav-onboard/tools/line_follow_node.cpp`
- `uav-onboard/src/autopilot/AutopilotState.hpp`
- `uav-onboard/src/autopilot/AutopilotMavlinkAdapter.cpp`
- `uav-onboard/src/safety/SafetyMonitor.*`
- `uav-onboard/config/safety.toml`
- `uav-onboard/config/runtime.sitl.toml`
- `uav-onboard/config/runtime.pixhawk1.toml`
- `uav-onboard/tests/test_autopilot_poll_drain.cpp`
- `uav-onboard/tests/test_safety_monitor.cpp`
- `uav-onboard/tools/mavlink_probe.cpp`
- `uav-onboard/tools/vision_debug_node.cpp`

## Current Execution Policy

SITL/Gazebo:

- RC is assumed present by config.
- Existing line-follow smoke can run without ELRS hardware.
- Use `--line-mode` per run instead of editing `vision.toml`.

Pixhawk1 real serial:

- Serial path:
  `/dev/serial/by-id/usb-ArduPilot_fmuv2_260034001451373037353835-if00`
- Command-producing mission path is blocked unless `--allow-arm-takeoff` is
  passed.
- Even with `--allow-arm-takeoff`, fresh `RC_CHANNELS.chancount > 0` is
  required before mode/arm/takeoff.
- During mission, RC loss requests LAND if still in `GUIDED`.
- If the operator switches out of `GUIDED`, companion velocity commands stop
  and the mission exits as operator takeover.

## RC / ELRS Bench Facts

- MAVLink `RC_CHANNELS.chancount == 0` means no RC channels are available.
  Source: https://mavlink.io/en/messages/common.html#RC_CHANNELS
- ArduPilot documents monitoring pilot RC input via `RC_CHANNELS`.
  Source: https://ardupilot.org/dev/docs/mavlink-rcinput.html
- Current receiver: BETAFPV Nano 2400 RX / ExpressLRS 2.4GHz Nano RX.
- Intended Pixhawk1 TELEM2/ELRS parameters:

```text
SERIAL2_PROTOCOL = 23
BRD_SER2_RTSCTS = 0
RC_PROTOCOLS = 512
RC_OPTIONS = 8480
RSSI_TYPE = 3
```

- ArduPilot CRSF/ELRS docs say CRSF uses full-duplex UART,
  `SERIALx_PROTOCOL=23`, and `RSSI_TYPE=3`.
  Source: https://ardupilot.org/copter/docs/common-tbs-rc.html
- After a MAVLink-triggered autopilot reboot, CRSF/ELRS may need receiver power
  cycling while the transmitter is already on.

Physical checks still needed:

- Transmitter is on and bound.
- Receiver LED indicates linked state.
- Receiver output mode is CRSF.
- Pixhawk/receiver is power-cycled with transmitter already on.
- Receiver TX -> Pixhawk TELEM2 RX.
- Receiver RX -> Pixhawk TELEM2 TX if telemetry/backchannel is used.
- TELEM2 plug orientation, 5V, and GND are correct.

## Remaining Real-Flight Blockers

Do not run real `--allow-arm-takeoff` flight until all gates below pass:

- `mavlink_probe --target pixhawk1 --strict-rc` passes.
- Battery telemetry/failsafe risk is addressed or explicitly accepted.
  Previous observation had `BATT_MONITOR=0`.
- Flight-controller pre-arm risk is addressed or explicitly accepted.
  Previous observation had `ARMING_CHECK=0`.
- Props-off motor order/direction is verified.
- Props-off arm/disarm/LAND command path is verified.
- Operator can immediately switch mode/kill/land from the transmitter.
- GCS video/telemetry and line overlay are stable.

## Command Gates

Build/test:

```bash
cd uav-onboard
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

SITL line-follow regression:

```bash
./build/line_follow_node --config config --target sitl \
  --vision gazebo --line-mode light_on_dark --video
```

Pixhawk no-arm gates:

```bash
./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 12000 --strict-local-estimate

./build/mavlink_probe --config config --target pixhawk1 \
  --duration-ms 30000 --strict-rc

./build/line_follow_node --config config --target pixhawk1 \
  --mavlink-smoke --smoke-duration-ms 5000 --no-telemetry
```

Vision-only line color check:

```bash
./build/vision_debug_node --config config \
  --line-only --line-mode light_on_dark --video

./build/vision_debug_node --config config \
  --line-only --line-mode dark_on_light --video
```

First real mission command, only after every gate passes:

```bash
./build/line_follow_node --config config --target pixhawk1 \
  --vision rpicam --line-mode light_on_dark --video --allow-arm-takeoff
```

## Operational Notes

- Do not run `vision_debug_node` and `line_follow_node` simultaneously as GCS
  video/telemetry senders during flight.
- During mission flight, `line_follow_node` owns camera, vision, telemetry,
  video, MAVLink control, and safety handling.
- `vision_debug_node` remains for vision-only smoke/tuning.
- GCS overlay uses onboard metadata only; GCS should not rerun detection.
