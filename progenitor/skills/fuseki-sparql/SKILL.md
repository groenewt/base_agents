---
name: fuseki-sparql
description: Query/update the live Jena Fuseki SPARQL store and run the Common Core Fuseki gate via the built-out Telephone.Service.Fuseki arrows (the `fuseki_sparql` bin). Use when work needs SPARQL SELECT/CONSTRUCT/ASK/UPDATE, Graph-Store named-graph CRUD (load TTL, fetch, count, delete), an ontology/SHACL readback over RDF, or to enforce SPARQL integration (FORGE fp-triad Scala ASG/SPARQL, LinkML ttl→load→SPARQL readback).
allowed-tools: Bash
---

# fuseki-sparql — drive the live Fuseki/Jena SPARQL store through the built-out arrows

SPARQL is **already built out** in Elixir: `Telephone.Service.Fuseki` exposes constants-resolved,
wire-instrumented effect arrows over a raw-TCP HTTP/1.1 transport (`.Http`) —
`construct` (SELECT/CONSTRUCT), `ask_graph`, `assert` (UPDATE), and the Graph Store Protocol set
`put/post/get/head/delete/count_graph`. Each op fires `effect_attempt → effect_ok|effect_error` on
the telephone wire (that observability — wired, retried, classified — *is* the "stricter SPARQL
integration": SPARQL stops being ad-hoc curl).

The CLI surface is the **`fuseki_sparql`** bin (PATH-installed via `~/.local/share/telephone` →
`bin/fuseki_sparql`), a thin marshaller that delegates to `priv/fuseki/sparql_cli.exs` via
`mix run --no-start` (daemon-safe). "Use the API, don't reimplement it" — no shell-side SPARQL/HTTP.

## 1. Resolution is CONSTANTS-authoritative (read before assuming an endpoint)

Host / port / dataset / path-suffixes / content-types resolve from
`config/constants/graph_stack.exs` (`graph_stack.fuseki`) via `ResearchRender.Config.Constants`.

- The bin sources `scripts/env/graph_stack.env` (`set -a`) so operator env is in scope for config
  resolution, then runs `mix run --no-start` (config loads, OTP app does NOT boot → no daemon/socket
  clobber). The **authoritative** values are the constants; override host/port/dataset with
  `SILMARIL_FUSEKI_{HOST,PORT,DATASET}` (read by `config/constants/base.exs` at config-load time).
- `FUSEKI_URL` in `graph_stack.env` is the **env mirror** (equals the constants host:port). The
  Elixir arrows do **not** read it — it's only for the raw-curl probe in §5. Keep the two homes in
  sync; constants win for SPARQL.

Live topology (measured 2026-06-19, `$/datasets`): one dataset **`/jena`**, services
`query` (`/query`,`/sparql`), `update` (`/update`), `gsp-rw` (`/data`), `gsp-r`, `upload`.

## 2. Ops (Codex bin surface)

```
fuseki_sparql <subcmd> [payload] [--graph IRI] [--file PATH] [--accept MT]
```
Payload (query/update/turtle) = a positional arg | `--file PATH` | piped stdin. Exit 0 on `{:ok}`;
nonzero on `{:error}`/usage (FAIL-CLOSED — never a vacuous 0). stdout = the result body.

| Subcmd | Arrow | Notes |
|---|---|---|
| `ping` | `.Http` admin GET `/$/ping` | server liveness |
| `select` / `query <sparql>` | `Fuseki.construct` | SPARQL SELECT → sparql-results+json |
| `construct <sparql>` | `Fuseki.construct` | → turtle |
| `ask <sparql>` | `Fuseki.ask_graph` | → `true`/`false` |
| `count --graph IRI` | `Fuseki.count_graph` | integer triple count |
| `update <sparql>` | `Fuseki.assert` | SPARQL UPDATE |
| `put-graph  --graph IRI --file ttl` | `Fuseki.put_graph` | GSP replace named graph |
| `post-graph --graph IRI --file ttl` | `Fuseki.post_graph` | GSP append |
| `get-graph  --graph IRI` | `Fuseki.get_graph` | → turtle (chunked-dechunked, byte-complete) |
| `head-graph --graph IRI` | `Fuseki.head_graph` | GSP probe |
| `delete-graph --graph IRI` | `Fuseki.delete_graph` | GSP delete |

Examples:
```bash
fuseki_sparql ping
fuseki_sparql ask  'ASK {}'
fuseki_sparql select 'SELECT * WHERE {?s ?p ?o} LIMIT 5'
echo 'SELECT (1 AS ?ok) WHERE {}' | fuseki_sparql select
fuseki_sparql count --graph urn:silmaril:graph:forge
fuseki_sparql put-graph --graph urn:silmaril:graph:forge --file forge.ttl
fuseki_sparql construct --file query.rq
```

## 3. The enforcement gate (separate mix task — NOT the bin)

```bash
mix research.corpus.common_core.fuseki_gate [--external-base-repo R]
```
Loads the Common Core graphs over GSP and validates triple counts + ASK/absence queries against the
manifest. **This task runs `app.start`** (it is NOT `--no-start` / not daemon-safe) — run it
deliberately, not in a hot loop. This is the canonical "enforce stricter SPARQL integration" check.

## 4. When to use (the integration hooks)

- **FORGE fp-triad** Scala ASG/SPARQL seed — run the prove-one SPARQL query over the ontology.
- **FORGE linkml** projection — after `put-graph`-loading the rendered `forge.shacl.ttl` /
  `forge.linkml.ttl`, `count` + `ask`/`select` to validate the graph loaded (readback).
- **SHACL / ontology readback** — `construct`/`select` to diff store-vs-canonical relation body.
- **Gate** — the mix task above to enforce the Common Core triple/ASK contract.

## 5. No-mix raw probe (when the Elixir build is down)

The bin needs a compiling project. If the build is broken, probe the live server directly via the
env mirror:
```bash
. scripts/env/graph_stack.env
curl -fsS -m 8 "$FUSEKI_URL/\$/ping"                                   # liveness
curl -fsS -m 8 "$FUSEKI_URL/\$/datasets"                               # topology (dataset names)
curl -fsS -m 8 -X POST "$FUSEKI_URL/jena/query" \
  -H 'Content-Type: application/sparql-query' -H 'Accept: application/sparql-results+json' \
  --data 'ASK{}'                                                        # SPARQL probe
```

## Discipline
- Endpoints/dataset come from constants (or `SILMARIL_FUSEKI_*`), never a hardcoded host/port/dataset.
- The bin is `--no-start`; the **gate** mix task is `app.start` (not daemon-safe).
- A 2xx is not success on its own — the arrows thread `{:ok,_}|{:error,_}`; the bin exits nonzero on `{:error,_}`.
- Keep `FUSEKI_URL` (env) and `graph_stack.fuseki` (constants) in sync; constants are authoritative for SPARQL.
