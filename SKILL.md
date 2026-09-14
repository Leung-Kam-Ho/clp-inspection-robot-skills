---
name: clp-inspection-robot
description: "Control CLP inspection robot/launcher on dev-mac via SSH."
last_updated: 2026-09-11
linked_files:
  - robot_playbook.md: Robot API endpoints, usage examples, and links to sub-docs
  - robot-commands.md: Servo, relay, pressure, LED, EL CID controls; digital valve mapping
  - robot-start-stop.md: Start and stop robot (hardware + Docker)
  - launcher_playbook.md: Launcher API endpoints, usage examples, and links to sub-docs
  - launcher-relay-control.md: Relay string format, bit functions, lock/unlock, brake operations
  - launcher-positioning.md: Slot-to-angle mapping, setpoint tolerance, reset
  - launcher-start-stop.md: Start and stop launcher (hardware + Docker)
  - devmac-start-stop.md: Start and stop dev-mac only (socattui, mac scripts, Docker)
---

## Before Starting the System

**Always ping all machines first** to check which are powered on and reachable. Only start components on machines that respond.

```bash
ping -c 2 -W 2 clp-lp.local 2>&1   # launcher computer
ping -c 2 -W 2 clp-ir.local 2>&1   # robot computer
ping -c 2 -W 2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} 2>&1  # dev-mac
```

Report the results before proceeding:
- If a machine is unreachable, skip its components and tell the user which ones are offline.
- If all machines are up, proceed with the start sequence normally.

## Before Acting

**Always check the system status before performing any action.** Verify the current state of the component you're about to control — don't assume or guess.

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/data | jq ."
```

## SSH Timeout

All SSH commands target a local network. Use **`ConnectTimeout=2`** for the SSH connection, but allow **~5s total** for curl+jq reads (the connection is fast but the API response may lag).

```bash
ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/launch_platform_status | jq '.relay'"
# Use timeout=5 for reads, timeout=2 for simple POST commands
```

**Never omit `ConnectTimeout=2`** — it prevents hanging if the dev machine is unresponsive. See `launcher-relay-control.md` for the full timeout conventions.

## General API Endpoints (GET)

| Endpoint | Description |
|----------|-------------|
| `/data` | Full system status (all components + camera + progress) |
| `/robot_status` | Robot component status |
| `/digital_valve_status` | Digital valve status |
| `/launch_platform_status` | Launch platform status |
| `/auto_status` | Automation status |
| `/el_cid_status` | EL CID status |
| `/camera_status` | Camera server connected (bool) |
| `/fbg_status` | FBG sensor status |
| `/progress_status` | Inspection progress (elcid_progress, hld_progress) |

## Troubleshooting

### Docker Emulation

- All containers run linux/amd64 images on an M1 Mac (Docker emulation layer)
- Performance is slower than native; some edge cases may differ
- Verify containers are running after start; report any errors

### Component Heartbeat

- The `_update_connection_status` method checks 2s heartbeat
- Components marked disconnected (`connected: false`) if no status update within 2 seconds
- If a component shows as disconnected but is actually running, wait a moment and re-check

### SSH Connectivity

- If SSH hangs, the dev machine may be asleep or off the network
- Try `ping ${CLP_DEV_HOST:-dev@development-mac-m1.local}` first to verify reachability
- Check that the dev machine is awake (wake-on-LAN, power settings)

### Container Verification

After `./startSystem.sh`, always verify all 9 containers:

```bash
/Applications/Docker.app/Contents/Resources/bin/docker compose -f clp-inspection-robot-ros2/compose.yaml ps
```

Expected: robot_node, auto_node, digital_valve_node, elcid_node, launch_platform_node, web_server_node, camera_viewer_node, rviz_node, ros2_dev

### RViz

RViz is available at `http://${CLP_DEV_HOST:-dev@development-mac-m1.local}:6080` (port 6080 forwarded to container).

## Related Playbooks

- **[Robot Playbook](robot_playbook.md)** — Servo, relay, pressure, LED controls; robot start/stop
- **[Launcher Playbook](launcher_playbook.md)** — Angle, relay lock/unlock, movement, laser; launcher start/stop
