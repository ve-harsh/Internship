# Incident Reconstructor — Final Build Writeup (Intern Hackathon)

**Author:** Harsh Verma
**Event:** Samsara Intern Hackathon, July 2026

> Company-specific identifiers and operational details have been generalized for academic discussion. External distribution remains subject to the company's review and approval process.

---

## 1. The Problem

Every time a fleet vehicle is involved in a serious event — a collision, a hard brake, a near-miss — the raw evidence already exists: the dashcam clip, the GPS trail, the speed and g-force telemetry, and the company's own event-detection model findings. The catch is that each of these lives in a separate place.

Today a safety or operations person has to manually stitch them together and hand-write a summary. That's roughly an hour of work per event, often under a deadline for an insurance claim or a driver dispute — and the output is inconsistent from person to person.

---

## 2. The Idea — One Click

Incident Reconstructor collapses that hour into one click. It adds a "Reconstruct incident" button directly onto the existing safety event detail page. Clicking it assembles every signal onto a single, synced view:

- The dashcam clip
- A GPS mini-map of the approach
- A speed / g-force chart marking the exact moment of the hard brake and impact
- The platform's trained-model insights (detected behaviors)
- A neutral, factual "what happened" narrative

A second click downloads the whole thing as a shareable, defensible evidence packet (PDF) — ready for an insurer, an internal review, or a coaching conversation.

**Impact:** an hour of manual, inconsistent report-writing becomes ~45 seconds of one-click, consistent, evidence-grade output.

---

## 3. Where It Lives in the Product

The feature attaches to the aggregated safety inbox event detail page — the screen a reviewer is already on when triaging an event. No new navigation, no new page: the button appears in the existing detail toolbar next to actions the user already knows.

```
Safety Inbox -> open an event -> detail toolbar -> [ Reconstruct incident ]
                                                        |
                                                        v
                                     +-------------------------------+
                                     |  Incident reconstruction modal |
                                     |  - Clip (dashcam)              |
                                     |  - Speed over time chart       |
                                     |  - GPS route mini-map          |
                                     |  - Key metrics tiles           |
                                     |  - Model insights              |
                                     |  - "What happened" narrative   |
                                     |  [ Download report ] (-> PDF)  |
                                     +-------------------------------+
```

---

## 4. Architecture at a Glance

The reconstruction is **assembled on the frontend from data already on the page**, and the *only* backend call is for the written narrative. This keeps the timeline perfectly in sync with what the reviewer sees and avoids a second, brittle data fetch.

```
          Frontend (React/TypeScript)
  selected event + trip
        |
        v
  buildTimeline() -> IncidentTimeline (speed, clip, route, metrics)
        |-> SpeedChart (Recharts)
        |-> MiniMap (dependency-free SVG)
        |-> Key-metric tiles + Model insights
        |
  buildReconstructionFacts() -> factsJson --GraphQL--> Backend

          Backend (Go)
  reconstructIncident(orgId, eventUuid, factsJson)
    1. Feature flag check (default OFF -> PermissionDenied)
    2. Authorize the specific event (read permission)
    3. generateIncidentNarrative():
         LLM (structured output, temperature 0)
           -- on any failure --> deterministic fallback
```

---

## 5. What We Built

### Frontend Components

| Component | What It Does |
|-----------|-------------|
| **Reconstructor button + modal** | Feature-flag-gated button and modal wrapper. Gates on the backend flag query and on usable speed/video data. Lazily fetches the narrative on first open |
| **Modal content** | Modal body: bordered section cards (Clip, Speed over time, Route, Key metrics, Model insights, What happened), icon stat tiles, and Download report footer. Built entirely with the design system |
| **Timeline builder** | Pure helper function — derives the incident timeline from the selected event and trip location/speed data, builds the structured fact set for the narrative |
| **Speed chart** | Speed-over-time chart with a speed-limit reference line, marking the exact event moment |
| **GPS mini-map** | Dependency-free SVG mini-map with a color-matched Start / Event / End legend |
| **Printable report** | Self-contained printable HTML report (Download report -> browser print -> PDF); escapes all interpolated values |
| **GraphQL operations** | Feature flag gating query and reconstruction mutation |

### Backend

| Component | What It Does |
|-----------|-------------|
| **Incident reconstructor** | Registers the query + mutation, enforces the feature flag and per-event authorization, generates the narrative via LLM structured output with a deterministic fallback and observability metrics |
| **Unit tests** | Tests for narrative generation and helper functions |
| **Feature flag** | Backend-only flag with generated accessor |

### Translations

36 translation strings covering every label, metric, legend, disclaimer, and error message — each with translator notes for localization.

### Tests

- Feature flag gating tests
- Fact derivation / timeline builder tests
- Report HTML generation and escaping tests
- Narrative generation unit tests

---

## 6. Interesting Engineering Decisions

### a) Facts computed on the frontend, passed to the backend

The mutation takes a `factsJson` payload built by the same `buildTimeline()` code that renders the modal, so the narrative is generated from exactly the numbers the reviewer sees. No drift, no second server-side data fetch.

### b) The narrative never fails — LLM with a deterministic fallback

The narrative generator tries the LLM path first (structured output, temperature 0, 30-second budget). If the LLM accessor isn't wired, the API key is missing, or the call fails or returns empty, it degrades to a **deterministic narrative** built from the same facts — and records why in a `SkipReason` field. Because the user explicitly clicked "Reconstruct," the mutation must always return something usable. The deterministic path doubles as the offline demo path.

### c) A strictly neutral, no-blame narrative

The system prompt is locked down: use only the provided facts, 2–4 past-tense sentences, refer to "the vehicle" (never a named driver), no blame, no speculation, no recommendations, omit any uncaptured metric. This is what makes the output evidence-grade.

### d) Gating a backend-only flag from the browser

The feature flag is backend-only (not exposed to the browser SDK). So we added a read-only GraphQL query that evaluates the same flag server-side — the button and the mutation can never disagree, and the mutation re-checks the flag as defense in depth.

### e) Safe production rollout with @indev

Both GraphQL operations are marked with an `@indev` directive, so shared client middleware strips them from the production bundle until the backend fields deploy to every GraphQL cell. A stripped field reads back as undefined, keeping the action hidden by default.

### f) Authorization that matches the detail page exactly

The mutation loads the event scoped to the caller's organization (closing a cross-org IDOR) and runs the same read permission check the inbox read paths use, with the widest visibility filters. The real access boundary is the permission check, not the visibility filter.

---

## 7. The Review Journey (Why 14 Commits)

| Slice | What Changed |
|-------|-------------|
| **Scaffold** | Feature flag + `buildTimeline()` + speed chart + translation keys |
| **Full feature** | Modal, mutation, deterministic-fallback narrative, printable report |
| **Authz fixes** | Closed a cross-org IDOR by authorizing against the specific event; fixed a permanently-stuck spinner when a narrative request was abandoned |
| **CI fixes** | TypeScript prop typing, JSX lint rules, `@indev` on the mutation |
| **Stale-narrative bug** | Keyed the lazy request by the facts payload (not just event UUID) so richer trip/speed data triggers a fresh narrative; fixed a duplicate-mutation bug |
| **Real-data fixes** | Replaced raw video tag with the platform's signed-URL video player; capped "time to stop" to the post-event window; treated speed limit of 0 as unknown; preferred IMU peak deceleration to match data exports |
| **UI restyle** | Bordered section cards, icon stat tiles, GPS legend, download moved into modal footer, inline styles replaced with design system components |
| **Final authz** | Authorized the gating query itself (failing closed) to satisfy the authorization linter after rebase |

---

## 8. What "Done" Looked Like

For a 3-day build, the goal was to ship the real button + reconstruction panel on the live safety event page, backed by data from one real event, with the timeline, AI narrative, and PDF export all fully working — not a mockup. That's what landed:

- Real "Reconstruct incident" button on the actual safety event detail toolbar
- Synced timeline: clip + GPS mini-map + speed/g-force chart + key metrics + model insights
- AI narrative (LLM) with a deterministic fallback so the demo never breaks
- One-click PDF evidence packet (self-contained printable report)
- Feature-flag-gated, authorized, tested, and green in CI

---

## 9. What We'd Do Next

1. Promote the backend-only flag to a client-side rollout and remove the `@indev` markers once deployed to all GraphQL cells
2. Move narrative generation async (the current sync mutation caps at 30s) for very large events
3. Persist the generated packet so the same evidence PDF can be re-downloaded without re-running the LLM
4. Wire a dedicated production LLM key instead of a shared development key
5. Expand beyond the safety inbox to any surface with a safety event (e.g., driver dispute flows)

---

## 10. Skills and Technologies

- **Frontend:** React, TypeScript, Recharts, SVG rendering, design system components
- **Backend:** Go, GraphQL mutations, feature flag gating, authorization checks
- **AI/LLM:** Structured output generation, deterministic fallback, neutral prompt engineering
- **Security:** Cross-org IDOR prevention, defense-in-depth flag checking, authorization linting
- **Testing:** Unit tests for gating, fact derivation, HTML escaping, narrative generation
- **Product:** Evidence-grade output design, one-click workflow, printable PDF generation

---

*This report documents a hackathon project built during an internship. Internal file paths, service names, and implementation-specific details have been generalized where appropriate. The report focuses on engineering decisions, architecture, and the author's contributions.*
