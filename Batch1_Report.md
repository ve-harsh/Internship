# GraphQL Federation — Batch 1: Foundation Report

**Author:** Harsh Verma
**Role:** Software Engineering Intern, Samsara
**Duration:** June 15 – July 1, 2026

> Company-specific identifiers and operational details have been generalized for academic discussion. External distribution remains subject to the company's review and approval process.

---

## 1. Overview

This report documents the first batch of work in building a federated GraphQL subgraph for the Platform, Identity, and Billing (PIB) domains. Batch 1 established the foundation by wiring the two most critical domain packages — platform operations and user management — and building the database-backed entity resolver required for authorization safety.

| Metric | Value |
|--------|-------|
| Domain packages wired | 2 |
| Field resolvers | ~30 |
| Dependency annotations | 24 |
| Files changed | 14 |
| Status | Merged |

---

## 2. What Was Done

### 2.1 Federation Registration

Created federation registration files for two core domain packages:

| Domain | Area | Field Resolvers | Dependency Annotations |
|--------|------|:---------------:|:---------------------:|
| Platform Operations | Organization management, maintenance status, trial detection | 11 | 1 |
| User Management | User profiles, permissions, regions, organization memberships | 19 | 23 |

Each registration file provides a `RegisterFederatedFields()` entry point that wires all package fields into the subgraph's schema, adds dependency annotations for federation routing, and registers subgraph-specific root query resolvers.

### 2.2 Database-Backed Entity Resolver

Built a database-backed federation entity resolver for the user-organization mapping type. This entity carries authorization-critical fields (organization identifier and group identifier) that the authorization system reads to determine permission domains.

**Why a passthrough resolver was insufficient:**

A passthrough resolver only forwards the entity's primary key without loading the full record. This would leave authorization fields zero-valued, causing the permission system to evaluate against empty domains — potentially granting incorrect access or blocking legitimate requests.

**Solution pattern:**

```go
type FederatedEntityFetcher struct {
    repository EntityRepository
}

func (f *FederatedEntityFetcher) Fetch(
    ctx context.Context,
    keys []EntityKey,
) ([]*Entity, error) {
    ids := extractIDs(keys)

    entities, err := f.repository.BatchLoad(ctx, ids)
    if err != nil {
        return nil, fmt.Errorf("batch load failed: %w", err)
    }

    byID := indexByID(entities)
    result := make([]*Entity, len(keys))
    for i, key := range keys {
        result[i] = byID[key.ID]
    }
    return result, nil
}
```

The resolver is registered globally so all subgraphs benefit from it, following the established pattern for the User entity resolver.

### 2.3 Schema Configuration

- Added read-write and read-only schema split for the platform operations package (separate database connections)
- Registered the subgraph in the compile-time server registry
- Added import boundary rules for the new subgraph package

### 2.4 Dependency Annotations

Used an internal code-generation tool to auto-generate dependency annotations across all field resolvers. The tool traces resolver code to determine which parent struct fields each resolver accesses, enabling the federation planner's "fast path."

---

## 3. Staging Validation

### Environment

- **Cell:** Staging environment
- **Feature flag:** Configured with approximately 27 field routes
- **Shadow-mode experiment:** 100% sample rate

### Results

| Metric | Value |
|--------|-------|
| Total queries tested | 147 |
| Pass | 119 (81%) |
| Errors | 28 (19%) |
| **PIB-related errors** | **7** |
| Non-PIB errors (false positives) | 21 |

### Error Breakdown

| Category | Count | Details |
|----------|:-----:|---------|
| False positives | 18 | Queries matching on field names that exist on non-PIB types |
| Bad test input | 4 | Incorrect variable formats in test queries |
| Pre-existing bug | 1 | Existing resolver bug unrelated to federation |
| Investigate | 2 | Queries with PIB fields that fail on non-PIB portions |
| Already fixed | 1 | Test timing issue resolved during development |
| Duplicates | 2 | Duplicate test queries |

### Key Finding

**Zero real federation bugs** after the database-backed entity resolver fix. The one federation issue encountered (partial entity objects with empty authorization fields) was the exact problem the DB-backed resolver was designed to solve, and it was fixed before the validation run.

### Shadow-Mode Experiment

The shadow-mode experiment validated 68 unique production query types during a one-week observation window. These queries spanned navigation, user management, permissions, organization configuration, coaching, forms, developer tools, and more — confirming the subgraph handled real production patterns correctly.

### Known Benign Differences

One field (`getUserPermissions.actions`) showed response differences due to non-deterministic Go map iteration order. This is a serialization artifact, not a semantic difference — the same set of values is returned in a different order.

---

## 4. Production Traffic

After merging, the subgraph was deployed to a production staging cell and began serving real customer traffic.

Over a 24-hour observation window:

| Metric | Value |
|--------|-------|
| Total queries served | Tens of thousands |
| Unique query types | 43 |
| Errors / panics | **0** |

Query types spanned user navigation, sidebar components, permissions, organization management, coaching features, forms, developer tools, asset maintenance, and more.

---

## 5. Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **DB-backed entity resolver** | Authorization-critical fields must be fully hydrated; passthrough would risk wrong permission evaluation |
| **RW/RO schema split** | The platform domain uses separate read-write and read-only database connections; the subgraph must respect this separation |
| **Dependency annotations** | Enable the federation planner to request only the parent fields each resolver actually needs, avoiding unnecessary DB lookups |

---

## 6. CI Validation

All CI checks passed:

| Check Category | Status |
|----------------|:------:|
| Backend tests | Pass |
| Backend integration | Pass |
| GraphQL schema validation | Pass |
| Frontend tests | Pass |
| Mobile tests | Pass |
| Federation symmetry | Pass |
| Code review | Approved |

---

*This report presents a generalized account of internship engineering work. Internal identifiers, production query names, traffic volumes, deployment details, and configuration specifics have been omitted or abstracted.*
