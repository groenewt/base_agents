---
name: hertzbeat-mcp
description: Monitor and query the graph-stack services (HBase, HDFS, YARN, ZooKeeper, KVRocks, Atlas, Solr, Kafka, Fuseki, Phoenix) through the HertzBeat `secure-api` MCP. Use when the user asks to check service health/uptime, read real-time or historical metrics, add a monitor, or inspect/define alerts for any service in scripts/env/graph_stack.env.
allowed-tools: mcp__secure-api, Bash
---

# hertzbeat-mcp — observe the graph stack via HertzBeat

The `secure-api` MCP (user scope, `~/.claude.json` → `http://silmaril.silmaril.internal:1157/api/mcp`)
wraps a HertzBeat server. Its `mcp__secure-api__*` tools list/query monitors, read metrics, and manage
alerts. **All hosts/ports come from `scripts/env/graph_stack.env`** — never assume defaults; these ports
are deliberately non-standard (HBase :615xx, Solr :9838, Kafka ZK :12181).

## 1. Precondition — is the MCP alive?

Call `mcp__secure-api__get_warehouse_status` → expect `ONLINE ✓`.
If it 401s / errors with `WWW-Authenticate: Digest realm=sureness_realm`, the Bearer JWT expired (and a
*refresh* token is rejected — you need an *access* token). See `~/.claude` memory **`hertzbeat-mcp`** for the
full gotcha + re-auth path.

## 2. The env file is the source of truth for hosts & ports

```bash
source "${AGENTS_CORE_BASE_REPO:-/home/tristan/Desktop/platform/core/agents}/scripts/env/graph_stack.env"
```

When you add a monitor or read a metric, use the env var — not a literal — so the skill speaks the repo's language.

## 3. graph-stack service ↔ HertzBeat monitor map

| Service | `graph_stack.env` var | Endpoint | HertzBeat `app` | Monitored? |
|---|---|---|---|---|
| HBase Master | `HBASE_MASTER_INFO` | localhost:61510 | `hbase_master` | ✓ |
| HBase RegionServer | `HBASE_REGIONSERVER_INFO` | localhost:61530 | `hbase_regionserver` | ✓ |
| HBase REST | `HBASE_REST_ADDR` | 127.0.0.1:8080 | `website` / `port` | — |
| HBase Thrift | `HBASE_THRIFT_ADDR` | 127.0.0.1:9090 | `port` | — |
| HDFS NameNode | `HDFS_NAMENODE_HTTP` | 127.0.0.1:9870 | `hdfs_namenode` | ✓ |
| HDFS Secondary NN | `HDFS_SECONDARYNN_HTTP` | 0.0.0.0:9868 | `hdfs_namenode` | ✓ |
| HDFS DataNode | `HDFS_DATANODE_HTTP` | 0.0.0.0:9864 | `hdfs_datanode` | ✓ |
| HDFS NameNode RPC | `HDFS_NAMENODE_RPC` | hdfs://localhost:9000 | `port` | — |
| YARN ResourceManager | `YARN_RM_WEBAPP` | 0.0.0.0:8088 | `yarn` | ✓ |
| ZooKeeper (Atlas/HBase, read-only) | `HBASE_ZK_QUORUM` | localhost:2181 | `zookeeper` | ✓ |
| ZooKeeper (Kafka, writable) | `ZOOKEEPER_ADDR` | 127.0.0.1:12181 | `zookeeper` | — |
| Kafka broker | `KAFKA_ADDR` | 127.0.0.1:19092 | `kafka` | — |
| KVRocks | `KVROCKS_ADDR` | 127.0.0.1:6666 | `kvrocks` | ✓ |
| Apache Atlas (REST/UI) | `ATLAS_REST_URL` (+`ATLAS_AUTH`) | http://127.0.0.1:21000/api/atlas | `website` / `api` | — |
| Solr | `SOLR_URL` | http://localhost:9838/solr | `website` / `api` | — |
| Fuseki (Jena SPARQL) | `FUSEKI_URL` | http://127.0.0.1:3030 | `website` / `api` | — |
| Phoenix Query Server | `PHOENIX_QUERYSERVER_ADDR` | 127.0.0.1:8765 | `port` | — (env notes: not listening) |

Live snapshot + findings: see `LIVE_REPORT_2026-06-19.md` in this dir (11 monitors, all Online; 0 alert rules).

## 4. Read-a-metric workflow (the canonical 4 steps)

1. `mcp__secure-api__list_monitor_types` → confirm the `app` name (138 supported).
2. `mcp__secure-api__query_monitors` → get the **monitorId**. **Quirk: pass `sort` + `order` + `status` together** or it errors:
   `{ sort: "name", order: "asc", status: "9" }` (status 1=online,2=offline,3=unreachable,0=paused,9=all). Filter with `app:` / `search:`.
3. `mcp__secure-api__get_apps_metrics_hierarchy` with `app` → get metric groups + field names. **Use the `value` field, never `label`.**
4. `mcp__secure-api__query_realtime_metrics` with `monitorId` + a `metrics` group name (e.g. `stats`, `ClusterMetrics`, `memory`).
   For trends: `mcp__secure-api__get_historical_metrics` with `app` + `instance` (`ip:port`) + `metrics` + `fieldParameter` + `history` (`1h`/`24h`/`7d`).

Worked example — YARN cluster health:
- `query_monitors {app:"yarn", sort:"name", order:"asc", status:"1"}` → id `…`
- `query_realtime_metrics {monitorId:…, metrics:"ClusterMetrics"}` → `NumActiveNMs`, `NumUnhealthyNMs`
- `query_realtime_metrics {monitorId:…, metrics:"QueueMetrics"}` → `AvailableMB`, `AppsRunning`

## 5. Add a monitor for an un-watched stack service

`mcp__secure-api__get_monitor_params {app}` for the param template, then `mcp__secure-api__add_monitor` with
the `app` from the map above and host/port taken from the matching env var. Example targets currently unmonitored:
`KAFKA_ADDR` (`kafka`), `ZOOKEEPER_ADDR` :12181 (`zookeeper`, the writable one), `ATLAS_REST_URL` (`website`).

## 6. Alerts

- `mcp__secure-api__list_alert_rules` / `query_alerts {status:"firing"}` / `get_alerts_summary` — inspect.
- `mcp__secure-api__create_alert_rule` — use field `value`s from `get_apps_metrics_hierarchy` in the expression
  (e.g. YARN `NumUnhealthyNMs > 0`, HBase `numDeadRegionServers > 0`). Then `bind_monitors_to_alert_rule` and
  `toggle_alert_rule` to enable. (As of 2026-06-19 there are **0** rules — nothing pages.)

## Discipline
- Ports/hosts always from `graph_stack.env`. Don't hardcode HBase :16000/RegionServer :16020/Solr :8983/ZK :2181-as-kafka — they're wrong here.
- A monitor being **Online ≠ metrics flowing**: e.g. `hbase_master` is up but its `Server`/`basic` metric pull was empty/errored on 2026-06-19. Check the actual metric values, not just status.
- `query_monitors` and `query_realtime_metrics` return human-formatted text, not JSON; read the fields, don't `jq`.
