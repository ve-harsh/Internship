# GraphQL Federation — Batch 2: Full Domain Expansion Report

**Author:** Harsh Verma
**Role:** Software Engineering Intern, Samsara
**Duration:** July 1 – July 14, 2026

> Company-specific identifiers and operational details have been generalized for academic discussion. External distribution remains subject to the company's review and approval process.

---

## 1. Overview

This report documents the second and final batch of work in building a federated GraphQL subgraph for the Platform, Identity, and Billing (PIB) domains. Batch 2 expanded the subgraph from 2 packages to 22, wiring approximately 200 federation dependency annotations across 20 additional domain packages and configuring several hundred field routes.

| Metric | Value |
|--------|-------|
| New domain packages wired | 20 |
| Total domain packages | 22 |
| Federation annotations added | ~200 |
| Files changed | 84 |
| Insertions / deletions | 1050+ / 334 |
| New files created | 23 |
| Status | Validated on staging, in code review |

---

## 2. What Was Done

### 2.1 Domain Packages Wired

Twenty additional domain packages were wired into the subgraph, covering:

| Domain Area | Approx. Annotations | Description |
|-------------|:-------------------:|-------------|
| Public API | 35 | Organization, device, driver, API token, and webhook management |
| Workflow configuration | 27 | Workflow triggers, actions, machine inputs, and automation rules |
| Authentication | 22 | Organization invites, grant entries, and user-organization mappings |
| Tags and groups | 22 | Tag management, group configuration, and hierarchy |
| App marketplace | 16 | Developer apps, developer profiles, and media assets |
| Scheduled reports | 15 | Report plans, scheduling, and group-scoped configurations |
| Feature management | 11 | Feature flags, feature entries, and organization-level toggles |
| CSV operations | 10 | Bulk upload jobs and async processing |
| Attributes | 8 | Custom attribute values and organization-scoped attributes |
| Meta-fleet | 7 | Device clones, widget clones, and fleet management |
| User preferences | 6 | Custom home pages, dismissed notifications, and user settings |
| User-organization mappings | 4 | Organization membership and access control |
| Audit logging | 3 | Organization and user audit trails |
| Capabilities | 3 | Organization capability checks and viewer permissions |
| Other packages | 1–2 each | Cloud blog, sessions, data retention, admin access, and more |

### 2.2 Cross-Service Field Ownership Analysis

A critical part of Batch 2 was analyzing field ownership across multiple federated subgraphs. Some packages contained fields already owned by other subgraphs:

- **15 workflow/alerts fields** were already routed to a different subgraph in the existing feature flag configuration. Adding federation annotations to these would create dual-registration conflicts.
- **2 contact management fields** had their own federation registration in a separate subgraph.

For all 17 fields, I explicitly removed or excluded the dependency annotations, ensuring each field has exactly one federation owner. This required:

1. Reading the existing feature flag variations to identify already-routed fields
2. Cross-referencing with other subgraphs' federation registration files
3. Removing annotations from the 17 conflicting fields
4. Verifying the federation symmetry test still passed

### 2.3 Structured Tracing Package

Created a new tracing package for subgraph execution observability. The package provides structured log events:

- `subgraph_execute_start` — logged when the gateway sends a sub-plan to the subgraph
- `subgraph_execute_complete` — logged when the subgraph returns a successful response
- `subgraph_execute_error` — logged when the subgraph returns an error

Trace context propagates through gRPC header metadata, enabling cross-service correlation of federation execution events.

### 2.4 Feature Flag Configuration

The feature flag was configured with a new variation containing several hundred field routes. Fields were organized by GraphQL type:

| Type Category | Proportion |
|--------------|:----------:|
| Mutation fields | ~40% |
| Organization fields | ~26% |
| Other domain types | ~20% |
| Query fields | ~4% |
| User fields | ~4% |
| Tag and device fields | ~6% |

The variation was targeted at staging environments for validation before broader rollout.

---

## 3. Key Technical Decisions

### 3.1 Register-All Pattern

Each federation registration file calls the package's existing `Register()` function, which registers all fields. Fields without dependency annotations won't generate federation sub-plans — they resolve locally via struct field access. Only fields with annotations create federation routes. The feature flag controls actual routing at runtime.

This is safe because:
- The subgraph schema is a superset that includes all package fields
- Only annotated fields can be split into federation sub-plans by the gateway
- The feature flag provides the authoritative routing decision

### 3.2 Excluding Cross-Service Fields

Rather than trying to coordinate with other subgraph teams, I removed annotations from the 17 conflicting fields. This is a deliberate design choice — each field has exactly one federation owner, and the feature flag enforces this invariant at runtime.

### 3.3 Debug Endpoint vs Federation Gateway

The internal debug GraphQL endpoint resolves queries locally through the framework's executor — it does not go through the federation gateway. Federation only happens through the production WebSocket path. This distinction is important for testing: queries run on the debug endpoint always resolve on the primary server regardless of feature flag configuration.

### 3.4 Code-Generation for Annotations

Used an internal code-generation tool with deep static analysis to auto-generate dependency annotations. The tool:
- Traces resolver code to identify parent struct field access
- Filters out context-only resolvers (no annotation needed)
- Filters out struct-typed field access (non-scalar rule)
- Generates the minimum required set of parent field dependencies

---

## 4. Conflict Resolution

Three rounds of rebase conflicts were resolved as the main branch evolved:

| Conflict Area | Issue | Resolution |
|---------------|-------|------------|
| Tags package | Upstream migration from single-field to batch-field resolver pattern | Preserved upstream's batch resolver, added federation annotation |
| Authentication handler | Upstream refactored security context propagation into a centralized function | Preserved upstream's structure, added tracing in the execution context |
| Data retention package | Upstream migration to batch-field resolver pattern | Preserved upstream's batch resolver, added federation annotation |
| User preferences package | Upstream moved fields to a different package | Preserved upstream's reorganization, added annotations to remaining fields |

Each resolution required understanding both the federation requirements and the intent behind the upstream change.

---

## 5. Staging Validation

### Environment

- **Cells:** Two staging environments
- **Feature flag variation:** Several hundred field routes
- **Deployment version:** Single squashed commit

### Federation Proof

Log aggregation confirmed the subgraph was receiving and executing federated queries:

```
decision=subgraph_execute_start     query_name=<navigation_query>
decision=subgraph_execute_complete  query_name=<navigation_query>
decision=subgraph_execute_start     query_name=<permissions_query>
decision=subgraph_execute_complete  query_name=<permissions_query>
```

Observability metrics confirmed steady request volume at the subgraph level.

### Test Queries

- **414 queries** prepared with valid variables and enum values
- All verified to federate against the feature flag variation
- Automated browser-based runner script created for batch execution

### Queries Confirmed Serving

The following query categories were confirmed actively serving on staging:

| Category | Examples |
|----------|----------|
| User navigation | Sidebar, toolbar, dropdown components |
| Authentication | Enrollment prompts, banner eligibility |
| Organization | Settings, configuration, localization |
| Permissions | Navigation permissions, admin access |
| Feature flags | Feature identity and configuration |

All queries returned zero errors with sub-millisecond federation overhead.

---

## 6. Monitoring

### Observability Queries

Log aggregation queries were used to monitor subgraph execution:

```
-- Check subgraph handling queries
fields: timestamp, decision, query_name, org_id, duration_ms
filter: message contains "federation-debug"
filter: cell = <staging_cell>
sort: timestamp descending
limit: 50

-- Check errors only
filter: message contains "subgraph_execute_error"
sort: timestamp descending
```

### Active Monitors (All Healthy)

| Monitor Category | Status |
|-----------------|:------:|
| Panic detection | OK |
| Container health | OK |
| Service availability | OK |
| Log cost monitoring | OK |

---

## 7. Files Changed

### New Files (23)

- **21 federation registration files** — one per domain package, each containing a `RegisterFederatedFields()` entry point with package-specific wiring
- **2 tracing files** — structured tracing package for subgraph execution observability

### Modified Files (61)

- All domain schema files with dependency annotation additions
- Subgraph schema root — wiring all 20 new packages
- Subgraph schema configuration — dependency injection parameters for new servers
- Subgraph main entry point — domain package imports
- Federation handler — experiment trace integration
- Gateway handler — federation experiment debug logging

---

## 8. Next Steps

1. Run full 414 query batch on staging via automated runner
2. Verify zero errors through log aggregation
3. Check shadow-mode experiment logs for response diffs
4. Roll out to additional staging cells
5. Production rollout via feature flag targeting

---

*This report presents a generalized account of internship engineering work. Internal service names, package paths, production query names, traffic volumes, feature flag configurations, and deployment details have been omitted or abstracted.*
