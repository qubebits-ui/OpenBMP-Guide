# Data Retention & Database Cleanup

This document describes how to manage OpenBMP database growth, configure data retention policies, and manually reclaim disk space when required.

---

## Why Data Retention Matters

OpenBMP stores BGP routing information, updates, statistics, and peer events in PostgreSQL/TimescaleDB. In environments receiving full Internet routing tables, database growth can be significant depending on:

* Number of monitored routers
* Route update frequency
* BMP monitoring mode (Post-Policy vs Pre-Policy)
* Historical data retention requirements
* Enabled enrichment features (GeoIP, RPKI, IRR)

Implementing appropriate retention policies helps prevent excessive disk usage while maintaining sufficient historical visibility.

---

## Recommended Retention Settings

Configure the following PostgreSQL retention settings in your `docker-compose.yml` file.

Example values for deployments with 80–100 GB SSD storage:

```yaml
environment:
  - POSTGRES_DROP_peer_event_log='1 month'
  - POSTGRES_DROP_stat_reports='14 days'
  - POSTGRES_DROP_ip_rib_log='14 days'
  - POSTGRES_DROP_stats_chg_byprefix='14 days'
```

Apply the changes:

```bash
docker compose up -d
```

---

## Retention Variables

| Variable                         | Description                      |
| -------------------------------- | -------------------------------- |
| POSTGRES_DROP_peer_event_log     | BGP peer up/down events          |
| POSTGRES_DROP_stat_reports       | Router statistics reports        |
| POSTGRES_DROP_ip_rib_log         | Historical routing table changes |
| POSTGRES_DROP_stats_chg_byprefix | Prefix change statistics         |

Lower retention periods reduce storage consumption but preserve less historical data.


---



# Manual Database Cleanup

If disk usage becomes high and space needs to be reclaimed immediately, old TimescaleDB chunks can be removed manually.

### Access PostgreSQL

```bash
docker exec -it obmp-psql psql -U openbmp -d openbmp
```

Expected prompt:

```sql
openbmp=#
```

---

### Remove Historical Data

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

---

### Optimize Database

After removing old data, run:

```sql
VACUUM ANALYZE;
```

Exit PostgreSQL:

```sql
\q
```

---

### Verify Available Disk Space

Check disk usage on the host:

```bash
df -h
```

Monitor the PostgreSQL data directory if required:

```bash
du -sh /opt/openbmp/postgres/data
```

---

## Storage Planning Notes

The sizing recommendations in this repository assume:

* BMP Post-Policy monitoring only
* Full Internet routing tables

  * IPv4: ~1.16 million routes
  * IPv6: ~260,000–270,000 routes

Enabling BMP Pre-Policy monitoring can substantially increase:

* Kafka message volume
* PostgreSQL storage consumption
* CPU utilization
* Memory usage

Adjust retention settings accordingly when deploying Pre-Policy monitoring.
