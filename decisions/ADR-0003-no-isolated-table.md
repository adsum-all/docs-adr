# ADR-0003 - No orphan or isolated table (audit related to utilisateur)

- Status: Accepted
- Date: 2026-06-28
- Sprint: 0 and 1 (schema integrity)
- Deciders: ADSUM engineering

## Context

Constitution I3 forbids any orphan or isolated table: every table must participate
in at least one coherent relation of the model. A schema audit on the real database
showed exactly one isolated table, `audit` (no incoming and no outgoing foreign key).

This is consistent with the DAT audit DDL, where `acteur_id uuid` is written without
a `REFERENCES` clause so the append-only audit trail survives. However, the DAT
relations section explicitly declares "utilisateur 1 to N audit" (the actor of each
sensitive action), so the relation is part of the model; the DDL simply omitted the
foreign key.

## Decision

Add the relation declared by the DAT: `audit.acteur_id` references `utilisateur(id)`
with `ON DELETE NO ACTION` (migration 0006).

- NO ACTION preserves the append-only nature of audit: staff accounts use logical
  deactivation (`utilisateur.actif`), never a hard delete, so the constraint never
  fires and never modifies an audit row.
- `acteur_id` stays nullable for system actions with no actor.
- The other 17 tables already participate in relations (verified).

## Consequences

- Zero orphan or isolated table. The 18 tables are all linked. Verified on the real
  database (Paris): the per table audit reports no isolated table after migration 0006.
- Because `audit` is range partitioned, the foreign key is propagated to every
  partition; this is expected PostgreSQL behavior and does not change the logical
  single relation.
- RGPD erasure of a staff actor is handled by anonymization rather than hard delete,
  consistent with NO ACTION. This is detailed in the hardening sprint (Sprint 10).
- An automated no-orphan check is part of the database test suite and is a candidate
  control for the schema audit job in `deployment/ci-templates` (PLAYBOOK section 10.3).
