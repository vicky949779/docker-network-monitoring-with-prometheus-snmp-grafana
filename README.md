# Full Stack Network Monitoring System (Prometheus + SNMP Exporter + Grafana)

This repository contains the complete configuration to set up a production-grade **Network Monitoring System** using Docker. It is designed to monitor Cisco, Arista, and VyOS devices, covering traffic, CPU, RAM, and health statistics. The file below is ready to copy/paste into a GitHub `README.md`.

---

## 📋 Table of Contents

1. [Prerequisites & Installation](#1-prerequisites--installation)
2. [Architecture Overview](#2-architecture-overview)
3. [Configuration Files](#3-configuration-files)

   * [docker-compose.yml](#31-docker-composeyml)
   * [snmp.yml (module definitions)](#32-snmpyml-module-definitions)
   * [prometheus.yml (job definitions)](#33-prometheusyml-job-definitions)
4. [Grafana Dashboard Setup](#4-grafana-dashboard-setup)

   * [Global Variables](#global-variables)
   * [Row 1: Health & Device Info](#row-1-health--device-info)
   * [Row 2: Traffic & Connection Trends](#row-2-traffic--connection-trends)
   * [Row 3: Interface Details Table](#row-3-interface-details-table)
5. [OID Reference Table (Cheat Sheet)](#5-oid-reference-table-cheat-sheet)

---

## 1. Prerequisites & Installation

### Requirements

* **OS:** Linux (Ubuntu/CentOS) or Windows with WSL2
* **Software:** Docker & Docker Compose
* **Network:** Connectivity to target devices on SNMP port `161/udp`

### Step-by-step Setup

#### Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER   # optional: run docker without sudo
```

#### Create directory structure

```bash
mkdir -p /opt/grafana/compose
mkdir -p /opt/grafana/prometheus
cd /opt/grafana/compose
```

#### Create files

Place the files described in Section 3 into the created directories (`docker-compose.yml`, `prometheus/prometheus.yml`, and a `snmp-exporter` build context that contains your `snmp.yml`).

#### Build & Start

```bash
# If snmp-exporter image bakes snmp.yml at build-time
sudo docker compose build snmp-exporter
sudo docker compose up -d
```

> **Note:** `network_mode: host` is used for the SNMP exporter so SNMP requests use host network stack (better reachability to devices). If running on a cloud VM, ensure firewall rules allow UDP/161 egress/ingress as required.

---

## 2. Architecture Overview

* **SNMP Exporter**: Translates SNMP OIDs into Prometheus metrics. Use 64-bit counters (`ifHC*`) for high-speed interfaces (e.g., 100G/400G).
* **Prometheus**: Scrapes the SNMP exporter. Default scrape interval in examples is 30s.
* **Grafana**: Dashboards built using PromQL queries and transformations for device-level views.

Typical flow: Device SNMP -> SNMP Exporter (module transforms) -> Prometheus scrape -> Grafana visualize.

---

## 3. Configuration Files

### 3.1 `docker-compose.yml`

```yaml
version: "3.8"
services:
  snmp-exporter:
    build:
      context: ../
    image: my-snmp-exporter:fixed
    restart: unless-stopped
    network_mode: "host"  # Crucial for SNMP reachability
    # If you prefer bridge mode, expose port mappings and ensure UDP 161 reachability

  prometheus:
    image: prom/prometheus:v2.53.0
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ../prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    volumes:
      - grafana-storage:/var/lib/grafana

volumes:
  grafana-storage:
```

> If you build snmp-exporter into a dedicated image, include your `snmp.yml` in the build context and COPY it into the image during Dockerfile build.

### 3.2 `snmp.yml` (Module Definitions)

Below is a compact but functional example of `snmp.yml` showing a split-module strategy. Put the full extended OID list in your repo file (you may have 200+ lines of OID definitions in production).

```yaml
# 1. AUTHENTICATION SECTION
# Defines the SNMP community string (Password) and Version.
auths:
  public_v2:
    community: PUBLIC
    version: 2

# 2. MODULES SECTION
# Each block below is a separate "Module" (Like a profile).
# You select which module to use in Prometheus config.
modules:

  # ------------------------------------------------------
  # MODULE A: GENERIC / BACKUP
  # Use this for unknown devices. Only basic Interface stats.
  # ------------------------------------------------------
  if_mib:
    walk:
      - 1.3.6.1.2.1.1.3        # List of OIDs to walk (collect)
      - 1.3.6.1.2.1.31.1.1.1   # Interface Names
      # ... (Add more global OIDs here)
    metrics:
      - name: sysUpTime        # Metric Name (Used in PromQL)
        oid: 1.3.6.1.2.1.1.3   # The OID
        type: gauge            # Data Type (gauge, counter, DisplayString)
      
      - name: ifName
        oid: 1.3.6.1.2.1.31.1.1.1
        type: DisplayString
        indexes:
          - labelname: ifIndex # The "Key" for this table
            type: gauge        # Index type (Always gauge for IF-MIB)

  # ------------------------------------------------------
  # MODULE B: CISCO DEVICES
  # Inherits nothing. We must repeat Global OIDs + Add Cisco ones.
  # ------------------------------------------------------
  cisco_router:
    walk:
      # --- GLOBAL OIDS (Must be repeated here) ---
      - 1.3.6.1.2.1.1.3
      - 1.3.6.1.2.1.31.1.1.1
      # --- CISCO SPECIFIC OIDS ---
      - 1.3.6.1.4.1.9.9.109.1.1.1.1.8 # Cisco CPU
      - 1.3.6.1.4.1.9.9.48.1.1.1.5    # Cisco Memory
    metrics:
      # ... (Define Global Metrics here again) ...
      
      # --- CISCO METRICS DEFINITION ---
      - name: ciscoCpu5Min
        oid: 1.3.6.1.4.1.9.9.109.1.1.1.1.8
        type: gauge
        indexes:
          - labelname: cpmCPUTotalIndex
            type: gauge

  # ------------------------------------------------------
  # MODULE C: LINUX DEVICES (Arista / VyOS)
  # Uses Host-Resources MIB for CPU/Mem
  # ------------------------------------------------------
  linux_device:
    walk:
      # --- GLOBAL OIDS ---
      - 1.3.6.1.2.1.1.3
      # --- LINUX SPECIFIC OIDS ---
      - 1.3.6.1.2.1.25.3.3.1.2 # hrProcessorLoad
    metrics:
      # ... (Define Global Metrics) ...
      
      # --- LINUX METRICS DEFINITION ---
      - name: hrProcessorLoad
        oid: 1.3.6.1.2.1.25.3.3.1.2
        type: gauge
        indexes:
          - labelname: hrDeviceIndex
            type: gauge

# Note: In production you will add many more metric definitions, OID translations,
# and mapping rules (e.g., ifIndex->ifName) for nicer labels.
```

> **Tip:** Always use `ifHCInOctets` / `ifHCOutOctets` (64-bit) for high-speed links to avoid counter wrap.

### 3.3 `prometheus/prometheus.yml` (Job Definitions)

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: 'snmp_cisco'
    params:
      module: [cisco_router]
    static_configs:
      - targets: ['192.100.100.1:161', '192.100.100.2:161']

  - job_name: 'snmp_arista'
    params:
      module: [arista_device]
    static_configs:
      - targets: ['192.0.2.1:161', '192.0.2.2:161']

  - job_name: 'snmp_vyos'
    params:
      module: [vyos_device]
    static_configs:
      - targets: ['198.51.100.5:161']

# Optional: alerting, remote_write, rule_files, etc.
```

---

## 4. Grafana Dashboard Setup

### Global Variables

* **Name:** `device`
* **Query:** `label_values(up{job=~"snmp.*"}, instance)`
* **Regex:** `/([^:]+)/`  (To strip port `:161` from instance values)

> Use the `device` variable across panels as `${device}`.

### ROW 1: Health & Device Info

#### 1) Device Identity Card (Table)

**PromQL / instant queries** (combine as multi-query panel and use transformations):

```promql
sysName{instance=~"${device}(:.*)?"}
sysDescr{instance=~"${device}(:.*)?"}
sysLocation{instance=~"${device}(:.*)?"}
sysContact{instance=~"${device}(:.*)?"}
entPhysicalSerialNum{instance=~"${device}(:.*)?", entPhysicalSerialNum!=""}
```

**Transformations (panel options)**

1. Reduce: (Series to rows, Last)
2. Labels to fields (Mode: Columns)
3. Merge series/tables
4. Organize fields: rename `sysName` → `Hostname`, etc.

#### 2) Uptime, CPU, RAM (Stat/Gauge)

* **Uptime (seconds):**

```promql
min(sysUpTime{instance=~"${device}.*"}) / 100
```

* **CPU Query (universal):**

```promql
avg(ciscoCpu5Min{instance=~"${device}.*"}) or avg(hrProcessorLoad{instance=~"${device}.*"})
```

* **RAM Query (Cisco OR Linux logic):**

Cisco-style logic (example):

```promql
(max(ciscoMemUsed{instance=~"${device}.*"}) / (max(ciscoMemUsed{instance=~"${device}.*"}) + max(ciscoMemFree{instance=~"${device}.*"}))) * 100
```

Host-Resources (hrStorage):

```promql
(sum(hrStorageUsed{instance=~"${device}.*", storageType=~".*ram.*"}) / sum(hrStorageSize{instance=~"${device}.*", storageType=~".*ram.*"})) * 100
```

### ROW 2: Traffic & Connection Trends

#### 1) Total Network Traffic History (Graph)

* **Download (bits/sec):**

```promql
sum(rate(ifHCInOctets{instance=~"${device}.*"}[2m])) * 8
```

* **Upload (bits/sec):**

```promql
sum(rate(ifHCOutOctets{instance=~"${device}.*"}[2m])) * 8
```

**Unit:** `bits/sec` (use `Data rate` in Grafana panel)

#### 2) Active Connections (Graph)

```promql
sum(tcpCurrEstab{instance=~"${device}.*"})
```

#### 3) Top Interface Utilization % (Bar Gauge / Time series)

```promql
(
  (rate(ifHCInOctets{instance=~"${device}(:.*)?", ifName!~"Lo.*|Nu.*|Vl.*"}[2m]) * 8)
  /
  (ifSpeed{instance=~"${device}(:.*)?", ifName!~"Lo.*|Nu.*|Vl.*"})
) * 100 > 0
```

**Legend:** `{{ifName}}`
**Format:** Time series. Use a meaningful unit and thresholds.

### ROW 3: Interface Details Table

The troubleshooting table that shows per-interface status, in/out rates and errors.

**Queries (instant, table-friendly):**

* **Status:**

```promql
ifOperStatus * on(instance, ifIndex) group_left(ifName) ifName
```

* **Download:**

```promql
rate(ifHCInOctets{instance=~"${device}(:.*)?"}[2m]) * 8
```

* **Upload:**

```promql
rate(ifHCOutOctets{instance=~"${device}(:.*)?"}[2m]) * 8
```

* **Errors:**

```promql
increase(ifInErrors{instance=~"${device}(:.*)?"}[2m]) + increase(ifOutDiscards{instance=~"${device}(:.*)?"}[2m])
```

**Transformations (order matters):**

1. Labels to fields (mode: columns)
2. Merge series/tables
3. Organize fields (hide internal labels, rename columns)

**Styling / Overrides:**

* `Status`: Value mapping (1=UP → Green, 2=DOWN → Red). Cell background color based on mapping.
* `Download/Upload`: Unit `bits/sec`. Use gradient gauge formatting.
* `Errors`: Threshold > 0 → Red highlight.

---

## 5. OID Reference Table (Cheat Sheet)

| OID                             |                   Name | Description                           | Type         |
| ------------------------------- | ---------------------: | ------------------------------------- | ------------ |
| `1.3.6.1.2.1.1.3`               |            `sysUpTime` | System uptime (hundredths of seconds) | Gauge        |
| `1.3.6.1.2.1.1.5`               |              `sysName` | Hostname                              | String       |
| `1.3.6.1.2.1.47.1.1.1.1.11`     | `entPhysicalSerialNum` | Serial number                         | String       |
| `1.3.6.1.2.1.31.1.1.1.1`        |               `ifName` | Interface name                        | String       |
| `1.3.6.1.2.1.2.2.1.8`           |         `ifOperStatus` | Status (1=up,2=down,...)              | Gauge / Enum |
| `1.3.6.1.2.1.31.1.1.1.6`        |         `ifHCInOctets` | 64-bit inbound octets                 | Counter64    |
| `1.3.6.1.2.1.31.1.1.1.10`       |        `ifHCOutOctets` | 64-bit outbound octets                | Counter64    |
| `1.3.6.1.2.1.2.2.1.5`           |              `ifSpeed` | Interface bandwidth (bps)             | Gauge        |
| `1.3.6.1.4.1.9.9.109.1.1.1.1.8` |         `ciscoCpu5Min` | Cisco CPU 5-min load (example)        | Gauge        |
| `1.3.6.1.4.1.9.9.48.1.1.1.5`    |         `ciscoMemUsed` | Cisco memory used                     | Gauge        |
| `1.3.6.1.2.1.25.3.3.1.2`        |      `hrProcessorLoad` | CPU load per core (Host-Resources)    | Gauge        |
| `1.3.6.1.2.1.25.2.3.1.6`        |        `hrStorageUsed` | Storage/RAM used (Host-Resources)     | Gauge        |

---

## OID Reference Table (Cheat Sheet) continue:-

## System
| Category | Name | Vendor Support | OID | Type | Description |
|---------|------|----------------|------|------|-------------|
| System | Device Name | All (Global) | 1.3.6.1.2.1.1.5 | String | Hostname (sysName) |
|        | Uptime | All (Global) | 1.3.6.1.2.1.1.3 | Gauge | System uptime |
|        | Description (OS) | All (Global) | 1.3.6.1.2.1.1.1 | String | System description (OS, hardware) |
|        | Serial Number | Cisco / Arista / Juniper | 1.3.6.1.2.1.47.1.1.1.1.11 | String | Device serial number |

## Interfaces
| Category | Name | Vendor Support | OID | Type | Description |
|---------|------|----------------|------|------|-------------|
| Interfaces | Interface Name | All (Global) | 1.3.6.1.2.1.31.1.1.1.1 | String | Interface name (ifName) |
|           | Description | All (Global) | 1.3.6.1.2.1.2.2.1.2 | String | Interface description |
|           | Status (Up/Down) | All (Global) | 1.3.6.1.2.1.2.2.1.8 | Gauge/Enum | 1=Up, 2=Down |
|           | Speed | All (Global) | 1.3.6.1.2.1.2.2.1.5 | Gauge | Bandwidth in bps |
|           | Traffic In (64-bit) | All (Global) | 1.3.6.1.2.1.31.1.1.1.6 | Counter | Inbound octets |
|           | Traffic Out (64-bit) | All (Global) | 1.3.6.1.2.1.31.1.1.1.10 | Counter | Outbound octets |
|           | Errors (In) | All (Global) | 1.3.6.1.2.1.2.2.1.14 | Counter | Interface input errors |
|           | Drops (Out Discards) | All (Global) | 1.3.6.1.2.1.2.2.1.19 | Counter | Output discards |

## Hardware
| Category | Name | Vendor Support | OID | Type | Description |
|---------|------|----------------|------|------|-------------|
| Hardware | CPU Load (5min) | Cisco | 1.3.6.1.4.1.9.9.109.1.1.1.1.8 | Gauge | CPU 5-min avg |
|          | Memory Used | Cisco | 1.3.6.1.4.1.9.9.48.1.1.1.5 | Gauge | Memory usage |
|          | CPU Load (Core) | Arista / VyOS / Linux | 1.3.6.1.2.1.25.3.3.1.2 | Gauge | CPU load per core |
|          | Memory/Storage | Arista / VyOS / Linux | 1.3.6.1.2.1.25.2.3.1.6 | Gauge | Storage/RAM used |
|          | CPU (Routing Engine) | Juniper | 1.3.6.1.4.1.2636.3.1.3 | Gauge | Juniper RE CPU |
|          | RAM (Routing Engine) | Juniper | 1.3.6.1.4.1.2636.3.1.2 | Gauge | Juniper RE memory |
|          | Temperature | Cisco | 1.3.6.1.4.1.9.9.13.1.3 | Gauge | Temperature sensors |

## Routing
| Category | Name | Vendor Support | OID | Type | Description |
|---------|------|----------------|------|------|-------------|
| Routing | BGP Peer Status | All (Global) | 1.3.6.1.2.1.15.3.1.2 | Gauge | 1=Idle, 2=Connect, 6=Established |
|         | OSPF Neighbor Status | All (Global) | 1.3.6.1.2.1.14.10.1.6 | Gauge | OSPF neighbor state |
|         | EIGRP Neighbor Status | Cisco Only | 1.3.6.1.4.1.9.9.449.1.4.1.1.3 | Gauge | EIGRP neighbor state |

## Diagnosis / Troubleshooting
| Category | Name | Vendor Support | OID | Type | Description |
|---------|------|----------------|------|------|-------------|
| Diagnosis | Routing Table Dest | All (Troubleshoot) | 1.3.6.1.2.1.4.24.4.1.1 | IpAddress | Route destination |
|          | Route Protocol | All (Troubleshoot) | 1.3.6.1.2.1.4.24.4.1.7 | Integer | Protocol type |
|          | TCP Connections | All (Global) | 1.3.6.1.2.1.6.9 | Gauge | Current established TCP sessions |

---
## Build & Start Services

> **Must rebuild to bake `snmp.yml` into the image**

```bash
sudo docker compose build snmp-exporter
```

### Start in detached mode

```bash
sudo docker compose up -d
```

---

## Operations & Debugging Cheat Sheet

### Service Management

* Check running containers

```bash
sudo docker ps
```

* View logs (example container name from compose)

```bash
sudo docker logs compose-snmp-exporter-1
```

* Reload Prometheus (send SIGHUP to Prometheus container)

```bash
sudo docker kill -s HUP compose-prometheus-1
```

---

### Backup

```bash
sudo cp /opt/grafana/snmp.yml /opt/grafana/snmp.yml.backup
```

---

### Terminal Testing (CLI)

#### Test SNMP reachability (snmpwalk)

```bash
snmpwalk -v 2c -c PUBLIC <DEVICE_IP> 1.3.6.1.2.1.1.3.0
```

#### Test SNMP exporter module (curl)

```bash
curl -s "http://localhost:9116/snmp?target=<DEVICE_IP>:161&auth=public_v2&module=<MODULE_NAME>"
```

#### Check specific protocol (BGP) via SNMP

```bash
snmpwalk -v 2c -c PUBLIC <DEVICE_IP> 1.3.6.1.2.1.15.3.1.2
```

---

### Quick troubleshooting tips

* If `curl` to exporter returns no metrics, verify exporter is reachable and UDP/161 is allowed between host and device.
* Use `sudo tcpdump -n -i <if> udp port 161` on host to observe SNMP traffic during tests.
* If Prometheus doesn't pick up a target, confirm `prometheus.yml` target IP:port and module param match and `snmp-exporter` is serving on expected port.

---

## Notes & Best Practices

* **Use 64-bit counters** (`ifHCInOctets` / `ifHCOutOctets`) for all 10G+ interfaces.
* **Indexes → Labels:** Ensure `ifIndex` -> `ifName` mapping exists in `snmp.yml` for human-readable dashboards.
* **SNMP versions:** Prefer SNMPv3 for production (authentication + encryption). Add `auth`/`priv` details to Prometheus job targets or SNMP exporter relabeling as required.
* **Firewall & MTU:** SNMP over UDP needs to cross the network path. If you have intermediate devices, ensure management ACLs and MTU allow SNMP traffic.
* **Scaling:** If monitoring thousands of interfaces, consider sharding targets across multiple Prometheus instances or using remote storage.

---

## Attribution

Documentation created by Vigneshkumar.


