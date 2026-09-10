# ADR-0001: Go for the API service, Python for the orbit and planner workers

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-09-10 |
| **Deciders** | Project owner |
| **Supersedes** | — |
| **Related** | ADR-0003 (JetStream and outbox), ADR-0005 (CP-SAT planner) |

## Context

The system has three backend components with different workloads:

- **API service** — request intake, validation, state rollup, read models, outbox relay. I/O-bound, long-running, many small concurrent operations against PostgreSQL and NATS. Must be cheap to run and simple to operate.
- **Access worker** — TLE ingest, SGP4 orbit propagation, footprint/target geometry. CPU-bound numerical work over time series, dependent on well-tested orbital-mechanics libraries.
- **Planner worker** — constraint scheduling over access windows. Dependent on a constraint-programming solver; correctness of the formulation matters more than throughput.

A single language for all three would be simpler to build and hire for. The question is whether any one language serves all three workloads well enough that the simplicity is worth it.

## Decision drivers

1. **Library maturity in the domain.** Orbit propagation and constraint solving are areas where a wrong or home-grown implementation is a correctness risk, not a performance risk. The choice should follow where the trusted, actively maintained libraries are.
2. **Operational fit for a long-running service.** The API is the always-on component: memory footprint, startup time, concurrency model, and static-binary deployment matter.
3. **Clear service boundaries.** The components already communicate only through an event contract and a shared database schema (ADR-0003). A language boundary that coincides with a service boundary adds little integration cost.
4. **Testability of numerical code.** Orbital and geometric computations must be checked against reference values; the language should make that easy.
5. **Team and ecosystem reality.** In ground-segment and planning teams, Python is the working language for anything touching orbital mechanics, geospatial analysis, and optimisation; Go and TypeScript dominate the service layer. The stack should look like one those teams would recognise and maintain.
6. **Scope discipline.** This is a small system with a fixed time budget. Every choice should reduce the amount of code written from scratch.

## Considered options

### Option A — Go everywhere

- API: natural fit.
- Orbit propagation: [`go-satellite`](https://github.com/joshuaferrara/go-satellite) implements SGP4 but is a port with a small user base and no ecosystem around it (no ephemeris/time-scale handling, no geodesy, no reference test suite comparable to Skyfield's). Geodesic polygon intersection would require pulling in or writing geometry code with no PostGIS-equivalent library in Go.
- Constraint solving: OR-Tools has no first-class Go API. Options are cgo bindings (fragile builds, awkward in containers), calling a Python or C++ sidecar (which reintroduces a second runtime anyway), or writing a bespoke scheduler. A bespoke scheduler is exactly the code this project should not be writing from scratch.

**Verdict:** the API is well served, but the two workers would each need either weak libraries or a second runtime in disguise.

### Option B — Python everywhere

- Orbit propagation: [Skyfield](https://rhodesmill.org/skyfield/) plus the reference [`sgp4`](https://pypi.org/project/sgp4/) package are the de facto standard, with published accuracy tests against the official SGP4 test vectors. Shapely and pyproj cover geodesic geometry.
- Constraint solving: OR-Tools' primary API is Python; CP-SAT examples, documentation, and community knowledge are overwhelmingly Python.
- API: FastAPI is capable, but a Python API service is heavier to run (interpreter start-up, GIL, larger images), needs more care for concurrency, and is a less common choice for the always-on service tier in infrastructure-oriented teams. The outbox relay and rollup logic are exactly the kind of plain concurrent I/O where Go's model is a better fit.

**Verdict:** both workers are well served; the API is served adequately but not well.

### Option C — Go for the API, Python for the workers (chosen)

Each component uses the language where its critical dependencies are strongest. The cost is two toolchains, two sets of container images, and two idioms for the same event-consumer pattern.

### Option D — TypeScript/Node for the API, Python for the workers

Equally reasonable for the API tier and a common pairing in practice. Not chosen because the project owner's depth is in Go, and because the front-end will already be TypeScript, so keeping the API in Go maintains a clear three-way split (UI / service / numerical) rather than two components sharing a language for incidental reasons. This would be a low-cost decision to revisit if the team composition changed.

## Decision

- **Go** for the `api` service.
- **Python** for `workers/access` (Skyfield, `sgp4`, Shapely, pyproj) and `workers/planner` (OR-Tools CP-SAT).
- No shared code across the language boundary. Integration is through:
  - the PostgreSQL schema, owned by `api/migrations`, and
  - the JetStream event contract, documented in the README and versioned in the event envelope.
- The event-consumer pattern (durable pull consumer, explicit ack after commit, `processed_events` idempotency check) is implemented once per language and covered by the same integration tests, so the two implementations are held to identical behaviour.
- Each Python worker ships as its own container with a pinned dependency lockfile; no worker imports from the other.

## Consequences

**Positive**

- Orbit propagation and geometry use libraries with reference test suites; the project validates against them rather than re-deriving them.
- The planner uses CP-SAT natively, keeping the model readable and the formulation easy to change.
- The API is a small static binary with low idle cost and a straightforward concurrency story for the outbox relay.
- The architecture mirrors what ground-segment teams actually run, which makes the design decisions easier to explain and defend.

**Negative / accepted costs**

- Two toolchains in CI, two base images, two dependency-management systems (Go modules, `uv`/`pip` lockfile).
- The consumer/idempotency pattern exists twice. Mitigated by shared integration tests and by keeping that code deliberately boring.
- Cross-cutting changes (a new event type, a schema change) touch both languages. Mitigated by the schema living in one place and the event envelope carrying a version.
- Developers must be comfortable in both languages to work end to end. Accepted: the boundary is stable and each side is small.

**Neutral**

- If the API tier were later moved to TypeScript to align with a front-end-heavy team, nothing on the worker side changes. The boundary is the contract, not the language.

## Compliance

- New backend components must state in their ADR which side of the boundary they sit on and why.
- Any proposal to add a third backend language, or to share code across the boundary, requires a new ADR that supersedes this one.

## References

- Skyfield: https://rhodesmill.org/skyfield/
- `sgp4` reference implementation and test vectors: https://pypi.org/project/sgp4/
- OR-Tools CP-SAT: https://developers.google.com/optimization/cp/cp_solver
- Shapely: https://shapely.readthedocs.io/
- Michael Nygard, *Documenting Architecture Decisions*: https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- MADR template: https://adr.github.io/madr/
