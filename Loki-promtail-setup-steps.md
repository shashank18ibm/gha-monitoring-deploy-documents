**Loki/promtail logs setup in gha stage/prod:-**





**Getting logs from workers using below commands:**



df -h

podman stats $CONTAINER --no-stream --format

podman logs $CONTAINER --tail 20

systemctl status $CONTAINER --no-pager

journalctl -u $CONTAINER -n 20 --no-pager

podman inspect $CONTAINER --format

---



**What’s now covered**

1.System metrics \& logs

2.systemctl status

3.journalctl

4.Disk usage (df)

5.Top CPU processes

6.Total running containers

7.Podman container logs (--tail 100)

8.Podman stats (CPU%, MEM, NET, BLOCK)

9.LXD console logs (Promtail reads directly from /var/snap/lxd/common/lxd/logs/\*/console.log)

10.LXD container lifecycle logs (lxc.log)

11.Old logs auto-deleted (system logs overwritten, push logs >12h removed once they are pushed to loki. Further will plan to set logs retention period in loki between 1-90 days after discussion with dev team.

#      



**What’s happening now**



System logs (lxd-app-system.log)

Collected every 30 min via lxd-cmd-system.timer --> includes systemctl status, journalctl (last 20 lines), podman inspect, df -h, top output.

Container logs (podman logs + podman stats)

Collected every 5 min via lxd-cmd-container.timer.

LXD console logs

Not moved anywhere; Promtail can read them directly from /var/snap/lxd/common/lxd/logs/ if added to promtail.yaml.

Promtail

Reads system logs, container logs, container stats, and can be configured to read LXD logs directly.

Pushes everything to Loki.

Old logs (>12 hours) are automatically deleted from $PUSH\_DIR.


\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

##########################################################################################################################################

**Below are the steps which we follow on each workers:- 12 Step installation guide:**

**--------------------------------------------------------------------------------------------------------------------------------------------**

**\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\***



**Pormtail installation steps:(till step 5):**
----------------------------------------------

**Step 1:**

sudo dnf update -y
sudo dnf install -y git curl tar gzip jq gcc make


---------------------------------------------------
**Step-2: Install Go (ppc64le compatible)**

cd /opt
sudo wget https://go.dev/dl/go1.23.1.linux-ppc64le.tar.gz
sudo tar -C /usr/local -xzf go1.23.1.linux-ppc64le.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version



**Step-3: Clone Loki Repository**

cd /opt \&\& sudo mkdir -p src \&\& sudo chown "$(id -u)":"$(id -g)" src
cd /opt/src
git clone https://github.com/grafana/loki.git
cd loki
LOKI\_TAG=$(git describe --tags --abbrev=0 || true); echo "Using tag: ${LOKI\_TAG}"
if \[ -n "$LOKI\_TAG" ]; then git checkout "$LOKI\_TAG"; fi

export LOKI_TAG="helm-loki-6.44.0"




**Step-4: Build Promtail**



cd clients/cmd/promtail
go build -trimpath -o promtail .
./promtail --version
sudo mv ./promtail /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
echo 'export PATH=/usr/local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
promtail --version
sudo mkdir -p /etc/promtail





**Step5:  create promtail system service:**
----------------------------------------

sudo tee /etc/systemd/system/promtail.service > /dev/null <<EOF
\[Unit]
Description=Promtail Service
After=network.target
\[Service]
ExecStart=/usr/local/bin/promtail --config.file=/etc/promtail/promtail.yaml
Restart=always
\[Install]
WantedBy=multi-user.target
EOF





################################################################################################################################
#####################################################################################################################################################################################################################################################



=====================================================================================================================



**Step 6:

vi /etc/systemd/system/lxd-cmd-system.service


[Unit]
Description=Collect LXD System Logs

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lxd_cmd_logs.sh system





-----------------------------------------------------------------

**Step 7:**


vi /etc/systemd/system/lxd-cmd-system.timer


[Unit]
Description=Run LXD System Log Collector Every 30 Minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=30min
Unit=lxd-cmd-system.service

[Install]
WantedBy=timers.target



---------------------------------------------------------------------

**step 8:**



vi /etc/systemd/system/lxd-cmd-container.service

\[Unit]
Description=Collect LXD Container Logs \& Stats (Only on new container)
\[Service]
Type=oneshot
ExecStart=/usr/local/bin/lxd\_cmd\_logs.sh container

-------------------------------------------------------------------------



**step 9:**



vi /etc/systemd/system/lxd-cmd-container.timer



[Unit]
Description=Collect LXD Container Logs & Stats (Only on new container)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lxd_cmd_logs.sh container

----------------------------------------------------------------------------



**step 10:**


After above scripts then run below to start above services:


mkdir -p /var/log/lxd-app/push

touch /var/log/lxd-app/lxd-app-system.log

chmod 644 /var/log/lxd-app/lxd-app-system.log

chmod +x lxd-app/

sudo chmod +x /usr/local/bin/lxd\_cmd\_logs.sh

sudo systemctl daemon-reload

sudo systemctl enable --now lxd-cmd-system.timer

sudo systemctl enable --now lxd-cmd-container.timer

systemctl daemon-reload

systemctl start  lxd-cmd-system.timer

systemctl start  lxd-cmd-container.timer

systemctl start lxd-cmd-system.service

systemctl start lxd-cmd-container.service





----------------------------------------------------------------------------------------------------------

**run this command to check all good---> systemctl list-timers | grep lxd**   ------------this give output --> to check if above services are running---->



expected output---> lxd-cmd-system.timer       lxd-cmd-system.service     ... 30m

lxd-cmd-container.timer    lxd-cmd-container.service  ... 5m





\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

\############################################################################################################################



**step 11:**



vi /etc/promtail/promtail.yaml


server:
  http_listen_port: 9081
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: "https://loki-custom-monitoring.gha-prod-sre-cluster-us-s-d3463d8f849ebe06c018b03078ea2661-0000.us-south.containers.appdomain.cloud/loki/api/v1/push"
    batchwait: 15s
    batchsize: 1048576
    timeout: 30s
    tls_config:
      insecure_skip_verify: true

scrape_configs:

  # ----------------------------
  # 1️⃣ System logs (journal + systemctl + metrics)
  # ----------------------------
  - job_name: lxd-app-system
    static_configs:
      - targets: [localhost]
        labels:
          job: "lxd-app-system"

          __path__: /var/log/lxd-app/lxd-app-system.log

  # ----------------------------
  # 2️⃣ Container logs (application logs)
  # ----------------------------
  - job_name: lxd-app-container
    static_configs:
      - targets: [localhost]
        labels:
          job: "lxd-app-container"

          __path__: /var/log/lxd-app/push/*-podman.log

  # ----------------------------
  # 3️⃣ Container stats logs (CPU, MEM, NET, BLOCK)
  # ----------------------------
  - job_name: lxd-app-stats
    static_configs:
      - targets: [localhost]
        labels:
          job: "lxd-app-stats"

          __path__: /var/log/lxd-app/push/*-podman-stats.log

  # ----------------------------
  # 4️⃣ LXD console logs (direct read, no copy)
  # ----------------------------
  - job_name: lxd-console
    static_configs:
      - targets: [localhost]
        labels:
          job: "lxd-console"

          __path__: /var/snap/lxd/common/lxd/logs/*/console.log

  # ----------------------------
  # 5️⃣ Optional: LXD container lifecycle logs
  # ----------------------------
  - job_name: lxd-lifecycle
    static_configs:
      - targets: [localhost]
        labels:
          job: "lxd-lifecycle"

          __path__: /var/snap/lxd/common/lxd/logs/*/lxc.log

-------------------------------------------------------------------------------------------------------

#########################################################################################################################################################################################################################################################################################################################################################################################################################################


Step:12 ------:

**Then run below:**



mkdir -p /var/lib/promtail

chmod 755 /var/lib/promtail
sudo systemctl daemon-reload

sudo systemctl enable --now promtail

sudo systemctl status promtail

 

###################################################################################################################################################################################################################################################

==========================================================================================================================================

\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

------------------------------------------------------------------------------------


**Step 12:**

vi /usr/local/bin/lxd_cmd_logs.sh



#!/bin/bash
# File: /usr/local/bin/lxd_cmd_logs.sh
# Purpose: Collect system logs, container logs, podman stats, system metrics
# Rotation: Deletes old logs automatically
# Logs directory
LOG_DIR="/var/log/lxd-app"
PUSH_DIR="$LOG_DIR/push"
STATE_DIR="$LOG_DIR/state"
CONTAINER_NAME="lxd-app"

mkdir -p "$LOG_DIR" "$PUSH_DIR" "$STATE_DIR"

SYSTEM_LOG="$LOG_DIR/lxd-app-system.log"
LAST_ID_FILE="$STATE_DIR/last_container_id"

###############################################################################
# 1️⃣ Collect system logs and metrics (replace every 30 mins)
###############################################################################
collect_system_logs() {
  {
    echo "===== $(date '+%Y-%m-%d %H:%M:%S') ====="

    # Systemctl status of main container/service
    echo "--- systemctl status $CONTAINER_NAME ---"
    systemctl status $CONTAINER_NAME --no-pager 2>/dev/null | head -n 20

    # Journalctl logs for main container/service
    echo "--- journalctl logs (last 50 lines) ---"
    journalctl -u $CONTAINER_NAME -n 50 --no-pager 2>/dev/null

    # Podman inspect of container
    echo "--- podman inspect ---"
    podman inspect $CONTAINER_NAME --format "State={{.State.Status}} PID={{.State.Pid}} Started={{.State.StartedAt}}" 2>/dev/null

    # Disk usage of container storage
    echo "--- Disk Usage (/var/lib/containers/storage) ---"
    df -h /var/lib/containers/storage

    # Total running containers
    echo "--- Total running containers ---"
    podman ps -q | wc -l

    # Top 20 CPU consuming processes
    echo "--- Top 20 processes ---"
    top -b -n1 | head -n 20

    echo
  } > "$SYSTEM_LOG"  # Replace each run
}

###############################################################################
# 2️⃣ Collect container logs every 5 minutes
###############################################################################
collect_container_logs() {
  TIMESTAMP=$(date '+%Y%m%d-%H%M%S')

  # Collect logs from all running containers matching the name
  podman ps --filter "name=$CONTAINER_NAME" --format "{{.ID}}" | while read -r CONTAINER_ID; do
    [ -z "$CONTAINER_ID" ] && continue

    STATS_LOG="$PUSH_DIR/${TIMESTAMP}-${CONTAINER_ID}-podman-stats.log"
    APP_LOG="$PUSH_DIR/${TIMESTAMP}-${CONTAINER_ID}-podman.log"

    # ---- PODMAN STATS ----
    {
      echo "===== $(date '+%Y-%m-%d %H:%M:%S') ====="
      podman stats $CONTAINER_ID --no-stream --format "Name={{.Name}} CPU={{.CPUPerc}} MEM={{.MemUsage}} NET={{.NetIO}} BLOCK={{.BlockIO}}"
      echo
    } > "$STATS_LOG"

    # ---- APPLICATION LOGS ----
    {
      echo "===== $(date '+%Y-%m-%d %H:%M:%S') ====="
      podman logs $CONTAINER_ID --tail 100 2>/dev/null
      echo
    } > "$APP_LOG"
  done

  # ---- CLEAN OLD LOGS (>12 hours) ----
  find "$PUSH_DIR" -type f -mmin +720 -name "*.log" -exec rm -f {} \;
}

###############################################################################
# MAIN EXECUTION
###############################################################################
case "$1" in
  system)
    collect_system_logs
    ;;
  container)
    collect_container_logs
    ;;
  *)
    echo "Usage: $0 {system|container}"
    ;;
esac




##############################################################################################################################################################################################################################################################

                        



After all check services and logs collection are working :


journalctl -u lxd-cmd-system.service -n 20 --no-pager

journalctl -u lxd-cmd-container.service -n 20 --no-pager



%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-------------------------------------------End of installation and all process---------------------------------------------

$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$###################################################################################################################################################################@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%






----------------------------------------------------------------------
Grafana important queries for logs:


{job="lxd-app-monitor"}

{job="lxd-app-stats"} |= "CPU"

{job="lxd-app-stats", instance="gha-staging-us-south-worker-3-blue"} |= "CPU"

&nbsp;

{job="lxd-app-app"} |= "ERROR"

{job="lxd-app-app", instance="gha-staging-us-south-worker-3-blue"} |= "ERROR"

{job="lxd-app-app"} |~ "Exception|ERROR|Failed"

{job="lxd-app-system"} |= "Active: active"

{job="lxd-app-system", instance="gha-staging-us-south-worker-3-blue"} |= "Active: active"

{job="lxd-app-system"} |= "overlay" | logfmt              -----> for disk usage and alert set

{job="lxd-app-stats", instance="$worker"} |= "CPU"


-------------------------------------------------------------------------------------------

currently working queries:
{job="lxd-app-system"} |~ "Active:"

{job="lxd-app-stats"} |= "CPU"

{job="lxd-app-system"} |~ "Disk Usage"

{job="lxd-app-system"} |~ "%Cpu"

{job="lxd-app-system"} |= "overlay" | logfmt





====================================================================================================================





+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Some Important commands:

---





commands:

 du -sh /var/log/lxd-app/\*    ---> to check which file has occupied how much storage

 sudo systemctl restart promtail

  sudo systemctl daemon-reload

 sudo journalctl -u promtail -f



 sudo tee /etc/systemd/system/lxd-cmd-logs.service > /dev/null <<'EOF'

 crontab -e

sudo systemctl enable --now lxd-cmd-logs.timer

 systemctl list-timers | grep lxd-cmd-logs

  ls -lh /tmp/cmd\_logs/

---

**file locations:**



/etc/promtail/promtail.yaml

/usr/local/bin/lxd\_cmd\_logs.sh



/var/lib/promtail/positions.yaml

 /etc/systemd/system/lxd-cmd-logs.service  and lxd-cmd-logs.timer  --> used to run /usr/local/bin/lxd\_cmd\_logs.sh  and collect logs every 5 min and store it     in /var/log/lxd-app/lxd-app-monitor.log  and other logs file

