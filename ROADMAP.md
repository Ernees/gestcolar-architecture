# Known limitations and roadmap

Written deliberately: knowing where a system is weak is part of owning it.

## Current system (NestJS + MongoDB)
- Tenant isolation is enforced by convention and tests, not structurally → addressed by the PostgreSQL + RLS migration.
- Tokens last one day; an endpoint re-issues a still-valid token, but a proper refresh-token flow is planned.
- Integration tests cover the grades and attendance services; other services and HTTP-controller-level integration tests are still missing.
- CI enforces a coverage threshold, but it is deliberately modest today and should be raised.
- CI does not build container images (Dockerfiles exist but are not part of the pipeline).

## Go + PostgreSQL service
- RLS context is set per transaction only in the report-card domain; the other ported domains still rely on query filters.
- RLS policies are not versioned in the repository's migrations yet.
- The family role is blocked for the report card until the family ↔ student check is connected to that authorization.
- Test coverage is uneven across domains (several still have none).
- No shadow-run against live traffic yet; the frontend switch is untested.

## Next
1. Version the RLS policies as migrations and extend the per-transaction tenant context to every domain.
2. Connect the family ↔ student check and unblock the family role for report cards.
3. Raise test coverage (Go service and the backend coverage threshold), domain by domain.
4. Add a shadow-run before any cutover.
