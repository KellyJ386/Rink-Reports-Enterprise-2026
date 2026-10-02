# Rink Reports Enterprise Architecture Blueprint

**Status:** Proposed for review  
**Version:** 0.1  
**Date:** 2026-10-02  
**Scope:** Phase 0 only—no production feature implementation

## How to read this document

This blueprint is the architectural contract for a multi-facility platform, not a promise to build every feature at once. A decision record uses four fields: **Decision**, **Alternatives**, **Recommendation/reasoning**, and **Implications**. Items labeled **Approval required** must be decided before the affected implementation begins.

## 1. Executive summary

Rink Reports Enterprise will be a modular monolith built with Next.js, TypeScript, Supabase/PostgreSQL, Supabase Auth, Row Level Security (RLS), Tailwind, shadcn/ui, and a PWA client. It will have strict organization tenancy, optional regions, reusable authorization, forms, events/workflows, schedules, reporting, notifications, integration, audit, and offline-sync services. A modular monolith keeps transactions and operations simple while domain boundaries, an outbox, and versioned APIs preserve an extraction path for services that later need independent scale.

The organization is the hard tenant boundary. Users may have memberships and scoped role grants in multiple organizations and facilities, but every private business row is traceable to one organization. Authorization is enforced at the database and API layers; the interface only reflects—not defines—access. Operational writes produce immutable domain events through a transactional outbox, enabling workflows, alerts, reporting, webhooks, and integrations without coupling feature modules.

The Corporate Command Center is a read model, not a parallel source of truth. It combines current facility status with transparent, configurable indicators and drill-down links. Facility employees receive a deliberately narrow workspace derived from their assignments and grants.

### Architectural principles

1. One canonical identity for each business concept; link domains rather than duplicate them.
2. Organization isolation is mandatory and tested negatively at every boundary.
3. Scope and capability are separate: a grant answers both **where** and **what**.
4. Synchronous transactions protect invariants; asynchronous events perform side effects.
5. Submitted records retain the exact schema version under which they were captured.
6. APIs paginate, filter server-side, authorize each object, and expose stable public identifiers.
7. Derived dashboards and reports remain explainable and drillable to source records.
8. Offline operations are explicit, idempotent, observable, and conflict-aware.
9. Financial data remains a separate bounded context from operations.
10. New modules must reuse identity, tenancy, authorization, audit, event, notification, reporting, and integration contracts.

## 2. Product boundaries

### In scope through the enterprise platform phases

- Organization/region/facility hierarchy, areas, departments, workforce identity, scoped authorization, audit, bulk management, and onboarding.
- Shared forms/records, scheduling, tasks, workflows, notifications, search, equipment, policy, operational safety, reporting, Connect, APIs, webhooks, embeds, displays, and integrations.
- Customer/household/program foundations and later separately bounded reservation, registration, and commerce capabilities.
- Public, employee, facility, regional, corporate, and platform-administration experiences over the same canonical services.

### Explicitly out of scope for the first release

- Payroll, tax filing, benefits, full HRIS, general ledger, and certified accounting.
- Building automation control loops or safety-critical equipment control; the platform may ingest telemetry and raise alerts but must not directly replace certified controllers.
- Clinical records or medical diagnosis. Incident data may document reported facts with restricted access and retention.
- A generic low-code application builder. Configurability is constrained to supported schemas, rules, and actions.
- Full POS and league scoring until dedicated phases and controls are approved.

### Context boundaries

`Identity & Access`, `Organization Directory`, `Workforce`, `Scheduling`, `Operations`, `Forms & Records`, `Safety`, `Assets & Maintenance`, `Automation`, `Communications`, `Reporting`, `Connect`, `Customers & Programs`, and `Commerce` own their invariants. Cross-context references use stable IDs and events; shared tables are not casually mutated by unrelated modules.

## 3. Enterprise organization hierarchy

```text
Platform
└── Organization (hard tenant boundary)
    ├── Region (optional grouping; one level in v1)
    │   └── Facility
    └── Facility (region_id may be null)
        ├── Area (typed: ice sheet, room, locker room, mechanical, work area)
        ├── Department
        ├── Equipment / Resources
        └── Assignments (employees, schedules, templates, policies)
```

`areas` supplies common spatial behavior while `ice_sheets`, `rooms`, and `locker_rooms` are typed extension tables when type-specific constraints exist. A facility belongs to exactly one organization and at most one region. A region belongs to exactly one organization. Region reassignment is audited and does not rewrite historical facts.

Employees belong to an organization, have one optional home facility, and receive zero or more time-bounded facility/department assignments. An authentication user can link to an employee profile per organization; service accounts are distinct principals.

**Decision — hierarchy depth.** Alternatives: arbitrary recursive org units; fixed organization/region/facility; duplicated hierarchy columns. **Recommendation:** fixed organization → optional region → facility for v1, plus departments and areas as orthogonal scopes. It matches the stated operating model and makes RLS and aggregation predictable. **Implication:** deeper geographic hierarchies require a reviewed `organizational_units` migration rather than abusing departments. **Approval required.**

## 4. Multi-tenant architecture

### Tenant contract

- `organization_id uuid not null` is physically present on every organization-owned private table, including rows also scoped to a region or facility.
- Region- and facility-scoped rows carry both the narrower foreign key and organization ID, protected by composite foreign keys such as `(organization_id, facility_id) → facilities(organization_id, id)`.
- IDs are UUIDv7 (or database-generated UUIDs until UUIDv7 support is approved); public surfaces use separate revocable opaque slugs/tokens.
- Cross-tenant foreign keys are impossible. No client-supplied organization ID is trusted without membership evaluation.
- Platform-global tables are rare and administered only through isolated platform roles.
- Storage object paths begin with organization ID and storage policies repeat the same access checks.

### Request context

Supabase Auth establishes `auth.uid()`. Server code selects an active organization and optional facility from the user's valid memberships; it never converts a UI selection into authority. Database helper functions evaluate grants against row scope. Elevated service credentials are restricted to workers, never shipped to clients, and every elevated operation states and audits the target tenant.

### Deployment shape

Begin with one regional PostgreSQL cluster and application deployment, with connection pooling and queue workers. Tenant-aware interfaces keep open the future choices of read replicas, archival partitions, and placement of selected large tenants. Database-per-tenant was rejected initially because migrations, reporting, support, and operational overhead grow rapidly.

**Decision — tenancy.** Alternatives: database per organization, schema per organization, shared tables without RLS, shared tables with tenant columns and RLS. **Recommendation:** shared schema with mandatory tenant columns, composite integrity constraints, RLS, and tenant-aware services. **Reasoning:** strongest balance of isolation, cross-facility queries, migration consistency, and cost at the target scale. **Implications:** policy simplicity, query linting, negative isolation tests, and tightly controlled bypass roles are release requirements. **Approval required.**

## 5. Canonical database/domain model

### Classification rules

| Classification | Ownership rule | Examples |
|---|---|---|
| Platform-global | No customer data; platform-admin only | capability catalog, system event types, plan definitions |
| Organization-scoped | `organization_id` mandatory | employees, customers, roles, forms, workflows, equipment |
| Region-scoped | organization plus `region_id` | regional assignments, template targets, policy targets |
| Facility-scoped | organization plus `facility_id` | areas, tasks, records, schedule allocations, work orders |
| User-scoped | owner user plus organization where business-related | preferences, device registrations, saved personal views |
| Public | explicitly projected and allowlisted | published facility profiles, widget publications |

“Public” is a publication state/read model, never a blanket policy on private source tables.

### Core relationship map

```text
auth.users ── users ── organization_memberships ── organizations
                         │              ├── regions ── facilities
                         │              └── facilities ── areas ── typed area extensions
                         ├── employees ── employee_assignments ── departments
                         └── role_grants ── roles ── role_permissions ── capabilities

forms ── form_versions ── form_fields
  └── form_assignments ── facilities
records ── record_values / record_repeated_items ── approvals

schedule_events ── resource_allocations ── resources
       └── bookings / programs / teams / customers

domain_events ── outbox_messages ── workflow_runs ── action_attempts
                                 ├── tasks
                                 ├── notifications
                                 └── webhook_deliveries

equipment ── equipment_assignments ── facilities/areas
         ├── inspections (records)
         ├── work_orders
         └── service_events / downtime_intervals
```

### Data conventions

- UUID primary keys; `created_at`, `updated_at`, `created_by`, and archival metadata where appropriate.
- UTC `timestamptz` for instants; facility IANA timezone for display and local recurrence interpretation.
- State transitions use constrained enums or reference catalogs and service functions, not free text.
- Soft deletion only where recovery/history is needed; immutable facts use correction/supersession rather than mutation.
- Flexible definitions and provider payloads may use validated JSONB; identities, ownership, relationships, statuses, and searchable/reportable facts use columns/relations.
- Files live in object storage with metadata, checksum, classification, retention, and attachment relations.
- Monetary values use integer minor units plus ISO currency; commerce owns balances and payment transitions.

## 6. Core entities and ownership

| Domain | Canonical entities | Scope |
|---|---|---|
| Directory | organizations, regions, facilities, areas, ice_sheets, rooms, locker_rooms, departments | organization/region/facility |
| Identity | users, organization_memberships, employees, employee_assignments, service_accounts | organization/user |
| Authorization | capabilities, roles, role_permissions, role_grants | global catalog + organization grants |
| Assets | equipment, equipment_types, equipment_assignments, service_events, downtime_intervals, work_orders, QR aliases | organization/facility |
| Forms | forms, form_versions, form_fields, form_assignments, records, record_values, approvals | organization/facility |
| Scheduling | schedule_events, bookings, resources, allocations, operational_offsets, availability_blocks | organization/facility |
| Automation | domain_events, outbox_messages, workflows, workflow_versions, workflow_assignments, runs, action_attempts | organization |
| Work | tasks, task_assignments, alerts, incidents, accidents | organization/facility |
| Communication | notifications, deliveries, preferences, announcements | organization/user |
| Governance | documents, policies, policy_versions, policy_assignments, acknowledgements, audit_events | organization |
| Connect | API clients, keys, webhook endpoints/deliveries, integrations, widget publications, display channels | organization/public projection |
| Customer | customers, households, household_members, customer_organization_links, teams, programs, participants | organization |
| Commerce | catalog items, orders, invoices, payments, refunds, credits | organization; isolated context |
| Analytics | metric_definitions, metric_facts, snapshots, saved_reports, report_runs | organization |

Incidents and accidents share common record/attachment primitives but remain explicit safety aggregates because access, retention, and lifecycle differ. `resources` provides schedulability; it references canonical employees, areas, or equipment rather than replacing them.

## 7. Roles and permissions

Authorization evaluates: `principal + organization membership + capability/action + scoped grant + resource attributes + time`. Capabilities use stable names such as `safety.incident.read`, with actions `read`, `create`, `edit`, `approve`, `administer`, and `report` represented explicitly rather than inferred.

A role is an organization-defined bundle of permissions. A role grant has exactly one scope type (`organization`, `region`, `facility`, or `department`), scope ID, validity interval, and provenance. Organization scope includes its regions/facilities; region scope includes current member facilities; facility scope can include its departments/areas; department scope never expands to a facility. Explicit deny is deferred in v1 to avoid ambiguous precedence; sensitive capabilities instead require a dedicated positive grant.

Employees may hold multiple role grants only when capability combinations pass a separation-of-duties policy. Conflicting hierarchy labels are avoided by treating job title as workforce data and authority as grants. Temporary assignments do not automatically grant capabilities: a time-bounded facility assignment and suitable role grant are both required.

Platform Super Admin uses a separate, strongly authenticated support path with just-in-time elevation, reason, expiry, and audit. Organization owners cannot grant platform capabilities.

**Decision — authorization model.** Alternatives: fixed role column, pure RBAC, pure ABAC, scoped RBAC plus attributes. **Recommendation:** scoped RBAC with limited ABAC conditions (tenant, assignment validity, ownership, sensitivity). **Reasoning:** administrators can understand roles while scope and time constraints cover enterprise staffing. **Implications:** one central evaluator and database helper contract must be shared by UI, API, workers, and RLS. **Approval required.**

## 8. RLS approach

1. Enable and force RLS on every tenant/private table; revoke default public grants.
2. Use small `security definer` helper functions owned by a non-login role, fixed `search_path`, and explicit arguments: `is_org_member`, `has_capability`, `can_access_facility`, and `can_access_department`.
3. `SELECT` policies require tenant membership plus a scoped read/report grant. Insert policies use `WITH CHECK` for organization, scope, and create capability. Update requires both visibility and `WITH CHECK` of the resulting row. Deletes are normally service-controlled archival operations.
4. Sensitive safety, HR, and customer tables require dedicated capabilities rather than generic facility access.
5. Views use `security_invoker` unless a narrowly reviewed projection intentionally masks columns. Database functions validate grants internally.
6. Workers with a bypass role carry an explicit tenant/job context, use repository methods that require organization ID, and generate audit events.
7. Public access is through curated publication views/functions keyed by revocable widget tokens; anonymous users cannot query source tables.

CI seeds two organizations, multiple regions/facilities, a regional manager, facility manager, Ice Tech, public token, service account, and expired grant. Tests assert allowed and denied selects, writes, RPC calls, storage access, and pagination probes. Policy coverage is generated from the schema; any new tenant table without forced RLS fails CI.

## 9. Corporate Command Center

The command center reads tenant-scoped operational projections, not raw fan-out queries on every page load. Event consumers maintain per-facility current status and time-bucket metric facts. A short-latency reconciliation job corrects missed events from source tables.

### Read models

- `facility_status_current`: open/closed state, active critical/attention indicators, staffing and equipment exceptions, source timestamps.
- `facility_metric_facts`: metric, numerator, denominator, dimensions, window, and freshness.
- `organization_daily_snapshot`: facility/ice-sheet/employee counts and issue rollups.
- `attention_items`: source type/ID, severity, opened/due times, accountable scope, status, and drill-down route.

Health is rule-driven and explainable. Each indicator names its rule/version, source observations, threshold, severity, owner, and last evaluation. “Normal” means no active rule evaluated above normal and data freshness is acceptable; stale data is visibly “unknown,” never silently normal.

Filters always execute server-side and preserve authorization. URLs retain organization, region, facility, date, and issue filters so users can drill down and return without losing context. Counts link to filtered record lists, then canonical records. Snapshot timestamps and metric definitions are visible.

**Decision — dashboard computation.** Alternatives: live joins only, external warehouse first, PostgreSQL read models with background aggregation. **Recommendation:** PostgreSQL read models/materialized aggregates plus event-driven updates and reconciliation. **Reasoning:** target scale fits PostgreSQL while avoiding slow dashboard fan-out. **Implications:** freshness SLOs, replayable consumers, and source drill-down are required.

## 10. Forms and records engine

Forms are stable identities; immutable `form_versions` contain sections, typed fields, validations, conditions, calculations, and approval definitions. Published versions cannot be edited. Records reference exactly one version and store lifecycle (`draft`, `submitted`, `approved`, `rejected`, `superseded`) plus facility, subject, submitter, effective time, and idempotency key.

Field definitions may use validated JSONB for type-specific configuration, but field identity, type, order, required status, and reporting key are columns. Scalar record values use typed columns with a one-of constraint; repeated groups use item and value tables. This supports validation, indexing, reporting, and future schema evolution without one table per checklist. Signatures record signer, intent, timestamp, record digest, and evidence metadata; a signature image alone is not legal proof.

Assignments target organization, region, facility, facility group, department, role, or equipment type with effective dates. Resolution produces one effective version per context and explains precedence. A **Corporate Standard** locks required/core fields and permits only named extension points. A **Corporate Template** can be forked within approved sections while retaining ancestry. Adoption state records assigned, acknowledged, scheduled, active, and superseded versions per facility.

Conditional/calculated expressions use a restricted, versioned DSL—never arbitrary JavaScript or SQL. Server validation is authoritative and repeats client validation. Threshold evaluation emits domain events. Draft schema migrations are previewed against existing assignments before publication.

**Decision — record storage.** Alternatives: a table per form, one opaque JSONB response, EAV only, hybrid typed values plus versioned definitions. **Recommendation:** hybrid typed values and validated definition JSONB. **Reasoning:** balances configurability, constraints, and cross-form reporting. **Implications:** field types and DSL changes require compatibility/version policies. **Approval required.**

## 11. Workflow and event architecture

Every meaningful committed change can append a canonical `domain_event` and `outbox_message` in the same transaction. Events contain event ID, name/version, organization and facility context, aggregate type/ID/version, actor, occurrence time, correlation/causation IDs, classification, and a minimized payload. A worker publishes/processes the outbox with at-least-once delivery.

Workflow definitions are immutable versions assigned by scope. A run records the triggering event, matched conditions, actions, status, and definition version. Conditions use the restricted rule DSL. Actions are allowlisted handlers (task, notification, status request, inspection, webhook, integration) with schemas, timeouts, retry policies, and permissions. Each attempt has a deterministic idempotency key. Retries use exponential backoff; poison work enters a dead-letter queue with replay tools.

Events are facts in past tense (`schedule.event.created`). Commands request changes (`complete_task`). Consumers must tolerate duplicates and out-of-order events by aggregate version. Sensitive event payloads carry references or minimized fields, not full medical/customer records. Retention differs between audit, operational event, and integration-delivery logs.

Region-specific workflow changes create a new version and region assignment; other scopes retain their effective version. Simulation shows impacted facilities and example matches before activation.

**Decision — delivery semantics.** Alternatives: synchronous hooks only, exactly-once claims, transactional outbox with at-least-once processing. **Recommendation:** transactional outbox and idempotent consumers. **Reasoning:** it prevents lost side effects without pretending distributed exactly-once delivery. **Implications:** every action and webhook needs idempotency and replay behavior.

## 12. Schedule architecture

`schedule_events` represent the operational event; `bookings` represent customer/commercial intent and may link to an event. Resources are allocated through `resource_allocations` with exclusive/capacity rules. Canonical resource references cover ice sheets, rooms, locker rooms, equipment, employees, and generic capacity resources. PostgreSQL exclusion constraints prevent conflicting exclusive allocations within a facility.

Events store UTC instants, facility timezone, setup/cleanup buffers, source, external ID, lifecycle, recurrence lineage, and visibility. Recurrence rules are stored with bounded materialized occurrences; editing one occurrence does not rewrite unrelated history. Maintenance blocks and closures are schedule events with constrained types.

Operational templates attach relative offsets to event types: e.g., locker-room preparation at −60 minutes and ice make at event end. Publishing or changing a schedule event emits versioned events. Automation materializes linked tasks with deterministic keys, reschedules eligible pending tasks, and flags rather than overwrites work already started. External feeds use source mappings and conflict queues.

Availability is computed from operating hours, closures, allocations, buffers, capacity, and publication rules. Public availability is a sanitized projection and supports organization/region/facility search with cursor pagination.

## 13. Reporting architecture

The reporting engine has a governed semantic catalog: metric name, description, source, dimensions, numerator/denominator, grain, timezone, owner, sensitivity, and version. Operational reports query indexed canonical views and metric facts. Expensive or historical reports run asynchronously against snapshots/read replicas when introduced.

Saved reports store a validated query specification—not raw SQL—including fields, filters, grouping, sort, visualization, and scope. The server intersects requested scope with current grants on every run. Exports are background jobs written to short-lived signed storage URLs, row-limited, audited, and protected against CSV formula injection. Scheduled delivery rechecks the owner's/service account's authorization at execution time.

Metrics use explicit denominators and expose missing/stale data. Comparisons never collapse safety into an unexplained score. Drill-through carries metric version and filters to source rows. Revenue and labor metrics enter only through governed domain views; reporting does not own commerce or payroll facts.

**Decision — analytics platform.** Alternatives: warehouse immediately, direct OLTP queries, PostgreSQL semantic views/facts with an extraction seam. **Recommendation:** PostgreSQL first with background facts and a future CDC/export boundary. **Reasoning:** adequate for 30–50 facilities and faster to govern. **Implications:** load testing establishes the threshold for a warehouse/read replica. **Approval required.**

## 14. API architecture

- Versioned HTTPS JSON endpoints under `/api/v1`; OpenAPI is generated and contract-tested.
- Browser sessions use Supabase Auth; enterprise integrations use hashed, rotatable API credentials bound to a service account, organization, scopes, optional IP policy, expiry, and last-used metadata. OAuth 2.1 is introduced for third-party delegated access when justified.
- Authorization occurs after authentication and before object access. Object scope and field sensitivity are enforced server-side and by RLS.
- Cursor pagination is mandatory for collections; limits have safe maxima. Filtering and sorting are allowlisted.
- Mutations accept idempotency keys, return stable error envelopes/correlation IDs, and use optimistic concurrency (`version`/ETag) where editing races matter.
- Rate limits apply by token, organization, IP, and endpoint class. Responses include traceable request IDs, never secrets.
- Public API resources come only from publication projections and opaque public identifiers. Incidents, accidents, employee details, customers, and safety data are private by default.

API keys are shown once, stored only as hashes, prefix-identifiable, rotatable with overlap, revocable, and audited. Secrets use managed environment/secret storage rather than database plaintext.

## 15. Webhook architecture

Subscriptions choose allowlisted event types and optional permitted facility filters. Deliveries serialize a versioned, minimized envelope with unique event/delivery IDs. Each endpoint has independently rotatable secrets. Signatures use HMAC-SHA256 over timestamp plus raw body; consumers receive timestamp and signature headers and should reject stale requests.

The dispatcher retries transient failures with exponential backoff and jitter, honors bounded `Retry-After`, stops on terminal failures, and dead-letters after the configured window. Delivery logs retain request metadata, response code, duration, attempt count, and a redacted/truncated body—not secrets. Administrators can test, disable, rotate, inspect, and replay a delivery; replay preserves event identity but receives a new delivery ID.

Outbound SSRF controls require HTTPS, resolve and block private/link-local destinations, limit redirects and response sizes, and revalidate DNS. Organization concurrency quotas prevent a failing endpoint from starving others.

## 16. Rink Reports Connect

Connect is an administration surface over four services:

1. **Publish:** widgets, public API publications, display channels, and availability rules.
2. **Integrate:** provider connections, credentials, mappings, imports/exports, sync status, and conflict resolution.
3. **Automate:** webhook subscriptions and externally callable approved workflows.
4. **Observe:** sync runs, webhook deliveries, failures, rate use, freshness, and audit history.

Provider adapters implement a common contract for authorize, validate, pull cursor, normalize, map, push, reconcile, revoke, and health. Canonical source mappings retain provider ID, local ID, source of truth, revision token, and last sync. Each connection chooses direction and conflict policy per object type. Credentials are envelope-encrypted and inaccessible to browser clients.

## 17. Website embed architecture

`widget.js` is a small versioned loader that creates an isolated iframe hosted on the embed origin. Iframes provide style and script isolation, CSP control, accessibility, and safer upgrades than injecting a full application into customer DOM. `postMessage` permits only a documented, origin-checked resize/navigation protocol.

An embed uses an opaque, revocable publication token. The token identifies an allowlisted organization/facility set, widget types, public data fields, allowed origins, theme, locale, and expiry/rotation metadata; it is not a secret granting private access. The public API validates origin where available, rate-limits abuse, and queries only publication projections. Browser caching/CDN keys include publication/version without leaking internal IDs.

Organization widgets support facility finder, events, programs, and cross-location availability. Locker-room assignments require an explicit privacy review and publication policy; participant names are never included by default. Search is bounded by time and result limits.

**Decision — embed isolation.** Alternatives: raw DOM script components, web components, iframe loader. **Recommendation:** iframe loader with a tiny script. **Reasoning:** strongest tenant styling and security isolation across varied website builders. **Implications:** responsive sizing and cross-origin accessibility need dedicated testing.

## 18. White-label strategy

Brand configurations are versioned organization defaults with optional facility overrides: logos, constrained color tokens, approved font choices, favicon, portal label, email theme, and button tokens. Contrast validation and fallback themes prevent inaccessible combinations. No arbitrary CSS or JavaScript is accepted.

Custom domains use verified DNS ownership, automated TLS, unique domain constraints, and an explicit routing map. Host resolution selects a public portal only after verification; authentication callback and cookie domains remain narrowly scoped. Email branding separates display customization from authenticated sending domains (SPF, DKIM, DMARC status).

**Decision — customization.** Alternatives: arbitrary templates/CSS, tokenized themes, separate deployments. **Recommendation:** tokenized themes and controlled layout variants on shared deployments. **Reasoning:** safe upgrades and accessibility without permanent forks. **Implications:** unusual branding requires new governed tokens/components, not customer code.

## 19. Offline strategy

The PWA caches the application shell and explicitly selected reference data. IndexedDB stores encrypted-at-rest-where-supported operational drafts and an append-only sync queue containing operation ID, tenant/scope, aggregate, base version, payload, created time, and dependency IDs. Offline access requires a previously authenticated device, bounded session, local data classification policy, and remote revocation on reconnect.

Each mutation has a client-generated idempotency key. Sync authenticates again, reauthorizes against current grants, validates the current form/schema version, and returns per-operation acknowledgement. Safe append operations merge; versioned state changes use optimistic concurrency; protected approvals, assignments, schedule changes, and destructive edits require online service or explicit human conflict resolution. The client never silently overwrites.

UI shows offline, pending, syncing, failed, conflicted, and acknowledged states at both global and record levels. Users can inspect/retry/export a failed draft. Queue size, age, failure reason, app/schema version, and device are observable without logging sensitive field values. Attachments upload in resumable chunks after the parent is accepted.

**Decision — offline scope.** Alternatives: online-only, broad database replication, operation queue with curated reference cache. **Recommendation:** curated cache plus command queue. **Reasoning:** predictable authorization and conflicts with smaller privacy exposure. **Implications:** every offline-capable command needs an explicit merge and expiry policy. **Approval required.**

## 20. Integration architecture

Inbound adapters normalize external data into canonical commands; they never write domain tables directly. Raw payloads are encrypted, size/retention limited, and referenced for diagnostics. Mapping rules are versioned. Import runs have staged validation and counts for received, valid, imported, updated, duplicate, skipped, and rejected rows with downloadable row-level errors.

Outbound adapters subscribe to domain events and execute idempotently. Rate limits, circuit breakers, cursor checkpoints, health, last success, and operator replays are standard. A connection declares source-of-truth direction for each entity. Ambiguous conflicts enter a queue rather than silently winning.

Calendar and booking integrations first target read/import and schedule synchronization; payments/accounting are isolated adapters to Commerce. BLE/BMS telemetry enters a time-series ingestion boundary with validation and alert rules, never privileged operational control.

## 21. Audit architecture

Audit events capture ID, occurred/recorded times, organization/facility, actor principal and impersonator, source IP/user agent where appropriate, action, object type/ID, correlation ID, reason, and redacted before/after diffs. Security, role, policy, form, workflow, export, API key, integration, and elevated-support actions are mandatory.

The application role has insert-only access through a controlled function and no update/delete permission. Daily partitions support retention and export. Hash chaining or periodic signed digests anchored outside the database make tampering evident; backups and restricted administrator access remain essential. Sensitive values and secrets are redacted at write time. Read access to audit data is itself audited.

Audit events answer who changed what; domain events drive system reactions. They may correlate but are not interchangeable. Retention schedules are category- and jurisdiction-aware, with legal hold and approved deletion/anonymization processes.

## 22. Notification architecture

Domain/workflow events create notification intents. Recipient resolution evaluates explicit users, assignments, scoped roles, departments, on-duty status, severity, preferences, and escalation policy at send time. Channel adapters support in-app, push, email, SMS, and webhook.

Templates are versioned, localized, branded, and schema-validated. A deduplication/grouping window and per-rule throttle prevent storms. Quiet hours defer low severity only; mandatory critical notices follow approved escalation and acknowledgement rules. Delivery attempts, provider IDs, status, and redacted errors are observable. Preferences cannot disable legally or operationally mandatory categories without an authorized policy change.

## 23. Security threat model

| Threat | Primary controls | Required verification |
|---|---|---|
| Cross-tenant/object access | tenant columns, composite FKs, forced RLS, scoped evaluator | negative SQL/API/storage tests |
| Privilege escalation | grant constraints, separate platform roles, JIT support, audit | role mutation and confused-deputy tests |
| Public data leakage | publication projections, opaque tokens, field allowlists | anonymous enumeration tests |
| Stolen API/session credentials | hashed keys, rotation, short sessions, MFA, rate limits | revocation and scope tests |
| Injection/XSS | parameterized SQL, schema validation, output encoding, CSP | SAST and browser security tests |
| SSRF/webhook abuse | HTTPS allowlist policy, DNS/IP checks, redirect limits | rebinding/private-network tests |
| Malicious files/imports | content/type/size checks, malware scanning, quarantine | adversarial upload/import tests |
| Offline device loss | limited cache, session expiry, device revoke, minimal sensitive data | revocation/expiry tests |
| Event replay/duplication | signatures, timestamps, idempotency, aggregate versions | replay and ordering tests |
| Insider/elevated misuse | least privilege, JIT reason/expiry, immutable audit, alerts | support-access review |
| Availability/noisy tenant | quotas, pagination, timeouts, queues, circuit breakers | load and fault-injection tests |
| Supply-chain compromise | lockfiles, dependency scanning, signed builds, least-privilege CI | SBOM and deployment provenance |

Security baselines include MFA for privileged users, secure headers, CSRF controls, secret scanning, encryption in transit/at rest, backup restore drills, vulnerability response, and documented incident response. Data classification controls logging, exports, retention, and support access.

## 24. Performance and scaling plan

- Establish SLOs before implementation: interactive p95 target, dashboard freshness, background job latency, webhook delivery, and availability.
- Index all foreign keys and frequent access paths beginning with `organization_id`; use composite indexes matching tenant/scope/status/time filters.
- Require cursor pagination and bounded date ranges. Prohibit unbounded `.select()` patterns through repository conventions and tests.
- Use connection pooling, query timeouts, `EXPLAIN (ANALYZE, BUFFERS)` review for critical queries, and production slow-query telemetry.
- Partition high-volume append tables (audit, events, notifications, telemetry) by time after measured thresholds; retain organization in indexes.
- Maintain dashboard metric facts asynchronously and cache safe public projections at the edge with explicit invalidation/TTL.
- Archive cold attachments and old event payloads according to retention. Keep canonical summaries and audit evidence accessible.
- Load-test the reference tenant: 28 then 50 facilities, 53+ sheets, 1,000 employees, years of records, concurrent shift changes, bulk template rollout, and integration bursts.

Scale triggers—not guesses—govern introducing read replicas, a dedicated queue, search service, time-series store, or analytics warehouse. Each extraction retains organization context, authorization, audit, and replay contracts.

## 25. Migration strategy

Migrations are timestamped/numbered, immutable after merge, transactional where PostgreSQL permits, deterministic, and replayed from zero in CI. Expand/contract deployment is mandatory: add compatible schema, deploy dual-compatible code/backfill, validate, switch reads, then remove old schema in a later release.

Large backfills are resumable, tenant-batched, observable jobs with checkpoints and invariants; they do not lock hot tables for long periods. New `NOT NULL` and validated constraints use staged defaults/backfill/validation. Concurrent indexes run outside transactions with explicit tooling. Every migration documents lock risk, rollback/forward-fix, application ordering, and verification SQL.

Existing data discovery precedes mapping. Legacy IDs are retained in a mapping table, duplicate concepts are reconciled, rejected rows are reported, and tenant ownership is verified before import. Schema types are generated into TypeScript after migration. Production drift is detected against the migration history; dashboard checks show application/schema compatibility.

**Decision — rollback.** Alternatives: down migrations for every change, restore-only, forward-fix plus expand/contract. **Recommendation:** forward-fix with backups and expand/contract; reversible down scripts only when demonstrably safe. **Reasoning:** destructive rollback after new writes often loses data. **Implications:** releases need compatibility windows and restore drills.

## 26. Testing strategy

The pyramid includes pure unit tests for policies/rules/calculations; database integration tests for constraints, functions, migrations, and RLS; API contract tests; worker/event idempotency tests; adapter contract tests; and a small set of end-to-end journeys.

Mandatory tenant matrix tests cover Organization A/B, same-organization Facility A/B, region scope, multi-facility grants, expired grants, temporary assignments, sensitive capabilities, anonymous publications, and over-scoped API keys. Each includes positive and negative select/create/update/archive/report/export cases. Property tests exercise scope containment and rule DSL safety.

Fresh-database and upgraded-database suites apply all migrations and compare schema. Event tests inject duplicates, reordering, partial failure, retries, and replay. Offline tests cover revoked access, obsolete form versions, concurrent edits, queue dependencies, and attachment interruption. Accessibility tests combine automated WCAG checks with keyboard, screen reader, touch target, glove/tablet, poor-light, and offline field trials.

Performance tests seed the critical reference tenant and measure dashboards, filtered lists, scheduling conflict detection, bulk assignment, exports, RLS overhead, and job throughput. Restore, secret rotation, webhook signing, SSRF, and incident-response exercises run on a schedule.

## 27. CI/CD strategy

Pull requests run formatting, lint, strict TypeScript, unit tests, dependency/secret/SAST scans, migration lint, fresh and upgrade database tests, generated-type drift, RLS coverage/isolation tests, API contract checks, accessibility smoke tests, and production build. A seeded preview environment supports review without production data.

Protected branches require review and passing checks. Build artifacts are immutable and promoted across environments with provenance/SBOM. Deployment order is expand migration → compatible application/workers → backfill/verification → flag enablement; destructive contracts occur separately. Smoke tests and error/SLO gates drive automated halt or application rollback, while schema uses forward fixes.

Production access is least privilege with environment separation, protected secrets, audited break-glass, backups, point-in-time recovery, and scheduled restore tests. Feature flags enable internal, pilot facility, selected organization, percentage, then global rollout.

## 28. Observability

Structured logs carry request/job ID, correlation/causation IDs, deployment version, organization/facility pseudonymous IDs, route/job, duration, and outcome—never secrets or sensitive field payloads. OpenTelemetry traces follow HTTP → database/outbox → worker → provider. Metrics cover latency, errors, saturation, queue age, outbox lag, workflow failures, notification/webhook delivery, sync conflicts, integration freshness, and database health.

Sentry captures application errors with privacy scrubbing; PostHog captures approved product analytics with tenant-aware governance and no sensitive record contents. Platform health dashboards separate infrastructure, deployment, database, queue, integration, webhook, and offline-sync status. Alerts map to runbooks, severity, owner, escalation, and post-incident review.

Customer administrators see their own integration/webhook/sync health; platform operators see aggregate infrastructure health with controlled drill-down. Audit is not substituted by application logging.

## 29. Feature flags

Flags are typed, named, owned, documented with expiry, and evaluated server-side for authoritative behavior. Targets support internal users, organizations, facilities, and deterministic percentage rollout. Client evaluations may hide UI but never bypass server authorization.

Flag decisions are logged as metadata for debugging, not as sensitive analytics. Changes to security- or data-affecting flags require audit and approval. Database migrations must be safe while either flag state runs. Stale flags fail CI policy or appear on a cleanup dashboard; permanent entitlements live in plan/capability data, not rollout flags.

## 30. Phased build plan and gates

| Phase | Outcome | Exit gate |
|---|---|---|
| 0 Discovery & architecture | Approved blueprint, ADRs, domain glossary, threat model, test/load plan | approval decisions resolved |
| 1 Enterprise foundation | tenancy, hierarchy, workforce, scoped auth, RLS, audit, switchers | isolation matrix passes |
| 2 Corporate admin | directory admin, bulk imports, settings, assignments, onboarding | 28-facility seed onboarded |
| 3 Forms/records | versioned forms, assignments, typed records, approvals | standard/template adoption proven |
| 4 Workflow/events | outbox, rules, tasks, notifications, retries | replay/idempotency proven |
| 5 Scheduling | resources, conflicts, availability, operational offsets | schedule-driven workflow proven |
| 6 Core operations | daily, ice, readings, safety, paperwork, communications | field/offline acceptance passed |
| 7 Command Center | status, alerts, comparisons, drill-down | freshness/load/explainability passed |
| 8 Reporting | semantic metrics, saved/scheduled reports, exports | authorization/export limits passed |
| 9 Connect | widgets, API, webhooks, integrations, displays | public/private boundary pen-tested |
| 10 Customer platform | shared customer, household, teams, programs | cross-facility dedupe verified |
| 11 Registration/reservations | public enrollment and reservation | capacity/concurrency verified |
| 12 Commerce | isolated invoices/payments/refunds/POS decisions | finance/security review passed |

No phase is a mandate to ship its full scope in one release. Vertical slices should deliver one complete, observable journey behind flags while preserving the phase dependencies.

## 31. Principal risks and mitigations

| Risk | Mitigation / trigger |
|---|---|
| Authorization becomes incomprehensible | fixed scope containment, grant simulator, explain-access endpoint, no v1 explicit deny |
| RLS performance or gaps | helper benchmarks, tenant-first indexes, schema policy gate, negative tests |
| Universal engine becomes a low-code trap | bounded field/action catalogs, explicit aggregate modules, architecture review |
| Dashboard data is stale or misleading | visible freshness, reconciliation, definitions/denominators, source drill-down |
| Event side effects duplicate | outbox, idempotency keys, aggregate versions, replay tests |
| Offline leaks or conflicts | curated cache, expiry/revoke, sensitivity limits, visible conflict queue |
| Integration variability consumes roadmap | adapter contract, first-class mappings/health, narrow initial providers |
| Bulk changes harm all facilities | preview/dry run, staged rollout, validation, rollback by new version |
| Sensitive safety/employee data spreads | classification, dedicated capabilities, minimized events/logs/exports |
| Premature microservices/warehouse add burden | measured extraction thresholds and modular boundaries |
| Existing data cannot map cleanly | discovery/profile, legacy mappings, quarantine, reconciled imports |
| Scope overwhelms delivery | phase gates, vertical slices, flags, explicit non-goals and approvals |

## 32. Assumptions

1. One legal/operating customer maps to one organization tenant; complex franchise sharing is not required in v1.
2. A facility belongs to one organization and zero or one region at a time.
3. Region nesting beyond one level is unnecessary initially.
4. Supabase/PostgreSQL remains the system of record and selected deployment region satisfies initial residency needs.
5. Most organizations remain below 50 facilities, 1,000 employees, and low millions—not billions—of annual operational facts.
6. Internet connectivity is intermittent, not absent for weeks; selected operational commands can be offline but sensitive administration is online-only.
7. Corporate administrators own template/workflow governance; facilities can customize only declared extension points.
8. Payroll remains external; labor hours can be exchanged without calculating payroll.
9. Public widgets expose only deliberately published information and do not serve authenticated employee operations.
10. Legal retention, consent, signature, SMS, accessibility, and data-residency requirements will be validated with counsel and pilot customers before launch.

## 33. Decisions requiring approval

| ID | Decision | Recommended choice | Needed before |
|---|---|---|---|
| A1 | Hierarchy depth | fixed organization/optional region/facility | Phase 1 schema |
| A2 | Tenant topology | shared schema + forced RLS | Phase 1 schema |
| A3 | Identifier generation | UUIDv7 where supported; opaque public IDs | Phase 1 migrations |
| A4 | Authorization | scoped RBAC + limited ABAC; no explicit deny v1 | Phase 1 roles |
| A5 | Support access | JIT audited platform elevation | production operations |
| A6 | Forms storage | typed hybrid values + versioned JSONB definitions | Phase 3 |
| A7 | Rule language | restricted versioned DSL | Phases 3–4 |
| A8 | Async execution | PostgreSQL outbox/worker initially; managed queue trigger defined by load | Phase 4 |
| A9 | Analytics | PostgreSQL facts/read models before warehouse | Phase 7 |
| A10 | Embed delivery | iframe loader and public projections | Phase 9 |
| A11 | Offline scope | curated cache + command queue; admin online-only | Phase 6 |
| A12 | Audit tamper evidence | append-only plus external signed digest | Phase 1 production |
| A13 | Initial integrations | choose first 2–3 providers and system-of-record rules | Phase 9 |
| A14 | Compliance baseline | jurisdictions, retention, accessibility target, incident data policy | before pilot |
| A15 | SLOs/RTO/RPO | approve numeric service and recovery objectives | before production |

### Major decision summary

The common alternatives—microservices first, database-per-tenant, arbitrary hierarchy, opaque JSON records, exactly-once messaging, warehouse first, injected widgets, and broad offline replication—were rejected for initial delivery because they increase operational or security complexity without improving the stated 28-facility scenario. The recommendations preserve extraction seams: bounded modules, organization context in every event, versioned APIs, outbox consumers, governed metrics, and adapter contracts. Approving these choices commits the team to disciplined shared-schema controls and compatibility testing, not to a permanent monolith.

## 34. First implementation tickets

Tickets are ordered; each must include migrations, generated types, tests, documentation, audit/observability, and rollout controls where applicable.

1. **ADR package and domain glossary.** Ratify A1–A15, naming, ownership, data classification, SLO candidates, and module boundaries.
2. **Local/CI database harness.** Reproducible Supabase stack, zero-to-head migrations, seed runner, drift check, and generated TypeScript types.
3. **Tenant foundation migration.** Organizations, organization memberships, regions, facilities, constraints, indexes, timestamps, and archival conventions.
4. **Area and department model.** Areas plus typed ice sheet/room/locker-room extensions, departments, operating timezone, and hierarchy APIs.
5. **Principal and employee model.** User profiles, organization employees, home facility, department memberships, and time-bounded facility assignments.
6. **Capability catalog and scoped grants.** Roles, permissions, role grants, separation-of-duties validation, and explain-access function.
7. **RLS helper library and policy baseline.** Forced policies for tickets 3–6, storage conventions, lint/coverage tooling, and service-role restrictions.
8. **Tenant isolation matrix suite.** Two-organization fixtures and positive/negative SQL/API/storage tests for corporate, regional, facility, Ice Tech, public, expired, and service principals.
9. **Append-only audit foundation.** Controlled writer, redaction, partitions, privileged-read audit, correlation, and signed-digest spike.
10. **Authenticated organization/facility context APIs.** Server-derived context, switcher endpoints, safe persistence, and URL state; no production dashboard yet.
11. **Enterprise onboarding read model.** Twelve-step readiness requirements, facility readiness, warnings, and explainable completion calculation.
12. **Bulk import framework.** Upload quarantine, schema mapping, dry run, row errors, idempotent commit, progress/counts, and employee/facility pilot schemas.
13. **Forms engine schema spike.** Forms/versions/fields/assignments, restricted expression DSL, typed value prototype, validation benchmarks, and ADR confirmation.
14. **Template rollout vertical slice.** Publish an opening checklist, target all/region/facilities, preview impact, track adoption, and preserve immutable submissions.
15. **Domain event/outbox foundation.** Versioned envelope, transactional append, dispatcher, idempotency, retries, dead letter, trace correlation, and replay tooling.
16. **Task/notification workflow slice.** On a threshold event, create one deduplicated task and in-app notification with scoped recipients and audit trail.
17. **Scheduling/resource schema spike.** Events, allocations, recurrence, exclusive-resource constraints, timezone/DST tests, and operational offsets.
18. **Command Center metric proof.** Facility status/attention read models for overdue tasks and equipment outage, transparent rules, region filter, freshness, and source drill-down.

## 35. Critical success scenario proof

### Reference organization

Seed one organization with four regions, 28 facilities, 53 typed ice-sheet areas, approximately 800 employees, departments, scoped grants, equipment, schedules, records, and alert/task history. A second organization of similar shape exists solely to prove isolation.

### Corporate operations manager journey

| Requirement | Architectural mechanism |
|---|---|
| 1. Log in once | one Supabase identity with an organization membership and organization-scoped role grant |
| 2. See 28 facilities | tenant-filtered facility directory and status projection with pagination |
| 3. Identify attention | explainable `attention_items` and `facility_status_current`, including unknown/stale state |
| 4. Filter by region | server filter constrained by scoped grant; URL preserves context |
| 5. Compare compliance | governed metric facts with explicit numerator, denominator, version, and freshness |
| 6. Inspect safety | dedicated safety capabilities and sensitivity-aware RLS/read models |
| 7. Review equipment | canonical fleet, current assignment, downtime, inspection, and work-order relations |
| 8–10. Drill down and return | canonical routes plus preserved organization/region/filter state |
| 11. Push checklist to 28 locations | immutable form version, organization assignment, preview, adoption tracking |
| 12. Update one region workflow | new workflow version with region assignment and impact simulation |
| 13. Temporary employee assignment | bounded facility assignment plus separately bounded role grant and qualification checks |
| 14. Multi-location available ice | availability query across authorized facility resources, closures, buffers, and allocations |
| 15. Corporate schedule embed | origin-bound organization publication token and iframe widget over public projections |
| 16. Selected API data | service account scopes, RLS, rate limits, field allowlists, and audit |
| 17. Organization report without leakage | tenant-first facts, authorization intersection, RLS, export audit, and two-tenant negative tests |

At reference load, automated acceptance tests must verify the manager sees exactly 28 facilities and never the second tenant, dashboard freshness meets the approved SLO, all counts drill to matching authorized sources, a bulk rollout reports 28 adoption targets without silent loss, and region workflow changes do not affect the other three regions.

### Facility 14 Ice Tech journey

The Ice Tech has an employee link, active Facility 14 assignment, department membership, and an Ice Tech facility-scoped role. Server navigation is derived from authorized capabilities. Queries and offline reference caches are facility-limited. The worker sees assigned tasks, authorized areas/forms, Facility 14 schedule, and assigned equipment; corporate comparisons, other facilities, restricted incidents, configuration, and bulk controls are neither returned by APIs nor selectable through RLS.

Acceptance tests attempt direct IDs, modified filters, report/export endpoints, stale offline data, public APIs, and cross-tenant references. Every unauthorized operation must return no data or a consistent authorization error without revealing object existence. A temporary assignment to another facility becomes visible only during its valid window and only with the required capability grant.

### Verdict

The architecture supports both experiences without separate products: the same tenant model, scoped grants, canonical entities, event stream, and reporting facts power both. Corporate scale comes from broader grants and aggregated projections; the Ice Tech experience comes from narrow grants and task-oriented presentation. Neither experience weakens the database boundary.

## 36. Architecture completion gate

Phase 0 is complete only when:

- A1–A15 have named approvers and recorded outcomes.
- The core ER model, scope containment, event envelope, and data classification are reviewed by product, engineering, security, and operations.
- The critical scenario is represented as executable seed/acceptance specifications.
- Numeric SLO, RTO, RPO, retention, residency, and compliance requirements are approved.
- The first tickets are estimated and sequenced with owners and dependencies.
- No production feature depends on an unresolved decision that could change tenancy, authorization, identity, event, or record foundations.

Until then, repository changes should remain documentation, prototypes, disposable spikes, and test harnesses—not production feature implementation.
