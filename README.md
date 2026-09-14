# CLP Inspection Robot Skills

Operational documentation for controlling the CLP inspection robot and launcher system via SSH.

## Architecture

| Component | Host | Description |
|-----------|------|-------------|
| Dev Mac | `${CLP_DEV_HOST:-dev@development-mac-m1.local}` | ROS2 API server, Docker containers, web server |
| Robot | `${CLP_IR_HOST:-dev@clp-ir.local}` | Robot hardware (servo, relay, pressure, LED) |
| Launcher | `${CLP_LP_HOST:-dev@clp-lp.local}` | Launcher hardware (angle, relay, brake, laser) |

## Quick Start

### Before Starting

**Always ping all machines first** to check which are powered on:

```bash
ping -c 2 -W 2 ${CLP_IR_HOST:-dev@clp-ir.local}
ping -c 2 -W 2 ${CLP_LP_HOST:-dev@clp-lp.local}
ping -c 2 -W 2 ${CLP_DEV_HOST:-dev@development-mac-m1.local}
```

Report which are online before proceeding. Only start components on reachable machines.

### Start Full System

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "~/clp-inspection-robot-ros2/startSystem.sh"
```

This starts robot hardware → launcher hardware → socattui → mac scripts → Docker containers.

### Stop Full System

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "~/clp-inspection-robot-ros2/stopSystem.sh"
```

### Start/Stop Individual Components

- **Dev-mac only**: see `devmac-start-stop.md`
- **Robot only**: see `robot-start-stop.md`
- **Launcher only**: see `launcher-start-stop.md`

## Documents

| File | Description |
|------|-------------|
| `SKILL.md` | Main skill entry — SSH conventions, API endpoints, troubleshooting |
| `robot_playbook.md` | Robot control overview |
| `robot-commands.md` | Robot API: servo, relay, pressure, LED, EL CID |
| `robot-start-stop.md` | Start/stop robot hardware + Docker |
| `launcher_playbook.md` | Launcher control overview |
| `launcher-relay-control.md` | Relay string format, lock/unlock, brake operations |
| `launcher-positioning.md` | Slot-to-angle mapping, setpoint tolerance, reset |
| `launcher-start-stop.md` | Start/stop launcher hardware + Docker |
| `devmac-start-stop.md` | Start/stop dev-mac only (socattui, tmux, Docker) |

## Key Conventions

- **SSH timeout**: Always use `ssh -o ConnectTimeout=2` for local network commands
- **Relay API**: Toggles (not sets) — always read current state before toggling
- **Brake**: Bit 2 — always disengage before moving, re-engage after
- **Verify**: Always verify after every action before proceeding
- **Docker**: 9 containers — verify with `docker compose ps` after start
