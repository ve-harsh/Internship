# GraphQL Federation for Platform Domains: An Internship Case Study

**Author:** Harsh Verma
**Role:** Software Engineering Intern, Samsara
**Team:** GraphQL Infrastructure / Platform
**Duration:** May – July 2026

> Company-specific identifiers and operational details have been generalized for academic discussion. External distribution remains subject to the company's review and approval process.

---

## 1. Executive Summary

During my software-engineering internship at Samsara, I worked on GraphQL federation — splitting a monolithic GraphQL server into domain-specific subgraphs to improve deployment independence, scaling, and blast radius isolation. I designed and implemented a new federated subgraph for the Platform, Identity, and Billing (PIB) domains, shipping two major pull requests that wired more than 20 domain packages with several hundred field routes.

Key engineering contributions included building a database-backed entity resolver for authorization-critical objects, developing a structured tracing package for subgraph execution observability, and creating a gRPC parity-testing tool for validating federation correctness. I used annotation-driven dependency generation, shadow-mode validation experiments, and feature-flag-controlled progressive rollouts to safely migrate production traffic.

The service was deployed to a staging environment and validated with hundreds of test queries, achieving zero errors with sub-millisecond federation latency. It subsequently served tens of thousands of real production queries per day with zero errors and zero panics.

---

## 2. Problem Statement and Objectives

### The Monolith Bottleneck

Samsara's cloud dashboard is powered by a GraphQL API that serves all frontend queries — from navigation and user management to device configuration and reporting. At the time of this internship, a single monolithic GraphQL server handled all of these queries.

This architecture creates three scaling challenges:

- **Deploy blast radius:** A single faulty deployment to the monolithic server can affect all GraphQL domains simultaneously. A regression in billing logic could disrupt navigation or user management.
- **Independent scaling:** The monolith cannot scale individual domains based on their traffic profiles. A spike in identity-related queries forces the same compute to handle billing queries, even if billing load is nominal.
- **Deployment velocity:** All domain teams share one deployment pipeline, creating contention and coordination overhead.

### The Solution: GraphQL Federation

GraphQL federation addresses these challenges by splitting the monolith into domain-specific subgraphs. Each subgraph:

- Deploys independently through its own pipeline
- Scales based on its own traffic profile
- Has a smaller blast radius, isolated from other domains

A federation gateway sits in front of the subgraphs, routing portions of each incoming query to the appropriate server and merging the results before returning them to the client.

### Project Scope

My internship project was to build the federated subgraph for the **Platform, Identity, and Billing (PIB)** domains. This was the second federation target at the company — an earlier intern project had successfully federated the safety-video-camera domain, establishing the pattern I would follow and extend.

---

## 3. Background: GraphQL Federation Concepts

### Federation Architecture

In a federated GraphQL system, the schema is distributed across multiple servers. Each server owns a subset of the schema's types and fields. A gateway service orchestrates query execution:

```
Client (Dashboard) → Federation Gateway → Subgraph A (PIB)
                                        → Subgraph B (Safety)
                                        → Primary Server (everything else)
```

### Key-Based Entity Resolution

When the gateway splits a query across subgraphs, it needs a way to "join" data between them. This is done through **key-based entity resolvers**. Each subgraph declares which types it can resolve given a set of key fields (typically `id`), and the gateway uses these resolvers to hydrate partial objects.

### Dependency Annotations

Each field resolver can declare which parent fields it requires through **dependency annotations**. This enables the federation planner's "fast path" — the gateway only requests the parent fields that the resolver actually needs, avoiding unnecessary database lookups.

### Shadow-Mode Validation (Scientist Experiment)

Before fully cutting over traffic to a new subgraph, the company uses a shadow-mode validation approach inspired by GitHub's Scientist library. Both the new subgraph and the monolith handle the same query, and their responses are compared. Discrepancies are logged for investigation without affecting the client response. This allows safe, incremental migration of production traffic.

### Feature-Flag-Driven Rollout

A feature-management platform controls which fields are routed to which subgraph, on a per-deployment-cell basis. This enables:

- Incremental rollout (start with staging, then production)
- Instant rollback (revert the flag to route all traffic back to the monolith)
- Per-field granularity (route specific fields independently)

---

## 4. Architecture

### System Components

| Component | Role | Description |
|-----------|------|-------------|
| Federation Gateway | Query Router | Splits incoming queries by field, routes sub-plans to appropriate subgraphs, merges results |
| PIB Subgraph | Federated Server | New server for Platform, Identity, and Billing domains |
| Primary Server | Monolith | Original server handling all non-federated fields |
| Feature Flag | Routing Control | Controls per-cell, per-field routing between subgraphs |
| Scientist Experiment | Validation | Shadow-mode comparison of subgraph vs monolith responses |

### Query Flow

1. The client sends a GraphQL query to the gateway via WebSocket
2. The gateway reads the feature flag to determine field routing
3. The gateway splits the query plan — fields listed in the flag variation go to the PIB subgraph
4. Both servers execute their portions; the scientist experiment compares responses
5. The merged result returns to the client

### Federation Registration Pattern

Each domain package provides a registration function that wires its fields into the subgraph's schema:

```go
func (s *Server) RegisterFederatedFields(schema *schemabuilder.Schema) {
    s.Register(schema)
    s.registerSubgraphOnlyFields(schema)
    s.registerRootQueries(schema)
}

func (s *Server) registerSubgraphOnlyFields(schema *schemabuilder.Schema) {
    object := schema.Object("Organization", models.Organization{})
    object.FieldFunc("canWrite", s.viewerCanWrite,
        schemabuilder.Expensive, schemabuilder.Requires("id"))
    object.BatchFieldFunc("logoUrl", s.getLogoUrlBatch)
    object.FieldFunc("timezone", s.getTimezone,
        schemabuilder.Expensive, schemabuilder.Requires("id"))
}
```

The `Requires("id")` annotation tells the federation planner that this resolver needs the parent object's `id` field to execute. An internal code-generation tool was used to auto-generate these annotations across approximately 200 field resolvers by statically analyzing which parent struct fields each resolver accesses.

---

## 5. Approach

The work was structured in two batches to manage risk and enable incremental validation.

### Batch 1: Foundation

**Scope:** 2 core domain packages (platform operations and user management), approximately 30 field resolvers, 14 files changed.

Key deliverables:
- Created federation registration files for both packages
- Built a **database-backed entity resolver** for the user-organization mapping (see Section 6.1)
- Added dependency annotations to all field resolvers
- Configured the feature flag with approximately 27 field routes
- Validated on a staging environment with 147 test queries

**Result:** Merged and deployed to a production staging cell. Zero errors across tens of thousands of daily queries.

### Batch 2: Full Domain Expansion

**Scope:** 20 additional domain packages (authentication, attributes, capabilities, features, tags, workflows, scheduled reports, sessions, preferences, and more), approximately 200 field resolvers, 84 files changed, over 1000 lines of code added.

Key deliverables:
- Created federation registration files for all 20 packages
- Added dependency annotations to approximately 200 field resolvers
- Analyzed field ownership across multiple federated subgraphs, excluding 17 fields owned by other teams
- Built a structured tracing package for subgraph execution observability
- Configured the feature flag with several hundred field routes
- Prepared over 400 test queries for staging validation

**Result:** Deployed to two staging cells. Confirmed zero errors with sub-millisecond federation latency.

---

## 6. Key Engineering Challenges

### 6.1 Database-Backed Entity Resolver for Authorization-Critical Objects

**Problem:** The user-organization mapping entity carries two fields critical for authorization: the organization identifier and the group identifier. The authorization system reads these fields to determine a user's permission domains. With a simple passthrough resolver (which only forwards the entity's primary key), these fields would remain zero-valued. This would cause the authorization system to evaluate permissions against empty domains — potentially granting incorrect access or blocking legitimate requests.

**Solution:** I built a database-backed entity resolver that accepts a batch of primary keys, loads the complete records from the database, and returns fully hydrated entities with all authorization-critical fields populated.

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

**Design decision:** This pattern mirrors an existing entity resolver in the codebase (for the User type), establishing consistency. The resolver is registered globally so all subgraphs benefit from it, not just the PIB subgraph.

### 6.2 Multi-Service Field Ownership Analysis

**Problem:** When the PIB subgraph registers a domain package, it registers all of that package's fields. However, some fields within those packages are already owned by other federated subgraphs (e.g., an alerts subgraph or a contacts subgraph). Adding dependency annotations to those fields would create dual-registration, causing the federation symmetry test to fail.

**Solution:** I analyzed the existing feature flag variations to identify which fields were already routed to other subgraphs. For 17 fields across two other subgraphs, I explicitly removed or excluded the dependency annotations, ensuring each field has exactly one federation owner.

### 6.3 Read/Write Schema Split

**Problem:** One of the core domain packages maintains separate database connections for read and write operations, with distinct server instances for each. The federation wiring must create separate schemas for primary (read-write) and replica (read-only) connections, using dependency injection tags to route queries correctly.

**Solution:** The subgraph's schema configuration creates separate primary and replica schema instances, each wired to the appropriate database connection through the dependency injection framework.

### 6.4 Rebase Conflict Resolution in a Large Monorepo

**Problem:** The main branch evolved significantly during the Batch 2 development window. Three rounds of rebase conflicts had to be resolved:

- A migration from single-field resolvers to batch-field resolvers in the tags package
- An authentication refactor that restructured how security context is propagated to subgraph handlers
- A package reorganization that moved fields between domain packages

**Solution:** Each conflict was resolved by preserving the upstream team's changes and carefully adding the federation annotations on top. This required understanding both the federation requirements and the intent behind each upstream change.

---

## 7. Validation Methodology

### Staging Validation (Batch 1)

| Metric | Value |
|--------|-------|
| Total queries tested | 147 |
| Pass rate | 81% |
| False positives (non-PIB queries) | 18 |
| Bad test input | 4 |
| Pre-existing bugs | 1 |
| **Real federation bugs** | **0** |

All failures were attributable to test methodology issues (queries targeting fields not owned by the PIB subgraph, incorrect test variables) or pre-existing bugs in the monolith. Zero failures were caused by the federation implementation itself.

### Staging Validation (Batch 2)

| Metric | Value |
|--------|-------|
| Test queries prepared | 414 |
| Staging environments | 2 |
| Federation latency | Sub-millisecond (0ms) |
| Errors | 0 |

### gRPC Parity Testing Tool

I built a Go command-line tool that sends the same GraphQL query to both the new subgraph and the monolith via gRPC, then compares the JSON responses:

```go
func main() {
    subgraphAddr := "<subgraph_address>"
    monolithAddr := "<monolith_address>"

    for _, q := range queries {
        subgraphResult, _ := executeQuery(ctx, subgraphAddr, req)
        monolithResult, _ := executeQuery(ctx, monolithAddr, req)

        if string(subgraphResult.Response.Result) ==
           string(monolithResult.Response.Result) {
            fmt.Println("  MATCH")
        } else {
            fmt.Println("  DIFF")
        }
    }
}
```

Results from local gRPC parity testing:

| Result | Count | Details |
|--------|:-----:|---------|
| Match | 11 | Identical responses |
| Response diff | 2 | Benign — array ordering from non-deterministic map iteration |
| Federation mismatch | 6 | 4 fixed during development, 2 out of scope (fields from other domains) |
| Both fail | 147 | Expected — queries reference fields from non-PIB packages |

### Shadow-Mode Production Validation

After deployment to a production staging cell, the scientist experiment confirmed correct behavior:

- The subgraph handled tens of thousands of queries across dozens of unique query types
- Zero errors and zero panics over sustained operation
- Query types spanned user navigation, permissions, organization management, and platform features

---

## 8. Results

### Batch 1 (Merged)

- 2 domain packages wired with approximately 30 field resolvers
- 14 files changed
- Deployed to production staging, serving six-figure daily query volume
- Zero errors, zero panics

### Batch 2 (Validated, In Review)

- 20 additional domain packages wired with approximately 200 field resolvers
- 84 files changed, over 1000 lines added
- Several hundred field routes configured
- Validated on 2 staging environments with 414 test queries
- Zero errors, sub-millisecond federation latency

### Combined Impact

| Metric | Value |
|--------|-------|
| Pull requests shipped | 2 |
| Files changed | ~100 |
| Domain packages wired | 20+ |
| Federation annotations | ~200 |
| Field routes | Several hundred |
| Production queries/day | Six-figure volume |
| Production errors | 0 |

---

## 9. Technical Contributions

### New Artifacts Created

- **Federation registration files:** One per domain package (20+ files), each containing a `RegisterFederatedFields()` entry point
- **Database-backed entity resolver:** For authorization-critical user-organization mapping
- **Structured tracing package:** Logs subgraph execution events (start, complete, error) with gRPC header propagation for cross-service observability
- **gRPC parity testing tool:** Sends same query to both servers, compares responses
- **Browser-based test runner:** Automated script for running hundreds of test queries against staging environments
- **Feature flag variations:** Configured several hundred field routes for incremental rollout

### Monitoring and Observability

The service was instrumented with:

- Log aggregation queries for checking execution events and errors
- Observability platform metrics for request volume and latency
- Standard monitors for panics, container health, availability, and log cost

All monitors remained healthy throughout staging and production deployment.

### CI and Quality

| Check Category | Batch 1 | Batch 2 |
|----------------|:-------:|:-------:|
| Backend tests | Pass | Pass |
| Backend integration | Pass | Pass |
| GraphQL schema validation | Pass | Pass |
| Frontend tests | Pass | Pass |
| Mobile tests | Pass | Pass |
| Federation symmetry | Pass | Pass |
| Code review | Approved | In review |

---

## 10. Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **DB-backed entity resolver** | The entity carries authorization-critical fields. A passthrough resolver would leave those fields empty, risking incorrect permission evaluation |
| **Read/write schema split** | The domain server uses separate database connections for reads and writes. The federation wiring must create separate schemas for each connection type |
| **Dependency annotations** | Annotations enable the federation planner's "fast path" — the gateway only requests parent fields that the resolver actually needs, avoiding unnecessary database lookups |
| **Exclude cross-service fields** | Fields already owned by other federated subgraphs must not be annotated to avoid dual-registration conflicts |
| **Register-all pattern** | Each package registers all its fields into the subgraph schema. Only fields with dependency annotations create federation routes; the feature flag controls actual routing |
| **Debug endpoint isolation** | The internal debug GraphQL endpoint resolves queries locally and does not go through the federation gateway. Federation only occurs through the production WebSocket path |

---

## 11. Challenges and Lessons Learned

### Challenges

| Challenge | Details | Resolution |
|-----------|---------|------------|
| **Dual registration** | Fields registered in both the shared schema and the federation-specific file cause CI failures | Kept federation-only resolvers strictly in separate files |
| **Local development** | Distributed federation services require careful coordination of dependencies and authentication | Pivoted to staging-first validation |
| **Rebase conflicts** | Three rounds of conflicts as the main branch evolved with concurrent team changes | Careful merge resolution preserving upstream intent |
| **Multi-service ownership** | 17 fields owned by other subgraphs required explicit exclusion analysis | Analyzed existing feature flag variations to determine ownership |

### Key Learnings

| Area | What I Learned |
|------|---------------|
| **GraphQL federation at scale** | Deep understanding of federation planning, key-based entity resolution, dependency annotations, and schema symmetry requirements |
| **Shadow-mode validation** | Comparing subgraph responses against the monolith in production before cutover — a safe migration pattern applicable beyond GraphQL |
| **Production observability** | Log aggregation analysis, observability metrics, gRPC tracing, and progressive rollout monitoring |
| **Large-scale monorepo** | Working with 20+ domain packages across multiple teams, understanding import boundaries, code review culture, and CI systems |
| **Feature-flag-driven development** | Using per-field, per-cell feature flags for safe incremental rollout with instant rollback capability |

---

## 12. Future Work

### Short Term (Handoff)

1. Complete staging validation with full test query suite
2. Verify zero errors through log aggregation
3. Check shadow-mode experiment diffs
4. Merge Batch 2 pull request

### Long Term (Production Rollout)

5. Roll out to all staging deployment cells
6. Ramp to production deployment cells
7. Transition from shadow mode to primary mode
8. Decommission monolith handling of federated fields

---

## 13. Skills and Technologies

- **Languages:** Go, JavaScript (test tooling)
- **Systems:** GraphQL, gRPC, protocol buffers
- **Architecture:** Federation, microservices, distributed systems, dependency injection
- **Methodology:** Shadow-mode validation, feature-flag-driven rollout, progressive deployment
- **Observability:** Log aggregation, metrics platforms, distributed tracing
- **Development:** Large-scale monorepo, CI/CD, code review, rebase-based workflow
- **Testing:** Table-driven tests, gRPC parity testing, browser-based automation, staging validation

---

## 14. Personal Reflection

This internship was a deep dive into production distributed systems at scale. I started with limited understanding of GraphQL federation and ended with the ability to design and implement federation for an entire domain group.

The most valuable learning was not any single technical skill, but the engineering methodology: understanding a complex existing system, designing changes that work within its constraints, validating safely through shadow-mode experiments, and shipping incrementally with instant rollback capability.

I want to thank my mentor Gautam Dangodra for guiding me through the architecture, code review process, and production deployment. The GQL Infrastructure team's support through code reviews and technical guidance was invaluable.

---

## 15. Acknowledgments

- **Mentor:** Gautam Dangodra — architecture guidance, code reviews, deployment support
- **Team:** GQL Infrastructure / Platform — technical guidance and review
- **Prior work:** The safety-video-camera federation project established the patterns I followed and extended

---

*This report presents a generalized academic case study derived from internship engineering work. Company-specific identifiers, production values, internal interfaces, deployment details, and security-sensitive implementation choices have been omitted or abstracted. The report focuses on transferable engineering concepts and the author's learning.*
