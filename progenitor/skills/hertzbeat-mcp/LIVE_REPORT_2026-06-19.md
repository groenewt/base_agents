# HertzBeat live services & metrics report — 2026-06-19

Source: the `secure-api` MCP (HertzBeat at `http://silmaril.silmaril.internal:1157/api/mcp`) **only**.
Every host/port is cross-referenced to `scripts/env/graph_stack.env` (the canonical graph-stack env file).

## Top line
- **Warehouse (metrics store):** ONLINE ✓ (`get_warehouse_status`).
- **Monitors:** 11 configured, **11 Online**, 0 offline/unreachable/paused.
- **Alerting:** **0 alert rules, 0 alerts firing** — no thresholds configured (gap).
- **Metric flow gap:** `hbase_master` is Online but emits no metric data (see below).

## Monitor inventory ↔ `graph_stack.env`

| Monitor name | Type (`app`) | Instance | `graph_stack.env` var | Status |
|---|---|---|---|---|
| Agile_Fox_63Ps | `zookeeper` | localhost:2181 | `HBASE_ZK_QUORUM="localhost:2181"` (Atlas/HBase ZK — the *read-only* instance) | Online |
| Bright_Dolphin_99Pe | `kvrocks` | localhost:6666 | `KVROCKS_ADDR="127.0.0.1:6666"` | Online |
| Dreamy_Lynx_35TF | `hdfs_namenode` | localhost:9870 | `HDFS_NAMENODE_HTTP="127.0.0.1:9870"` | Online |
| Radiant_Elk_38tR | `hdfs_namenode` | localhost:9868 | `HDFS_SECONDARYNN_HTTP="0.0.0.0:9868"` — note: Secondary NN port watched as `hdfs_namenode` | Online |
| Humble_Gazelle_84dx | `hdfs_datanode` | localhost:9864 | `HDFS_DATANODE_HTTP="0.0.0.0:9864"` | Online |
| Loyal_Ocelot_89Ss | `hbase_regionserver` | localhost:61530 | `HBASE_REGIONSERVER_INFO="localhost:61530"` (non-default port) | Online |
| Proud_Zebra_84qf | `hbase_master` | localhost:61510 | `HBASE_MASTER_INFO="localhost:61510"` (non-default port) | Online* |
| Zesty_Lynx_49sJ | `yarn` | localhost:8088 | `YARN_RM_WEBAPP="0.0.0.0:8088"` | Online |
| Jolly_Shark_72MQ | `debian` | 192.168.122.1:22 | — (host VM; not in env file) | Online |
| Lively_Elk_43wU | `hive` | localhost:10002 | — (HiveServer2 web UI :10002; not in env file) | Online |
| Yummy_Giraffe_43bq | `postgresql` | silmaril.silmaril.internal:5432 | — (Postgres :5432; not in env file — likely HertzBeat's own backing store) | Online |

\* Online by liveness, but **no metric data flowing** — see Findings.

## Live metric samples (real values, `query_realtime_metrics`)

**ZooKeeper** — `Agile_Fox_63Ps` (id 652598516470528), localhost:2181, metric `stats`
- version 3.8.4, state **standalone**
- alive connections **6**, avg latency **0.17 ms** (max latency 5223 ms — a past spike), outstanding requests 0
- znodes **269**, ephemerals 16, watches 70
- approximate data size ~**721 MB** (738,161 kb); open FDs 1018 / 524,288 max
- Healthy.

**YARN ResourceManager** — `Zesty_Lynx_49sJ` (id 652599494300416), localhost:8088
- `ClusterMetrics`: **1 active NodeManager**, 0 unhealthy / lost / decommissioned
- `QueueMetrics` (root, root.default): **8 vCores** & **8192 MB available**, 0 allocated/pending/reserved; apps running/pending/failed = **0**
- Idle but healthy.

**KVRocks** — `Bright_Dolphin_99Pe` (id 652600200925952), localhost:6666
- `memory`: RSS **196.47 MB** (startup 75.3 MB)
- `stats`: 7 connections received, **2,849 commands processed**, instantaneous ops/sec **0** (idle), net in 65,245 B / out 1,123,842 B
- Healthy, lightly used.

## Findings / gaps (from MCP data only)

1. **`hbase_master` metrics not collected.** Monitor `Proud_Zebra_84qf` is Online, but `query_realtime_metrics(metrics="Server")` → *"No field parameter data available"*, and `metrics="basic"` → server error *`Cannot invoke "String.trim()" because "in" is null`*. Liveness is green; JMX metric pull is broken. Worth investigating (likely the master JMX/info endpoint on :61510 not returning the expected payload).

2. **No alerting at all.** `list_alert_rules` = 0 and `get_alerts_summary` = 0 across Critical/Emergency/Warning. Nothing will page on a service dropping. Candidate rules: YARN `NumUnhealthyNMs > 0` / `NumActiveNMs < 1`, HBase `numDeadRegionServers > 0`, ZooKeeper `zk_server_state != leader/standalone`.

3. **Writable Kafka stack is unmonitored.** Monitored ZK is `:2181` (`HBASE_ZK_QUORUM`, the read-only Atlas instance). The **writable** standalone ZK `ZOOKEEPER_ADDR="127.0.0.1:12181"` that backs the Kafka broker — and the broker itself `KAFKA_ADDR="127.0.0.1:19092"` — have **no monitors**.

4. **Other `graph_stack.env` services with no monitor:** Atlas REST `ATLAS_REST_URL` (:21000), Solr `SOLR_URL` (:9838), Fuseki `FUSEKI_URL` (:3030), HBase REST `HBASE_REST_ADDR` (:8080), HBase Thrift `HBASE_THRIFT_ADDR` (:9090), Phoenix Query Server `PHOENIX_QUERYSERVER_ADDR` (:8765, noted "not currently listening" in env), HDFS NameNode RPC `HDFS_NAMENODE_RPC` (:9000).

5. **Three monitors are not graph-stack services:** `debian` host (192.168.122.1:22), `hive` (:10002), `postgresql` (:5432) — none appear in `graph_stack.env`.

## MCP usage quirks discovered (capture for the skill)
- `query_monitors` requires **`sort` + `order` + `status` passed together** — any subset returns `Invalid value 'null' for orders` or `Property must not be null or empty`.
- `get_warehouse_status`, `get_alerts_summary`, `list_alert_rules`, `list_monitor_types`, `get_apps_metrics_hierarchy` take no/simple args and work cleanly.
- `query_realtime_metrics` needs `monitorId` (from `query_monitors`) + a `metrics` name (the `value` field, type=`metric`, from `get_apps_metrics_hierarchy`).
