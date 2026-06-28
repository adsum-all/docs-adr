# ADR-0002 - Row level security enforcement model

- Status: Accepted
- Date: 2026-06-28
- Sprint: 1 (database part)
- Deciders: ADSUM engineering

## Context

The DAT states that authentication, RBAC and the QR signature are enforced by the
backend (Python and FastAPI), not by a managed service, so that ADSUM stays master
of its logic and is not captive to a provider. At the same time, the Constitution
and the security posture require defense in depth at the database level (per role
RLS) on every table.

The database is hosted on Supabase (PostgreSQL 15). The backend connects with a
privileged role that owns the tables. A primary key based role check in the
database must not block the backend, yet it must enforce the access matrix of the
functional spec (cahier section 5.2) for any non owner access.

## Decision

1. Row level security is enabled on all 18 tables (baseline deny by default,
   migration 0004). Per role policies are added in migration 0005.
2. The backend sets a session variable ``adsum.role`` to the caller role
   (utilisateur.role) for the duration of each request transaction, with
   ``SET LOCAL "adsum.role" = '<role>'`` or ``set_config('adsum.role', '<role>', true)``.
3. Policies read it through a helper, ``adsum_current_role()``, and apply the
   access matrix: who can SELECT and who can INSERT, UPDATE or DELETE per table.
   The ``audit`` table has no write policy (append only); inserts come from the
   backend only.
4. RLS is not forced on the table owner, so the backend connection role and the
   Supabase service role bypass RLS. RLS is therefore defense in depth, fully
   consistent with the DAT decision that the backend enforces RBAC. It never
   blocks legitimate backend access and adds a second barrier for any other path.

## Consequences

- Policies follow the cahier section 5.2 matrix. Verified on the dedicated
  PostgreSQL (Paris) per role: super_admin writes everywhere; admin writes member
  data but not user accounts; gestionnaire and controleur write attendance but not
  members; direction is read only. 35 policies across 18 tables, the function reads
  the session variable correctly.
- The backend must set ``adsum.role`` at the start of every request transaction.
  This is implemented in the FastAPI data layer (services/adsum-api) as part of
  Sprint 1.
- If a stricter posture is later required (RLS enforced even for the owner),
  ``FORCE ROW LEVEL SECURITY`` can be enabled once the backend reliably sets the
  role on every connection. That change would get its own ADR.
