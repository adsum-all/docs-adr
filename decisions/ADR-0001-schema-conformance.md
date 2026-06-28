# ADR-0001 - Schema conformance with the DAT data model

- Status: Accepted
- Date: 2026-06-28
- Sprint: 0 (foundations)
- Deciders: ADSUM engineering

## Context

The DAT (`05-technique/ADSUM DAT.dc.html`) is the absolute source of truth for the
data model. While implementing the `deployment/database` migrations exactly per
the DAT, two internal inconsistencies in the DAT were found. The Constitution
(I9, no scope drift) and the project rule require flagging any DAT contradiction
through an ADR rather than guessing.

### Point 1: table count (17 stated, 18 defined)

The DAT prose states "the model has 17 tables", but its own domain enumeration and
its DDL define 18 tables:

- identity and membership (5): membre, commission, appartenance, engagement, document
- activity and presence (5): evenement, presence, jeton_qr, terminal, comptage_volet_b
- account and traceability (4): utilisateur, session, audit, notification
- settings and campaigns (4): parametre, import_lot, recensement, recensement_reponse

5 + 5 + 4 + 4 = 18. Every one of the 18 has an explicit `CREATE TABLE` in the DAT.
Omitting any table would break referential integrity (for example dropping
`appartenance` would remove the only non orphan link between membre and commission,
which the DAT itself calls out as the anti orphan junction).

### Point 2: audit primary key under range partitioning

The DAT declares the audit table as `id bigserial PRIMARY KEY` together with
`PARTITION BY RANGE (horodatage)`. PostgreSQL requires the partition key to be part
of any primary key or unique constraint on a partitioned table, so
`PRIMARY KEY (id)` alone is rejected at creation time.

## Decision

1. Implement all 18 tables defined by the DAT DDL. The "17" figure is treated as a
   counting slip in the prose, not an instruction to drop a defined table. No table
   is invented and none is omitted.
2. Implement the audit primary key as `PRIMARY KEY (id, horodatage)` so the
   append only, monthly range partitioned design of the DAT is valid in PostgreSQL.
   The `id` stays a `bigserial` surrogate; `horodatage` is the partition key.

## Consequences

- The migrations match the DAT DDL exactly, table by table, column by column,
  with the two corrections above documented here.
- Verified on the dedicated PostgreSQL (region Paris): 18 domain tables, 13 audit
  partitions (12 months of 2026 plus a default), the UNIQUE(membre_id, evenement_id)
  anti duplicate constraint, 22 foreign keys, the trigram search index and baseline
  row level security on all 18 tables.
- Follow up: the DAT prose can be corrected to read 18 tables in a future revision.
  Until then this ADR is the reference. Per role RLS policies are added in Sprint 1.
