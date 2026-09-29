# Pact Broker MkII — Prototype Project Plan

This plan sequences the work to prototype the design in [broker-mkii-design.md](broker-mkii-design.md),
which answers the problems listed in [broker-mkii.md](broker-mkii.md). It follows the shape of the
[Janus prototype plan](../../../pact-janus/Documentation/project-plan.md), because that worked: design and
exploration tasks sit alongside implementation, and the point of the prototype is to **produce evidence
for the design's bets and open questions**, not just code.

The prototype lives in its own repository (working name `pact-broker-mkii`, a sibling of `pact_broker`
and `pact-janus`; naming is task 0.1). This repository keeps the design documents and lends its feature
specs, fixtures and client tooling as the conformance corpus for the facade (Phase 8).

Task types: **[design]** produces a spec/ADR for review · **[spike]** time-boxed experiment, throw-away
code allowed, produces a written finding · **[explore]** survey/investigation, produces a report ·
**[build]** production-intent prototype code · **[gate]** a decision point that later phases depend on.

---

## 1. What the prototype must prove

The design (§16) makes eight bets. Each phase below exists to test one or more of them:

| # | Bet | Falsifiable claim to test |
|---|-----|---------------------------|
| B1 | Party/role/publication/fact is sufficient | Pact (HTTP + messages), OAS provider contracts with BDCT comparison, Janus contracts and provider shapes can all be modelled **without a module adding a table or a top-level route**. |
| B2 | Blobs out of the index fixes the publish path | A 256 MB pact publishes in under the load-balancer timeout (60 s) on a modest instance; a day of republishing an unchanged 50 MB pact adds zero blob bytes. |
| B3 | Decision state makes can-i-deploy a read | On a dataset the size of PactFlow production, `can-i-deploy --to-environment` completes in tens of milliseconds and does not scale with history. |
| B4 | Retention policy keeps the hot set bounded | Query latency after simulating a year of CI publishing under the default policy is flat. |
| B5 | Coalesced events tame the herd | A burst of N consumer publishes produces one provider build trigger; a pull client sees the same information with no inbound connectivity. |
| B6 | The facade is good enough | The existing `pact-broker` CLI, `pact_broker-client` gem, and at least two Pact SDK verifiers run their full broker-related test suites against MkII unchanged. |
| B7 | Import is lossless where it matters | An existing broker imported into MkII returns identical `can-i-deploy` answers for a recorded corpus of real queries. |
| B8 | Scope is sufficient for Teams + RBAC | Team-level visibility and permissions can be implemented as an access module over design §4.6 with no change to core tables, and the per-request filter adds no measurable latency to B3's read. |

**Prototype non-goals** (out of scope; design-only where the design needs an answer): production hardening
and API stability; a full UI (a minimal read-only UI at most, Phase 8); every blob and index backend
(filesystem + S3-compatible; SQLite + Postgres + one more SQL engine to prove F7); the complete facade
surface (the resources the B6 client suites exercise, not every route in the current broker); badges,
dashboards and the integrations graph; PactFlow's actual RBAC product (a reference Teams module only);
a hosted deployment; migration timelines, naming and governance decisions (framed for the community,
not decided here).

---

## 2. Phase overview and sequencing

```
Phase 0  Foundations ──┐
Phase 1  De-risking spikes ──► G1 (language, index store, blob store)
                                │
Phase 2  Core design round ◄────┘   (overlaps with late Phase 1)
                │
Phase 3  Core kernel: index, blobs, modules, workers ──► M1
                │
Phase 4  Pact module + facade vertical slice ──► M2
                │
Phase 5  Decision engine + import ──► M3 ──┐
                │                          │
Phase 6  Lifecycle and events ──► M4       │
                │                          │
Phase 7  Modules proof (BDCT, Janus, Teams, 2nd engine) ──► M5
                │                          │
Phase 8  Facade conformance + clients ──► M6 ◄──┘
                │
Phase 9  Evaluation and design feedback
```

Phases 1 and 2 overlap (spike results feed the designs). Phase 5's importer is what makes Phases 6, 7
and 9 measurable at production scale, so it is on the critical path. Phase 7's type modules depend only
on the module interface (Phase 3) and can start as soon as M1 lands; Phase 7's Teams module depends on
Phase 5 (its cost is measured on the can-i-deploy read). Phase 8 runs its client suites continuously from
M2 on and *closes* once they all pass. Durations are deliberately omitted; each phase lists an indicative
size (S/M/L relative to the others) as a sequencing aid, not a commitment.

Milestones (each is demo-able):

- **M1** — A multi-document publication is staged as blobs and completed with a manifest through the new
  HAL API, on SQLite + filesystem and on Postgres + MinIO, with the same binary; the index resource
  advertises the registered types.
- **M2** — `pact-broker publish` and a Pact SDK verifier (fetch pact, publish verification result) work
  unchanged against MkII through the facade; a 256 MB pact publishes in under 60 s.
- **M3** — A real broker database is imported; `can-i-deploy --to-environment` through the facade returns
  the same answers as the source broker for a recorded query corpus, served from decision state.
- **M4** — A year of simulated CI publishing runs under the default retention policy with flat query
  latency; a burst of consumer publishes produces one coalesced webhook delivery and one pull-client
  observation.
- **M5** — An OAS provider contract, a Janus contract and a Janus provider shape are published and compared
  through type and comparator modules; a Teams access module hides another team's rows; the same suite
  passes on a third SQL engine.
- **M6** — The B6 client suites all pass against MkII; the Pact CLI uses the new links where present and the
  facade links where not.

---

## 3. Phase 0 — Project foundations (size: S)

Goal: a place to put decisions and code, and an honest inventory of what can be reused.

- **0.1 [design] Prototype charter.** One page: the success criteria (§1, refined), the non-goals, what
  "prototype complete" means, and the working name. The yardstick for every later scope argument.
- **0.2 [design] Decision log structure.** `Documentation/decisions/` as an ADR log (`NNNN-title.md`,
  status proposed/accepted/superseded). Every [gate] and every contested choice lands here. The design's
  open questions (§16) seed the index (see §13 traceability).
- **0.3 [build] Repo scaffolding.** Rust workspace (`core/`, `modules/`, `facade/`, `server/`, `cli/`,
  `importer/`, `spikes/`, `corpora/`), CI (build, test, clippy, fmt, a Postgres + MinIO service matrix),
  contributor docs. Spike code lives in `spikes/` and may rot; everything else stays green. Rust is the
  §13 recommendation and is revisited at G1, so the scaffolding stays thin until then.
- **0.4 [explore] Reuse inventory.** Assess *reuse as dependency / fork and adapt / rewrite / reference
  only* for: the quilt prototype's repository layer (`quilt-cli/src/repository/` — blob layout, index
  manifests, per-document versioning); `webmachine-rust` (HAL resource dispatch); `pact_models` (pact
  parsing for the pact type module's analyse step, and the `content_that_affects_verification` hashing);
  the Janus engine as a WASM component (subsumption comparator); the current broker's `spec/features`,
  `spec/fixtures` and `spec/service_consumers` (facade conformance corpus); `pact_broker-client` and the
  `pact-broker` CLI test suites (B6); the current broker's `CleanIncremental` keep selectors (retention
  defaults); the webhook template language (kept per design §9.3). Output: a report per item with
  licensing and coupling notes.
- **0.5 [explore] Production shape of the data.** Collect, from PactFlow operators, the anonymised
  statistics the benchmarks need: parties, versions and publications per party, pact size distribution,
  publishes per day, verification counts per pair, branch counts, environments, deployed-version churn,
  matrix query rates and latencies. Output: a dataset profile that 1.4's generator and 9.1's report are
  sized against. No customer data leaves PactFlow; only the profile does.

---

## 4. Phase 1 — De-risking spikes (size: M, parallelisable)

Goal: settle the storage and language decisions everything else sits on, with evidence, before any
architecture ossifies. Each spike is time-boxed and ends in a written finding even if the answer is "it
doesn't work". B2 and B3 come first because they are cheap to test and decide the storage choices.

- **1.1 [spike] Streaming publish path (B2).** A toy server that streams a request body to a blob store
  while hashing it, then accepts a manifest referencing the hash. Backends: filesystem and MinIO (S3 API).
  Measure: wall time for 50 MB and 256 MB uploads over a throttled link; memory ceiling; behaviour on a
  disconnect mid-stream (no partial blob visible); `HEAD /blobs/{hash}` short-circuit for a republish.
  Compare with the current broker publishing the same 256 MB file. Answers B2's first half and confirms
  the two-step publish is viable behind a 60 s load balancer.
- **1.2 [spike] Decision state read path, KV vs SQL (B3).** Load a synthetic dataset (from 1.4) into
  (a) SQLite and Postgres with the `decision_state` shape from design §7.2, (b) an embedded KV store
  (redb or RocksDB) with hand-built secondary indexes on `(left_version)` and `(right_version)`. Run the
  can-i-deploy read (resolve selectors → range read → in-memory collapse) at production-profile sizes and
  at 10× history. Measure p50/p99 and how latency moves with history. Also measure the write side: cost of
  the per-fact and per-publication decision-state updates. Answers the design's "KV vs SQL" open question
  and B3's shape before the schema is designed.
- **1.3 [spike] Multi-engine SQL feasibility (F7).** The dozen-table index schema expressed once and run
  through `sqlx` (or `sea-orm`) against SQLite, Postgres, MySQL and SQL Server in CI containers. Which
  features have to be avoided (upserts, returning clauses, JSON columns, advisory locks for the job
  queue)? Is a portable job-lease pattern possible? Output: the list of allowed SQL features and the cost
  of each engine. Decides whether "narrow storage interface" is a schema discipline or an abstraction
  layer.
- **1.4 [build] Dataset generator and query corpus.** A generator that emits a synthetic broker history
  matching 0.5's profile at 1×, 10× and "one year of CI" scale, in a neutral format both the spikes and
  the importer (5.5) can load; and a recorder that captures a corpus of real `can-i-deploy`/matrix queries
  from a broker's request log (or from the existing feature specs) with their answers. Kept for the life
  of the prototype: it is how B3, B4 and B7 are measured.
- **1.5 [spike] Event coalescing model (B5).** Paper design plus a toy implementation of the outbox and a
  webhook subscriber with coalescing keys, quiet periods and causation ids. Replay a synthetic burst
  (ten consumers publishing within a minute, each publish triggering a provider build which publishes
  a verification) and count deliveries with and without coalescing and loop detection. Confirms the model
  before Phase 2 designs the subscription resource around it.
- **1.6 [spike] Web layer and HAL rendering.** The same three resources (index, a publication, a
  paginated collection with cursor) built in webmachine-rust and in axum, with content negotiation,
  conditional requests and link omission driven by a `permitted?` hook. Output: which is less friction
  for HAL resources; the design says it is not a deciding factor, so this is short.
- **1.7 [spike] Janus engine as an in-broker comparator.** Load the Janus engine's WASM component under
  wasmtime from a Rust host and run one subsumption walk on a contract/shape pair from the Janus corpus,
  keyed by input hashes. Confirms the comparator module shape (design §10.1) and the cost of running the
  walk in a worker rather than in CI.
- **1.8 [gate] G1 — Decisions: language; index store baseline; blob store baseline; web layer.** ADRs for:
  Rust confirmed or revised (the design's §13 gate); SQL vs KV for the index and which engines the
  prototype carries; S3-compatible + filesystem for blobs; webmachine vs axum; the allowed-SQL-features
  list from 1.3. Phase 2 designs are written against these decisions.

---

## 5. Phase 2 — Core design round (size: L, mostly parallel, review-gated)

Goal: turn the design's sketches into reviewable specifications with worked examples, reviewed before
later phases build against them. These graduate into the executable specification as schemas, corpora
and conformance suites attach to them.

- **2.1 [design] Core model and index schema.** Parties, versions, coordinates (branch, tag, environment
  membership, deployed/released), publications, documents, roles, facts, scopes and scope memberships,
  decision state, outbox, jobs, retention marks. For each: identity, uniqueness, ordering, what is
  indexed and what is a blob reference. Includes the `analysis: pending | complete | failed` state on a
  publication and the immutability rules. Worked examples: an HTTP pact, a message pact, a verification
  fact, a provider shape with no consumer role. Output: schema (engine-neutral DDL within 1.3's allowed
  features), a JSON Schema per resource representation, and a glossary that settles the working words
  (the "naming" open question — decided here for the new API; the facade keeps the old words).
- **2.2 [design] Module interfaces.** The two kinds from design §10 and their rules. *Artifact modules*:
  `type` (role names, `analyse(blobs) → metadata + derived documents`, `expected_counterparts`,
  `content_hash_for_change_detection`, renderers, diff), `fact` kind (submission, verdict semantics,
  summary schema), `comparator` (two publications → a fact, cached by input hashes). *Infrastructure
  modules*: blob store, index store, authentication, access (`filter` and `permitted?`), notification
  sink. States the table rule (artifact modules add none; infrastructure modules own tables keyed by core
  ids and never query core tables) and how the index resource advertises types and fact kinds. Worked
  examples: the pact type module and the allow-all access module.
- **2.3 [design] Publish protocol.** `POST /blobs` streaming semantics and idempotency, `HEAD /blobs/{hash}`
  and its two scoping policies (design §5.4), the manifest resource, staged uploads across CI jobs,
  the grace period for unreferenced blobs, and the analysis lifecycle including what a client sees while
  analysis is pending and how the facade waits. Answers the "auto-complete after quiet period" and
  "where analyse runs for very large documents" open questions, with a recommendation for each.
- **2.4 [design] Decision engine specification.** Selector language (consumer version selectors, WIP,
  pending, `--to-environment`, `--to` branch) and its resolution to version ids; the `decision_state`
  maintenance rules for each write (new fact, new publication with `expected_counterparts`, coordinate
  changes); the read; the three-valued summary and the reason vocabulary carried over from today's
  matrix; policy applied on read with the current clock (ignore rules, exemptions with expiry); the
  rebuild job. Answers the "per-pair vs per-publication summary for many-to-many fact kinds" open
  question. Worked examples reproduce the answers in `spec/features/can_i_deploy_spec.rb` and
  `matrix_spec.rb` from the current broker.
- **2.5 [design] Retention policy specification.** Keep rules (selector + duration) at instance, party and
  branch-pattern levels; the defaults that mirror `CleanIncremental`; the marker and collector algorithms
  as bounded batches; the ordering (publications → facts → decision rows → blobs); the `retention` link
  on a version and the `dry-run` resource. States the invariant that deleting an expired version never
  changes a live version's answer.
- **2.6 [design] Events and subscriptions specification.** Event types and payload shapes (denormalised
  enough to filter without a second request), the cursor and retention window, `GET /events` with
  filters, long-poll and SSE as optional, the subscription resource (webhooks as the built-in sink) with
  coalescing key, quiet period, rate limits, budgets and causation-chain limits, and `verdict_changed` as
  derived from decision state. The webhook template language is specified against the event payload.
- **2.7 [design] HAL API specification.** Resource families from design §11, link relations (including the
  `pb:*` relations the facade must keep serving), cursor pagination everywhere, error shapes, content
  negotiation, and the link-omission rule for `permitted?`. Includes the access filter contract: every
  collection read passes through `filter`, and the test that proves a resource cannot forget it.
- **2.8 [design] Facade mapping.** For each current broker route the B6 client suites touch: the MkII
  reads and writes it maps to, the vocabulary translation (pacticipant/consumer/provider ↔
  party/roles), which facade behaviours are synchronous (one-shot publish waits for analysis) and
  which current behaviours are deliberately not reproduced. Scoped by what 0.4 found the client suites
  actually call.
- **2.9 [design] Import specification.** Mapping from the current broker's tables to MkII: pacticipants →
  parties, versions/branches/tags/environments/deployments/releases → coordinates,
  `pact_publications` + `pact_versions` → `pact` publications with blobs, `verifications` →
  `verification` facts with detail blobs, webhooks → subscriptions, then a decision-state rebuild.
  Idempotency key (party, version, content hash), incremental re-runs while the old broker is live, and
  the answer to the "unscoped entities" open question for imported data.
- **2.10 [design] Reference Teams + RBAC access module.** Its own tables (users, teams, memberships,
  permissions), team → scope mapping, `filter` and `permitted?` semantics, the unscoped-entity policy
  and cross-scope visibility switch (tenancy). Answers the "scope granularity" open question by checking
  the design's five scopable entity kinds against what a team boundary actually needs.

---

## 6. Phase 3 — Core kernel: index, blobs, modules, workers (size: L)

Goal: the seven concepts stored, retrievable and extensible, before any Pact-specific behaviour exists.

- **3.1 [build] Index store.** Schema from 2.1 with migrations, repositories for every core entity, the
  access filter hook wired into every collection query with the identity filter as default. Runs on
  SQLite and Postgres in CI from the first commit (the third engine is Phase 7).
- **3.2 [build] Blob store.** Filesystem and S3-compatible backends behind `put/get/exists/delete/size`,
  streaming with hash-on-write, atomic visibility (no partial blob is ever readable), and the in-index
  fallback for zero-config installs.
- **3.3 [build] Module registry.** Type, fact-kind, comparator and infrastructure module registration;
  the index resource advertises registered types and fact kinds; a trivial `blob` type module exists for
  tests so the core can be exercised with no real artifact type.
- **3.4 [build] Publish path.** `POST /blobs`, `HEAD /blobs/{hash}`, the manifest resource and the
  publication resource with `analysis` state; the worker picks up analysis jobs and calls the type
  module.
- **3.5 [build] Workers and outbox.** Jobs table with leases (portable per 1.3), an in-process worker
  pool, and the append-only event log written in the same transaction as each state change. No
  subscribers yet.
- **3.6 [build] HAL layer.** Index, parties, versions, coordinates, publications, documents, facts, with
  cursor pagination and `permitted?`-driven link omission, on the web layer chosen at G1.
- **3.7 [build] Core conformance corpus v0.** HTTP-level scenarios (publish, stage, complete, read,
  filter) run against both store combinations in CI, so every later change is checked against the
  contract in 2.7.
- **3.8 [explore] Core-boundary review.** After 3.1–3.6: has anything Pact-shaped leaked into the core?
  Did any module need a table or route? Findings feed B1 and Phase 7.

**Milestone M1** closes this phase.

---

## 7. Phase 4 — Pact module and facade vertical slice (size: M)

Goal: the first real artifact type and the first client that does not know MkII exists.

- **4.1 [build] `pact` type module.** Roles for HTTP and message pacts, `analyse` (parse with
  `pact_models`, extract interaction count, has-messages, spec version, interaction SHAs),
  `content_hash_for_change_detection` matching today's "content that affects verification" semantics,
  merge of staged per-suite documents into one logical pact at analyse time, HTML renderer and diff.
- **4.2 [build] `verification` fact kind.** Submission, verdict mapping, summary schema, detail as blob.
  `expected_counterparts` for pact ("verifications by the provider") so decision rows appear as
  *unknown* on publish.
- **4.3 [build] Facade, publish and fetch.** `/contracts/publish`, the legacy `PUT /pacts/...` publish,
  pact fetch by latest/branch/tag, provider "pacts for verification", verification result publish, and
  `/pacticipants` with the `pb:*` links, mapped per 2.8. One-shot publish waits for analysis.
- **4.4 [build] Large-publish demonstration (B2).** The 1.1 measurement repeated on the real binary:
  256 MB pact through the facade under 60 s; a day of unchanged 50 MB republishes adds zero blob bytes;
  staged publish across three CI jobs with one changed document stores one new blob.
- **4.5 [build] Client harness.** `pact-broker publish` (Ruby CLI) and one Pact SDK verifier (pact-reference
  or pact-jvm) run against MkII in CI. This is the start of the B6 suite that Phase 8 completes.
- **4.6 [explore] Facade friction report.** Every place the facade had to fake, approximate or block on
  the core. This is evidence for or against B6 and for the "two brokers for a while" drawback.

**Milestone M2** closes this phase.

---

## 8. Phase 5 — Decision engine and import (size: L)

Goal: B3 and B7 — can-i-deploy as a read, proven against real data.

- **5.1 [build] Decision state maintenance.** The per-write update rules from 2.4 inside the publish and
  fact transactions; the rebuild job that drops and recomputes the projection.
- **5.2 [build] Selector resolution and the read.** Consumer version selectors, WIP, pending,
  `--to-environment`, branch targets; the indexed range read; the in-memory collapse, three-valued summary
  and reason vocabulary.
- **5.3 [build] `/decisions` and facade `/matrix` + `can-i-deploy`.** The new resource and the old ones,
  producing the output shape the `can-i-deploy` CLI expects. Badges are out of scope.
- **5.4 [build] Policy on read.** Ignore rules and exemptions with expiry applied with the current clock;
  a cached report versus an uncached decision, per design §7.3.
- **5.5 [build] Importer.** Per 2.9: reads a current broker database (Postgres), writes MkII, rebuilds
  decision state, re-runnable and incremental. Verified first on the generator's output (1.4), then on a
  real broker database (the pact_broker example database or a volunteer's dump).
- **5.6 [build] Corpus replay (B7).** Replay the recorded query corpus from 1.4 against the source broker
  and against MkII after import; diff answers and reasons. Any divergence is a finding or a fix.
- **5.7 [build] Decision benchmark (B3).** The 1.2 measurement on the real binary at production profile
  and 10× history, with and without a scoped principal (prepares B8). Kept running in CI as a trend.

**Milestone M3** closes this phase.

---

## 9. Phase 6 — Lifecycle and events (size: M)

Goal: B4 and B5 — the two problems that grow with time rather than with data.

- **6.1 [build] Retention policies.** Policy resources, keep rules at three levels, defaults mirroring
  `CleanIncremental`, the `retention` link and the `dry-run` resource.
- **6.2 [build] Marker and collector.** Continuous bounded batches stamping `expires_at`, then deleting
  publications, facts, decision rows and unreferenced blobs in order; the unreferenced-blob grace period
  from 2.3.
- **6.3 [build] Year-of-CI simulation (B4).** The generator drives a year of publishing at production
  profile through the real API with retention running; query latency and store sizes are sampled over
  simulated time. Flat is the pass condition.
- **6.4 [build] Events pull API.** `GET /events?after=` with filters and per-party cursors; long-poll as
  an optimisation; a small pull-client example that answers "has anything happened to my integrations
  since my last build".
- **6.5 [build] Webhook subscriber.** Subscriptions with coalescing keys, quiet periods, rate limits,
  budgets and causation-chain limits; `verdict_changed` events; the template language reimplemented over
  event payloads; facade `/webhooks` mapped onto subscriptions for the B6 suites.
- **6.6 [build] Herd demonstration (B5).** The 1.5 burst scenario end to end: N consumer publishes, one
  provider build trigger, the loop cut by causation ids, and the pull client observing the same facts.

**Milestone M4** closes this phase.

---

## 10. Phase 7 — Modules proof (size: L, parallel tracks)

Goal: B1 and B8 — the extension model tested by the artifacts that broke the current one, and the access
model tested by the case that forced it. Each type module here is written against 2.2 and the docs only;
every point where reading core source was necessary is a docs/design finding.

- **7.1 [build] `oas` type module and BDCT comparator.** Provider contract publication with a single
  `provider` role; a comparator module that compares an OAS publication with a pact publication and
  produces a `bdct-compatibility` fact. The comparator can be a thin wrapper over an existing tool; the
  point is the module shape, not the comparison quality.
- **7.2 [build] `janus-contract` and `janus-provider-shape` type modules.** The shape publication with no
  consumer role (the case the current schema cannot express); the `subsumption` comparator running the
  Janus engine WASM component in a worker, keyed by input hashes, per 1.7 and the
  [broker integration notes](../../../pact-janus/Documentation/broker-integration-notes.md); decisions
  that read subsumption facts with exemptions applied on read.
- **7.3 [build] Message pact roles.** Expose `publisher`/`subscriber` for message pacts through the module
  and facade per the "roles for messages" decision from 2.1; check both vocabularies against the same
  stored roles.
- **7.4 [build] Teams + RBAC access module (B8).** Per 2.10: own tables, `filter` and `permitted?`,
  link omission in HAL, row hiding across every collection including decisions and events; the scoped
  variant of the `HEAD /blobs` policy. The core test suite runs twice, identity filter and scoped, per
  the design's §14 warning.
- **7.5 [build] Scoped decision benchmark.** 5.7 repeated with a principal in one team of many; the
  pass condition is no measurable added latency.
- **7.6 [build] Third index engine (F7).** MySQL or SQL Server added to the CI matrix; the full suite
  passes with no engine-specific branches in core code.
- **7.7 [explore] Module boundary audit (B1).** For each module in 7.1–7.4: tables added (must be zero for
  artifact modules), routes added (must be zero), core queries issued by infrastructure modules (must be
  zero), and what hooks were missing. This is the direct evidence for B1 and B8.

**Milestone M5** closes this phase.

---

## 11. Phase 8 — Facade conformance and clients (size: M, runs from M2 onward)

Goal: B6 — existing tooling works unchanged, and new tooling finds the new links.

- **8.1 [build] Client suite matrix.** In CI against MkII: `pact-broker` CLI, `pact_broker-client` gem
  specs, pact-reference verifier broker tests, pact-jvm broker tests, and the current broker's
  `spec/features` corpus replayed over HTTP where the facade claims coverage. Pass rate tracked per suite
  from M2; all green closes the phase.
- **8.2 [build] New-client path.** The Pact CLI (or `pact_broker-client`) looks for the MkII links on `/`
  and uses the streaming publish, `HEAD /blobs` shortcut and events cursor when present, falling back to
  facade links otherwise. Stretch: the Janus CLI publishing a contract + shape manifest.
- **8.3 [build] Minimal UI.** Read-only pages over HAL for parties, versions, publications, decisions and
  retention, enough to demonstrate link omission and scoped visibility. Not a design driver.
- **8.4 [explore] Compatibility report.** What the facade could not do, what the client suites assume
  that the design should change, and which current behaviours should be dropped rather than emulated.

**Milestone M6** closes this phase.

---

## 12. Phase 9 — Evaluation and design feedback (size: S/M)

Goal: turn the prototype into the design's next revision and a credible staged plan for the real build.

- **9.1 [build] Performance report.** B2, B3, B4 and B8 numbers side by side with the current broker on
  the same generated dataset and the same hardware: publish wall time by size, can-i-deploy latency by
  history, latency over a simulated year, scoped versus unscoped reads, store sizes.
- **9.2 [design] Open-questions resolution.** Walk the design's §16 list (see §13 below); for each, an ADR
  or an honest "still open, here is what we learned". Revise the design in place with prototype evidence,
  including drawbacks that materialised (facade friction from 4.6, module gaps from 7.7, import loss from
  5.6).
- **9.3 [doc] Demo and report.** A demo script CI runs: import a broker → publish through the old CLI →
  stage a multi-document Janus publication → a comparator produces a fact → can-i-deploy answers from
  decision state → retention expires a feature branch → one coalesced webhook fires → a pull client sees
  the same event. Plus a short report for the community and PactFlow.
- **9.4 [design] Staged implementation plan for the real build.** What graduates from the prototype, what
  is rewritten, the order, the migration story for OSS and PactFlow installations, and the naming and
  governance questions framed for the community.

---

## 13. Traceability: design open questions → plan tasks

| Design §16 open question | Addressed by | Answer (9.2) |
|---|---|---|
| Roles for messages: first-class names or aliases | 2.1 (glossary), 7.3 | |
| Publication identity across staged uploads; auto-complete after quiet period | 2.3, 3.4, 4.4 | |
| Where analyse runs for very large documents | 1.1, 2.3, 4.4 | |
| Decision state for many-to-many fact kinds (per pair vs per publication) | 2.4, 7.2 | |
| Scope granularity (finer than party/environment/subscription/secret/policy?) | 2.10, 7.4, 7.7 | |
| Unscoped entities: visible to all or none | 2.9, 2.10, 5.5 | |
| KV vs SQL for the index | 1.2, 1.3 → G1 (1.8), 5.7 | |
| Naming ("party", "publication", "fact") | 0.1, 2.1, 8.2 | |
| Language (Rust, revisit at first gate) | 0.3, 1.1–1.7 → G1 (1.8) | |

The bets B1–B8 map to milestones as follows: B1 → M1, M5 (3.8, 7.7); B2 → M2 (4.4); B3 → M3 (5.7);
B4 → M4 (6.3); B5 → M4 (6.6); B6 → M6 (8.1); B7 → M3 (5.6); B8 → M5 (7.4, 7.5).

---

## 14. Cross-cutting practices

- **Decision log** (`Documentation/decisions/`): every gate and contested choice is an ADR, linked from the
  design when the design is revised.
- **Two store combinations in every CI run** from Phase 3: SQLite + filesystem and Postgres + MinIO. A
  third SQL engine joins at 7.6. No engine-specific code in core.
- **Two access modes in every CI run** from Phase 3: identity filter and a scoped principal, so the
  unscoped path cannot mask bugs in the scoped one.
- **Corpora are load-bearing**: the query corpus (1.4), the core conformance corpus (3.7) and the client
  suite matrix (8.1) are the executable specification. Behaviour changes require a corpus change in the
  same commit.
- **Benchmarks in CI** from Phase 5 (trend, not gate), on the generated dataset.
- **Spikes are disposable, findings are not**: every `spikes/` entry has a `FINDINGS.md`; code may be
  deleted, the finding is referenced by ADRs.
- **Modules from the docs**: every module in Phase 7 records where reading core source was necessary.
- **Honesty about drawbacks**: 4.6, 5.6, 7.7 and 8.4 exist to gather evidence *against* the bets where it
  exists. A prototype that cannot fail cannot de-risk anything.

## 15. Immediate next steps

1. **0.1** — write the charter and pick the working name; create the repository.
2. **0.2, 0.3** — ADR log and workspace scaffolding, kept thin until G1.
3. **0.4, 0.5** — reuse inventory and the production data profile, in parallel; 0.5 blocks 1.4 and so
   blocks 1.2, so it starts first.
4. **1.1 and 1.4** — the streaming publish spike and the dataset generator, the two cheapest pieces of
   evidence and the ones the rest of Phase 1 depends on.
