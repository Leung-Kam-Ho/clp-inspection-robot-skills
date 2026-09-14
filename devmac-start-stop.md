---
name: devmac_start_stop
description: "Start and stop dev-mac only (socattui, mac scripts, Docker)."
last_updated: 2026-09-14
---

# Dev-Mac Start/Stop

## Start Dev-Mac Only

1. **Start socattui on dev-mac:**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "socattui up"
   ```

2. **Start mac scripts on dev-mac:**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "~/clp-inspection-robot-ros2/scripts/main.mac.sh"
   ```

3. **Start Docker containers on dev-mac:**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "/Applications/Docker.app/Contents/Resources/bin/docker compose -f ~/clp-inspection-robot-ros2/compose.yaml up -d --no-build"
   ```

4. **Verify connection** (wait ~10s for heartbeat):
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "/Applications/Docker.app/Contents/Resources/bin/docker compose -f ~/clp-inspection-robot-ros2/compose.yaml ps"
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/data | jq ."
   ```

## Stop Dev-Mac Only

1. **Stop socattui:**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "socattui down"
   ```

2. **Stop all tmux sessions (camera + audio):**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "tmux kill-server"
   ```

3. **Stop Docker containers:**
   ```bash
   ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "/Applications/Docker.app/Contents/Resources/bin/docker compose -f ~/clp-inspection-robot-ros2/compose.yaml down -v"
   ```

## Verify Shutdown

```bash
ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/data | jq ."
ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} "tmux list-sessions"
```
- System status should return empty or error — no longer connected
- Tmux sessions should be empty (no output)

## Notes

- The `startSystem.sh` script runs: robot hardware → launcher hardware → socattui → main.mac.sh → Docker. This section skips the remote hardware (robot/launcher) and only starts local dev-mac components.
- Always ping all machines first before starting — see SKILL.md.
