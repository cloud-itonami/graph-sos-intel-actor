# Operator quickstart — graph-sos-intel-actor

Every step below was actually executed on 2026-08-25 (macOS, nbb, node/npx,
jq) from the repo root, and the expected output shown is the real output.
If a step does not reproduce, the repo has drifted — do not skip it silently.

## Prerequisites

- `nbb` (ClojureScript-on-Node script host — the workspace default)
- `node` / `npx` (vitest is fetched on demand by `npx --yes`)
- `jq`

No install step: this repo has no `package.json`; vitest runs standalone
against the single test file.

## 1. Inspect the manifest

```bash
jq -r '{id: .["@id"], runtime,
        capabilities: (.capabilities|length),
        pipelines: (.pipelines|length),
        rules: (.governance.rules|map(.id))}' actor-manifest.jsonld
```

Expected:

```json
{
  "id": "did:web:graph-sos-intel.etzhayyim.com",
  "runtime": "k8s-langserver",
  "capabilities": 4,
  "pipelines": 5,
  "rules": [
    "RULE-GRAPH-SOS-NO-HEAVY-DDL",
    "RULE-GRAPH-SOS-COMPACT-SNAPSHOT"
  ]
}
```

## 2. Run the manifest invariants (vitest)

```bash
npx --yes vitest run actor-manifest.test.ts
```

Expected: `Test Files  1 passed (1)` / `Tests  6 passed (6)`. The six
invariants pin the canonical identity, MCP-primitives-only capabilities, the
cron + xrpc runtime surfaces, catalog-inventory SQL, and the absence of heavy
DDL (`CREATE/DROP/ALTER` of tables, indexes, materialized views) in any
pipeline.

## 3. Exercise the actor boundary — deny by default

The pure boundary lives in `src/graph_sos_intel/murakumo.cljc`. With **no
attestations**, every cell plan is `:blocked` and carries zero effects:

```bash
nbb --classpath src -e '
(require (quote [graph_sos_intel.murakumo :as m]))
(let [blocked (m/cell-plan :health {:attestations {}})]
  (println "blocked status:" (:status blocked))
  (println "missing gates:" (count (:missing-gates blocked)))
  (println "effects:" (count (:effects blocked))))'
```

Expected:

```text
blocked status: :blocked
missing gates: 7
effects: 0
```

With all seven common gates attested, the same cell becomes `:ready` and
plans one `:mst/put-record` effect per collection:

```bash
nbb --classpath src -e '
(require (quote [graph_sos_intel.murakumo :as m]))
(let [atts (into {} (map (fn [g] [g true]) m/common-gates))
      ready (m/cell-plan :health {:attestations atts
                                  :computed-at "2026-08-25T00:00:00Z"
                                  :request-id "qs-demo-1"})]
  (println "ready status:" (:status ready))
  (println "effects:" (mapv :op (:effects ready)))
  (println "collection:" (:collection (first (:records ready))))
  (println "rkey:" (:rkey (first (:records ready)))))'
```

Expected:

```text
ready status: :ready
effects: [:mst/put-record]
collection: com.etzhayyim.graph-sos-intel.health
rkey: qs-demo-1
```

`(keys m/cell-specs)` lists the other cells (`:listrelations`,
`:listfindings`, `:shinka`, …) — all of them require the same seven gates.

## 4. What you can NOT do from this repo

- There is no live RisingWave connection here: the plans in step 3 are pure
  values; executing `:mst/put-record` effects is the runtime's job, not this
  repo's.
- Heavy DDL (creating/dropping tables, indexes, materialized views) is
  intentionally absent and rejected by the step-2 invariants; it belongs to
  the RisingWave DDL governance path, with the schema migration in the legacy
  graph-schema tree (`30-graph/graph-schema/migrations/20260507170000_graph_sos_intel.ts`).
