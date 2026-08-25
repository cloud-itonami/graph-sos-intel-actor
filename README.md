# graph-sos-intel-actor

Graph System-of-Systems intelligence actor. It continuously inventories and
reasons over the RisingWave graph substrate — `vertex_*` tables, `edge_*`
tables, `mv_*` materialized/read-model relations, `idx_*` indexes — and writes
compact snapshots and findings into graph tables, while leaving heavy DDL
execution to the existing RisingWave DDL governance path
(`RULE-GRAPH-SOS-NO-HEAVY-DDL` in the manifest).

## Identity

- DID: `did:web:graph-sos-intel.etzhayyim.com`
- DID document: [`.well-known/did.json`](.well-known/did.json)

## What is in this repo

| path | what it is |
|---|---|
| [`actor-manifest.jsonld`](actor-manifest.jsonld) | Declarative actor manifest: capabilities, cron + xrpc pipelines, governance rules |
| [`actor-manifest.test.ts`](actor-manifest.test.ts) | vitest invariants over the manifest (canonical identity, MCP primitives only, no heavy DDL in any pipeline SQL) |
| [`src/graph_sos_intel/murakumo.cljc`](src/graph_sos_intel/murakumo.cljc) | Pure actor boundary: `cell-plan` checks the 7 required gates and returns `:blocked` (no effects) or `:ready` with `:mst/put-record` effects |
| [`storage-profile.edn`](storage-profile.edn) | Repository storage profile declaration (`:kotoba/local-agent-kagi-chunks-v1`) |
| [`docs/operator-quickstart.md`](docs/operator-quickstart.md) | Runnable operator quickstart — every step in it has been executed as written |

## Gate model (deny by default)

Every cell in `cell-specs` requires all seven common gates
(`:council-charter-attestation`, `:no-platform-held-key-baseline`,
`:no-probing-baseline`, `:murakumo-only-inference-baseline`,
`:did-primary-baseline`, `:append-only-gate-baseline`,
`:kotoba-only-substrate-baseline`). A plan with any missing attestation is
`:blocked` and carries **zero** effects; only a fully attested plan is
`:ready` and yields `:mst/put-record` effects. See the quickstart for a
demonstration of both outcomes.

## Provenance

- Migrated from `etzhayyimcojp/20-actors` on 2026-05-21 (see `NOTICE`).
- The RisingWave schema migration for this actor
  (`30-graph/graph-schema/migrations/20260507170000_graph_sos_intel.ts`)
  lives in the legacy graph-schema tree, **not** in this repo. This repo holds
  the actor's identity, manifest, gate model, and manifest invariants.

## License

Apache License 2.0 with the etzhayyim Charter Compliance Rider v3.1 — see
`NOTICE`.
