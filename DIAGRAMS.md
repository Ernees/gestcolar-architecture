# Diagrams

All diagrams are based on the production code (NestJS backend, Angular frontend, CI workflow) and on the Go + PostgreSQL service. Simplified on purpose.

## 1. Request path (NestJS)

```mermaid
flowchart TD
  A["Client request"] --> B["helmet + CORS<br/>origins = active schools' domains, reloaded every 5 min"]
  B --> C["Throttler guard<br/>30 requests / minute / IP"]
  C --> D["AuthGuard<br/>verifies the JWT"]
  D --> E["RolesGuard<br/>deny-by-default: route must list the role"]
  E --> F["@SchoolId() decorator<br/>reads tenant id from the token"]
  F --> G["Controller to Service"]
  G --> H[("MongoDB<br/>query always filtered by schoolId")]
  G -.->|"mutations on audited controllers"| L["Audit interceptor<br/>field-level audit log"]
  H --> I["Response interceptor<br/>wraps the payload in data"]
```

## 2. Data model (MongoDB, logical view)

Every collection carries `schoolId`: that is the tenant boundary.

```mermaid
erDiagram
  USUARIO ||--o{ ASIGNACION : "profesorId"
  USUARIO }o--o{ ALUMNO : "hijos (family role)"
  ALUMNO ||--o{ NOTA : "alumnoId"
  ALUMNO ||--o{ ASISTENCIA : "alumnoId"
  USUARIO ||--o{ NOTA : "profesorId"
  USUARIO {
    string schoolId
    string rol
  }
  ALUMNO {
    string schoolId
    string dni
    string legajo
    boolean activo
  }
  ASIGNACION {
    string schoolId
    number cicloLectivo
    string gradoNombre
    string division
    string materiaNombre
    boolean esTitular
  }
  NOTA {
    string schoolId
    number cicloLectivo
    string periodoNombre
    string materiaNombre
    string valorGeneral
  }
  ASISTENCIA {
    string schoolId
    number cicloLectivo
    string periodoNombre
  }
  CRITERIO {
    string schoolId
    number cicloLectivo
    string grado
    string materia
  }
  CONFIGURACION {
    string schoolId
    number cicloLectivoActual
  }
  AUDIT_LOG {
    string schoolId
    string userId
    string action
    string targetId
  }
```

Grades have a unique index on (schoolId, alumnoId, materiaNombre, periodoNombre, cicloLectivo), so the same student/subject/period in two schools never collides.

## 3. Tenant isolation: today vs. target

```mermaid
flowchart LR
  subgraph TODAY["Today: NestJS + MongoDB"]
    T1["JWT carries schoolId"] --> T2["Each service adds the schoolId filter<br/>(by convention)"] --> T3["Integration tests catch<br/>a missing filter"]
  end
  subgraph TARGET["Target: Go + PostgreSQL"]
    G1["JWT carries schoolId"] --> G2["Request sets the tenant<br/>on the DB transaction"] --> G3["Row-Level Security rejects<br/>cross-tenant rows"]
  end
```

## 4. Report-card authorization (Go service)

```mermaid
flowchart TD
  R["Request: report card of student X"] --> Q{"Caller role"}
  Q -->|"ADMIN or DIRECTOR"| OK1["Allowed"]
  Q -->|"ALUMNO"| S{"Is X the caller?"}
  S -->|"yes"| OK2["Allowed"]
  S -->|"no"| NO1["Forbidden"]
  Q -->|"DOCENTE"| D{"Teacher assigned to X's grade/section this year?"}
  D -->|"yes"| OK3["Allowed"]
  D -->|"no"| NO2["Forbidden"]
  Q -->|"FAMILIA"| NO3["Blocked on purpose<br/>(check not wired in yet)"]
```

## 5. CI pipeline

```mermaid
flowchart LR
  subgraph BE["Backend job"]
    B1["npm ci"] --> B2["Lint"] --> B3["Type check"] --> B4["Unit tests"] --> B5["Integration tests<br/>in-memory MongoDB"] --> B6["Coverage threshold"]
  end
  subgraph FE["Frontend job"]
    F1["npm ci"] --> F2["Lint"] --> F3["Unit tests<br/>headless Chrome"] --> F4["Build"]
  end
```

## 6. Migration phases (strangler fig)

```mermaid
flowchart LR
  P0["Production<br/>NestJS + MongoDB"]:::done --> P1["Pilot: report card in Go + PostgreSQL<br/>RLS context per transaction"]:::done
  P1 --> P2["Version the RLS policies<br/>extend tenant context to every domain"]:::next
  P2 --> P3["Shadow-run against live traffic"]:::next
  P3 --> P4["Cut over endpoint by endpoint"]:::future
  classDef done fill:#d4edda,stroke:#2e7d32,color:#000
  classDef next fill:#fff3cd,stroke:#b8860b,color:#000
  classDef future fill:#e2e3e5,stroke:#666,color:#000
```

Green: done. Yellow: next. Grey: later. The Go service does not serve production traffic yet.
