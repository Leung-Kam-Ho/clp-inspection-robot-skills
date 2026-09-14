---
name: launcher_start_stop
description: "Start and stop launcher (hardware + Docker)."
last_updated: 2026-09-11
---

# Launcher Start/Stop

## Start Launcher Only

1. **Start hardware scripts on remote launch platform:**
   ```bash
   ssh ${CLP_LP_HOST:-dev@clp-lp.local.local} "cd ~/clp-inspection-robot-ros2; ./scripts/main.launchplatform.sh"
   ```

2. **Start local ROS2 containers:**
   ```bash
   ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "/Applications/Docker.app/Contents/Resources/bin/docker compose -f ~/clp-inspection-robot-ros2/compose.yaml up -d --no-build"
   ```

3. **Verify connection** (wait ~10s for heartbeat):
   ```bash
   ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/launch_platform_status | jq .relay"
   ```

## Stop Launcher Only

1. **Stop local ROS2 containers:**
   ```bash
   ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "/Applications/Docker.app/Contents/Resources/bin/docker compose -f ~/clp-inspection-robot-ros2/compose.yaml down -v"
   ```

2. **Stop hardware scripts on remote launch platform:**
   ```bash
   ssh ${CLP_LP_HOST:-dev@clp-lp.local.local} "/home/linuxbrew/.linuxbrew/bin/socattui down"
   ssh ${CLP_LP_HOST:-dev@clp-lp.local.local} "/usr/bin/tmux kill-server"
   ```

3. **Verify shutdown:**
   ```bash
   ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/launch_platform_status | jq ."
   ```
   (Should return empty or error — no longer connected)
