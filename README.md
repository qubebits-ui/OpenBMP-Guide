# OpenBMP Deployment & Management Guide

This repository provides deployment, configuration, and operational guidance for running a complete OpenBMP stack using Docker.

OpenBMP is an open-source BGP monitoring platform that collects BMP (BGP Monitoring Protocol) telemetry from routers, processes routing information through Apache Kafka, and stores both current routing state and historical BGP updates in PostgreSQL/TimescaleDB for analysis and visualization. It supports large-scale BGP monitoring, route analytics, routing security validation (RPKI/IRR), historical route tracking, and real-time network visibility.

This repository focuses on:

* Deploying OpenBMP using Docker Compose
* Integrating PostgreSQL/TimescaleDB, Kafka, and Grafana
* Configuring routers for BMP export
* Enabling GeoIP and RPKI enrichment
* Managing data retention and storage utilization
* Operational procedures and troubleshooting
* Production sizing recommendations for ISP and enterprise networks

The deployment stack includes:

* OpenBMP Collector
* Apache Kafka
* PostgreSQL / TimescaleDB
* OpenBMP PSQL Application
* Grafana

## 📚 Official OpenBMP Resources

* OpenBMP Website: [OpenBMP Website](https://www.openbmp.org/overview.html?utm_source=chatgpt.com)
* OpenBMP Documentation: [OpenBMP Documentation](https://www.openbmp.org/docs/?utm_source=chatgpt.com)
* OpenBMP Installation & Configuration: [Install & Configuration Guide](https://www.openbmp.org/install_config/?utm_source=chatgpt.com)
* OpenBMP Getting Started Guide: [Getting Started Guide](https://www.openbmp.org/getting_started.html?utm_source=chatgpt.com)
* OpenBMP Container Configuration: [Container Configuration Guide](https://www.openbmp.org/install_config/containers.html?utm_source=chatgpt.com)
* OpenBMP APIs & Kafka Schemas: [OpenBMP APIs](https://www.openbmp.org/api/?utm_source=chatgpt.com)

> This repository is not an official OpenBMP project. It provides deployment examples, operational procedures, and best practices built on top of the OpenBMP platform.


---

# 🚀 Architecture Overview

```text
+-----------+      BMP (TCP/5000)      +------------+
|  Router   | -----------------------> | Collector  |
+-----------+                          +------------+
                                             |
                                             v
                                      +-------------+
                                      |   Kafka     |
                                      +-------------+
                                             |
                                             v
                                      +-------------+
                                      |  PSQL-App   |
                                      +-------------+
                                             |
                                             v
                                      +-------------+
                                      | PostgreSQL  |
                                      +-------------+
                                             |
                                             v
                                      +-------------+
                                      |  Grafana    |
                                      +-------------+
```

### Components

| Component  | Function                                  |
| ---------- | ----------------------------------------- |
| Router     | Streams BGP/BMP telemetry                 |
| Collector  | Converts BMP messages into JSON           |
| Kafka      | High-speed message buffering              |
| PSQL-App   | Data enrichment (GeoIP, RPKI, IRR)        |
| PostgreSQL | Stores BGP telemetry and time-series data |
| Grafana    | Dashboards and visualization (Port 3000)  |

---

# 🛠 Prerequisites

## Operating System

* Ubuntu 22.04 LTS or newer
* Debian 11 or newer

## Hardware Requirements

### Minimum
For up to 2 routers exporting BMP Post-Policy feeds with full Internet routing tables:
* 4 vCPU
* 8 GB RAM
* 80 GB SSD

### Recommended

* 8 vCPU
* 16 GB RAM
* 200+ GB SSD

> Recommended for full Internet routing tables and long-term retention.

## Software

Install:

* Docker Engine
* Docker Compose V2

Verify installation:

```bash
docker --version
docker compose version
```

---

# 📥 Installation

## 1. Create Directory Structure

```bash
sudo mkdir -p /opt/openbmp/postgres/{data,ts}
sudo mkdir -p /opt/openbmp/{zk-data,zk-log,kafka-data,config,grafana}

sudo chown -R 472:472 /opt/openbmp/grafana
sudo chmod -R 777 /opt/openbmp/postgres
sudo chmod -R 777 /opt/openbmp/kafka-data
```

---

## 2. Configure Environment

Create the environment file:

```bash
echo "OBMP_DATA_ROOT=/opt/openbmp" > /opt/openbmp/.env
```

Verify:

```bash
cat /opt/openbmp/.env
```

Expected output:

```text
OBMP_DATA_ROOT=/opt/openbmp
```

---

## 3. Deploy the Stack

Place your `docker-compose.yml` file inside:

```text
/opt/openbmp/
```

Start all services:

```bash
cd /opt/openbmp

docker compose up -d
```

Check status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

---

# 📡 Adding a Device (Router Configuration)

Configure your router to export BMP telemetry to the OpenBMP Collector.

## Cisco IOS-XR

```ios
bmp server 1
 host 192.168.1.10 port 5000
 description OpenBMP-Collector
 update-source Loopback0
!

router bgp 65000
 neighbor 1.1.1.1
  address-family ipv4 unicast
   bmp-activate server 1
```

---

## Juniper JunOS

```junos
set routing-options bmp station OPEN-BMP hold-down 30
set routing-options bmp station OPEN-BMP hold-down flaps 3
set routing-options bmp station OPEN-BMP hold-down period 60
set routing-options bmp station OPEN-BMP local-address 10.10.10.1
set routing-options bmp station OPEN-BMP connection-mode active
set routing-options bmp station OPEN-BMP monitor enable
set routing-options bmp station OPEN-BMP route-monitoring post-policy
set routing-options bmp station OPEN-BMP station-address 192.168.10.10
set routing-options bmp station OPEN-BMP station-port 10179

```

---

# 🌍 GeoIP & RPKI Configuration

OpenBMP v2.2.x supports automatic GeoIP imports.

## Enable GeoIP and RPKI

Edit the `psql-app` section in your `docker-compose.yml`:

```yaml
environment:
  - ENABLE_DBIP=1
  - ENABLE_RPKI=1
```

Restart services:

```bash
docker compose up -d
```

---

## Verify GeoIP Import

```bash
docker exec -it obmp-psql \
psql -U openbmp -d openbmp \
-c "SELECT count(*) FROM geo_ip;"
```

Expected output:

```text
 count
-------
 XXXXX
```

A non-zero value indicates GeoIP data has been loaded successfully.

---

# 📊 Accessing Grafana

Grafana is exposed on:

```text
http://SERVER-IP:3000
```

Check container status:

```bash
docker ps | grep grafana
```

---

# 💾 Data Retention & Disk Optimization

To reduce disk usage and automatically remove older data, configure the PostgreSQL retention settings in your `docker-compose.yml` file.

For deployments with limited storage (80–100 GB SSD), the following values are recommended:

```yaml
environment:
  - POSTGRES_DROP_peer_event_log='1 month'
  - POSTGRES_DROP_stat_reports='14 days'
  - POSTGRES_DROP_ip_rib_log='14 days'
  - POSTGRES_DROP_stats_chg_byprefix='14 days'
```

### Apply Changes

After updating the retention values, restart the stack:

```bash
docker compose up -d
```

### Retention Guidelines

* Shorter retention periods reduce disk consumption.
* Longer retention periods preserve more historical data but require additional storage.
* Adjust the values based on available disk space and data retention requirements.


---
# 🧹 Manual Database Cleanup

If disk usage becomes high and you need to reclaim space immediately, you can manually remove older OpenBMP data from the database.

## Access PostgreSQL

Connect to the OpenBMP database:

```bash
docker exec -it obmp-psql psql -U openbmp -d openbmp
```

You should see the PostgreSQL prompt:

```sql
openbmp=#
```

## Remove Old Data

Run the following commands to delete historical data older than the specified retention periods:

```sql
-- Delete BGP Prefix logs older than 7 days
SELECT drop_chunks(relation => 'ip_rib_log', older_than => INTERVAL '7 days');

-- Delete Router Statistics reports older than 7 days
SELECT drop_chunks(relation => 'stat_reports', older_than => INTERVAL '7 days');

-- Delete Prefix Change statistics older than 7 days
SELECT drop_chunks(relation => 'stats_chg_byprefix', older_than => INTERVAL '7 days');

-- Delete Peer RIB statistics older than 7 days
SELECT drop_chunks(relation => 'stats_peer_rib', older_than => INTERVAL '7 days');

-- Delete BGP Peer Event logs older than 1 month
SELECT drop_chunks(relation => 'peer_event_log', older_than => INTERVAL '1 month');
```

## Optimize Database

After removing old data, run:

```sql
VACUUM ANALYZE;
```

## Exit PostgreSQL

```sql
\q
```

## Verify Available Disk Space

Check disk usage on the host:

```bash
df -h
```

> Adjust the retention intervals as needed to match your storage capacity and data retention requirements.


---

# 🔍 Useful Commands

## Container Status

```bash
docker compose ps
```

## Restart All Services

```bash
docker compose restart
```

## Stop the Stack

```bash
docker compose down
```

## View Logs

```bash
docker compose logs -f
```

## PostgreSQL Shell

```bash
docker exec -it obmp-psql psql -U openbmp -d openbmp
```

## Kafka Container Shell

```bash
docker exec -it obmp-kafka bash
```

---

# 📈 Recommended Production Sizing

The following sizing recommendations are based on routers exporting **BMP Post-Policy routes only** and receiving a **full Internet routing table**.

### Full Internet Routing Table Reference

> These sizing recommendations assume BMP Post-Policy route monitoring only and full Internet routing tables (~1.16M IPv4 routes and ~270K IPv6 routes per router).
>
> BMP Pre-Policy monitoring can increase update volume, database growth, CPU utilization, memory usage, and storage requirements significantly.hese recommendations assume **Post-Policy BMP monitoring only**. Enabling **Pre-Policy BMP** can significantly increase message volume, database growth, CPU utilization, and storage requirements.

| Deployment Size | Routers          | CPU      | RAM    | Storage |
| --------------- | ---------------- | -------- | ------ | ------- |
| Small Network   | Up to 2 routers  | 4 vCPU   | 8 GB   | 80 GB   |
| Regional ISP    | Up to 10 routers | 8 vCPU   | 16 GB  | 200 GB  |
| Large ISP       | Up to 25 routers | 16+ vCPU | 32+ GB | 500 GB+ |

### Storage Considerations

Storage consumption depends on:

* Number of monitored routers
* Route churn and update frequency
* Data retention settings
* Historical data requirements
* Enabled OpenBMP features (GeoIP, RPKI, IRR)

For environments with limited storage, consider reducing retention periods to automatically remove older data and minimize disk usage.


---

# ✅ Verification Checklist

After deployment verify:

* [ ] Docker containers are running
* [ ] Collector listening on TCP 5000
* [ ] Router BMP session established
* [ ] Kafka receiving messages
* [ ] PostgreSQL populated with routing data
* [ ] GeoIP import successful
* [ ] RPKI validation enabled
* [ ] Grafana accessible on port 3000
* [ ] Data retention policies configured

---

# License


This repository provides deployment and operational guidance for OpenBMP.

OpenBMP is an open-source project maintained by the OpenBMP community. Refer to the official OpenBMP repositories for software licensing information.

- https://github.com/OpenBMP
- https://www.openbmp.org
