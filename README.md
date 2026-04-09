# truenas-scale-25x-grafana-dashboard

A Grafana dashboard and Prometheus graphite-exporter mapping file built specifically for **TrueNAS SCALE 25.x (Goldeye)**. This project was created because existing community dashboards were built for older TrueNAS versions and are incompatible with the metric format changes introduced in 25.x.

Tested on TrueNAS SCALE 25.10.2.1.

> This project was created with significant assistance from Claude (Anthropic). The mapping file and dashboard were built through iterative testing against a live TrueNAS 25.x system, debugging metric pipeline issues specific to this version, and validating every panel against real data.

---
## Screenshots

<img width="1383" height="1081" alt="image" src="https://github.com/user-attachments/assets/6ac0e7a5-2d2a-4047-b216-565f7b1f3aff" />
<img width="1380" height="1063" alt="image" src="https://github.com/user-attachments/assets/9748c339-82e4-4a3a-801e-4c0deb0d7217" />
<img width="1380" height="315" alt="image" src="https://github.com/user-attachments/assets/53782fef-7f97-4934-8d16-97006bcb1822" />
<img width="1381" height="687" alt="image" src="https://github.com/user-attachments/assets/c31d41dc-b8d8-4f16-ae3f-25f1f6d6488c" />
<img width="1376" height="271" alt="image" src="https://github.com/user-attachments/assets/ac5e7bc2-45a0-4918-90f5-b112d4043dae" />
<img width="1380" height="608" alt="image" src="https://github.com/user-attachments/assets/b0338d98-0b86-4112-b7ac-2942bb272137" />
<img width="1387" height="611" alt="image" src="https://github.com/user-attachments/assets/d53422fd-e3d7-44b7-83ee-c0bedc8369e9" />
> (Last Image show Cgroup / Container Resources is serving old data. For the name pbs (system) and pbs (user) is what you would see for your containers.)

## Overview

TrueNAS does not expose a native Prometheus endpoint. Instead it pushes metrics via the Graphite protocol. This project provides:

- **`graphite_mapping.conf`** — Maps the raw Graphite metric paths from TrueNAS 25.x into clean, labelled Prometheus metrics. 503 mapped metrics producing 63 unique metric names.
- **`dashboard.json`** — A Grafana dashboard that queries those metrics across 9 sections covering the full system.

---

## What is Monitored

### CPU
- Per-core usage (%)
- Per-core temperature (°C)
- System load average (1m / 5m / 15m)
- Total average CPU usage (overview stat)

### Memory
- Total, available, and used (bytes over time)
- Memory used % (gauge with thresholds)

### Network
- Per-interface I/O with a dynamic interface selector dropdown — no hardcoded NIC names
- System-wide combined network I/O
- Active link state (shows only UP interfaces)
- Packet drops per interface

### Storage
- Per-disk read/write throughput (KB/s) — pool disks and SSDs in separate panels
- Per-disk busy % 
- Per-disk IOPS (read and write)
- Per-disk temperature — HDD pool disks and SSDs/non-pool disks separated automatically

### ZFS Pool
- Pool used % and available % (gauges with colour thresholds — green to red as capacity fills)
- ARC size, free, and available
- ARC demand data and metadata hit/miss ratios and percentages
- L2ARC hits, misses, bytes, and hit/miss percentages
- Total demand reads

### NFS
- Read/write I/O throughput
- RPC calls and active threads
- Read cache hits/misses/nocache
- File handle statistics
- NFSv4 procedure and operation counts

### Services
- Per-service CPU usage
- Per-service memory usage (MB)
- Per-service I/O (read and write)
- Per-service IOPS (read and write)

### Cgroup / Containers
- CPU usage per container (system and user time shown separately)
- Memory usage per container
- I/O per container (read and write)

Container names are extracted automatically from the metric path. New containers appear in the dashboard without any mapping changes.

---

## Requirements

- TrueNAS SCALE 25.04 or 25.10.x
- Prometheus
- Grafana
- `prom/graphite-exporter` Docker image

The graphite-exporter, Prometheus, and Grafana can run on any host that has network access to TrueNAS. A lightweight Alpine Linux VM works well.

---

## Setup

### 1. Configure the graphite-exporter

The graphite-exporter **must** be started with `--graphite.sample-expiry=60m`. TrueNAS 25.x sends some metrics (disk temperatures, pool usage) with timestamps that are already 10–15 minutes old by the time they arrive. The default 5-minute expiry silently drops these metrics, resulting in empty panels.

Example `docker-compose.yml` entry:

```yaml
graphite-exporter:
  image: prom/graphite-exporter:latest
  container_name: graphite-exporter
  volumes:
    - ./graphite_mapping.conf:/tmp/graphite_mapping.conf
  command: >
    --graphite.mapping-config=/tmp/graphite_mapping.conf
    --graphite.sample-expiry=60m
  ports:
    - "9109:9109"   # Prometheus scrape port
    - "2003:2003/udp"
    - "2003:2003/tcp"
  restart: unless-stopped
```

### 2. Configure Prometheus

Add a scrape job for the graphite-exporter:

```yaml
scrape_configs:
  - job_name: 'truenas'
    static_configs:
      - targets: ['graphite-exporter:9109']
    scrape_interval: 1m
```

### 3. Configure TrueNAS

In TrueNAS go to **Reporting -> Exporters -> Add** and set:

| Field | Value |
|---|---|
| Type | GRAPHITE |
| Destination IP | IP of your graphite-exporter host |
| Destination Port | 2003 |
| Prefix | (leave blank) |
| Namespace | truenas |
| Send Names Instead Of IDs | Enabled |

TrueNAS 25.x hardcodes a `scale.` prefix regardless of settings. Leaving Prefix blank with Namespace `truenas` produces metric paths in the format `scale.truenas.<chart>.<dimension>`, which is exactly what this mapping file expects.

### 4. Import the dashboard

In Grafana go to **Dashboards -> New -> Import**, upload `dashboard.json`, and set the Prometheus datasource when prompted.

On first load, select your network interface from the **Interface** dropdown at the top of the dashboard. This dropdown is populated automatically from your data — no editing required.

---

## Disk Panel Behaviour

The storage panels use regex patterns to separate disk types without needing any configuration:

- **Pool disks (HDD panel)** — matches `_serial_lunid_` serials beginning with `2TK`, which is the lunid prefix pattern for SAS/SATA spinning disks in most configurations.
- **SSDs / Non-pool panel** — matches `_serial_lunid_S` serials (SSD lunid format) and any `_serial_` entry that is not a lunid serial (e.g. bare SATA SSDs and boot devices).

If your pool disks use a different lunid prefix you may need to adjust the regex in the HDD panels. Check your disk labels with:

```sh
curl -s "http://localhost:9090/api/v1/label/disk/values?match[]=truenas_disk_temperature" | python3 -m json.tool
```

---

## Known Behaviour

**Stale series after mapping changes** — If you replace the mapping file, old series from the previous mapping take up to 60 minutes to expire from Prometheus (matching the `--graphite.sample-expiry=60m` setting). During this window you may see duplicate or oddly-labelled series in some panels. This is normal and resolves on its own.

**Timestamp skew on pool and memory metrics** — TrueNAS pushes the `available` and `total` variants of pool and memory metrics at slightly different timestamps. All division queries in this dashboard use `ignoring(type)` to handle this correctly. If you write custom queries against these metrics, apply the same pattern.

**Cgroup metrics are TrueNAS-managed containers only** — The cgroup panels reflect containers running within TrueNAS's own cgroup hierarchy (Apps, jails, etc.). Docker containers on a separate host will not appear here.

---

## License

MIT License — see `LICENSE` for details.
