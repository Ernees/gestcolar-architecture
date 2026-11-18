# Engineering decisions

Short records of the main trade-offs. Each one states the cost, not only the benefit.

## 1. MongoDB first, not PostgreSQL from day one
**Decision:** start on MongoDB. **Why:** flexible document shape while the academic data model was still changing.
**Cost:** tenant isolation depends on every query remembering its filter; the database cannot enforce it. That cost surfaced later (see ARCHITECTURE §2).

## 2. Railway, not AWS/Kubernetes
**Decision:** a managed PaaS. **Why:** for ~300 users, a self-managed cluster adds operational load with no business justification.
**Cost:** less control, and the CI pipeline does not build containers today.

## 3. Strangler-fig migration, not a rewrite
**Decision:** port one domain at a time, validate against real data, keep NestJS in production meanwhile.
**Why:** hundreds of real users depend on the current system, which already had three isolation bugs in its history. A big-bang rewrite multiplies that risk for an uncertain gain.

## 4. Report card as the first migrated domain
**Decision:** pilot with the report card. **Why:** read-only, so its blast radius is small; it exercises RLS, role authorization and data parity together.

## 5. Row-Level Security instead of "more careful code"
**Decision:** move isolation into PostgreSQL policies. **Why:** a missing `WHERE school_id = ...` stops being a data leak and becomes an empty result.
**Cost:** RLS alone is not enough; the session variable must be set per transaction, and roles inside a school still need application-level rules.

## 6. Flat roles with deny-by-default
**Decision:** five fixed roles with an explicit allow-list per endpoint. **Why:** simple to reason about and test.
**Cost:** no role hierarchy; adding finer permissions later will need a model change.
