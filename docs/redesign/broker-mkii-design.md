# Pact Broker MkII — a thought experiment

Companion to [broker-mkii.md](broker-mkii.md), which lists the problems. This document proposes a design.
It follows the shape of the [Pact MkII RFC](https://github.com/pact-foundation/roadmap/pull/146):
keep / change / throw away first, then a reference-level design, then migration, drawbacks and open
questions. Where the Janus prototype already settled something that constrains the broker (the
[broker integration notes](../pact-janus/Documentation/broker-integration-notes.md), ADR 0011), this
document takes it as given rather than re-deciding it.

Working title: **Broker MkII**. Nothing here is a commitment to a name, a language or a storage engine;
those are the questions a prototype should answer, and §9 says which ones.

## Contents

1. [Summary](#1-summary)
2. [What we keep, change and throw away](#2-what-we-keep-change-and-throw-away)
3. [The problems, restated as design forces](#3-the-problems-restated-as-design-forces)
4. [Core model: parties, artifacts, roles, facts](#4-core-model-parties-artifacts-roles-facts)
5. [Storage: content-addressed blobs, a small relational index](#5-storage-content-addressed-blobs-a-small-relational-index)
6. [Publishing: staged, multi-document, resumable](#6-publishing-staged-multi-document-resumable)
7. [The decision engine: can-i-deploy without the matrix](#7-the-decision-engine-can-i-deploy-without-the-matrix)
8. [Lifecycle: retention as policy, not as a rake task](#8-lifecycle-retention-as-policy-not-as-a-rake-task)
9. [Events: an outbox, pull first, push second](#9-events-an-outbox-pull-first-push-second)
10. [Modules and extension points](#10-modules-and-extension-points)
11. [API: HAL kept, resources rethought](#11-api-hal-kept-resources-rethought)
12. [Migration path for existing brokers](#12-migration-path-for-existing-brokers)
13. [Implementation choices](#13-implementation-choices)
14. [Drawbacks](#14-drawbacks)
15. [Alternatives considered](#15-alternatives-considered)
16. [Unresolved questions and what a prototype must prove](#16-unresolved-questions-and-what-a-prototype-must-prove)

---

## 1. Summary

The current broker is a **relational model of Pact** with an HTTP API in front of it. Every one of the nine
problems in broker-mkii.md traces back to one of three decisions made early and never revisited:

1. The **domain vocabulary is Pact's** (consumer, provider, pact, verification), so anything that is not a
   pact needs its own tables and its own API (problems 1, 2, 4).
2. **Everything lives in one Postgres database**, content included, and every question is answered by
   joining it at read time (problems 3, 5, 6, 7).
3. **Side effects are synchronous with publishing** and pushed at the world (problems 8, 9).

Broker MkII replaces those three decisions with:

1. A **generic core model**: *parties* publish *artifacts* about *versions*; artifacts have *types*
   registered by modules; *roles* relate a party to an artifact; *facts* record what one artifact said
   about another. Pact, OpenAPI/BDCT, Janus contracts and provider shapes are all instances, not
   special cases.
2. **Content-addressed blob storage** for the bytes, a **small relational index** for identity and
   relationships, and **precomputed decision state** so `can-i-deploy` is a lookup, not a join.
3. An **event outbox** with a pull API as the primary integration surface and push (webhooks) as a
   subscriber of it, with dedup and rate limits so a publish cannot fan out into a storm.

The HAL API and the concepts of branch, environment, deployed/released versions and `can-i-deploy` are
kept. The wire protocol clients speak today is provided by a **facade module** so existing tooling keeps
working while the new resources are adopted at the client's pace, exactly the way the broker itself
introduced `/contracts/publish`.

## 2. What we keep, change and throw away

| | What | Why |
|---|---|---|
| **Keep** | HAL and link-driven evolution | Proven: the publish endpoint change and RBAC-by-absent-link both worked. It is the mechanism that makes a facade and a gradual migration possible. |
| **Keep** | Pacticipant *version*, *branch*, *tag*, *environment*, deployed/released records | These are the coordinates every question is asked in; they are not Pact-specific. `can-i-deploy --to-environment` is the product. |
| **Keep** | Three-valued `can-i-deploy` (yes / no / unknown) | The Janus notes found this already correct and load-bearing for subsumption. |
| **Keep** | The idea of a *verification result* keyed to a pair of versions | The unit of evidence. Generalised to "a fact one artifact asserts about another". |
| **Keep** | Consumer version selectors / WIP / pending semantics | Hard-won behaviour; they become *queries over the index* rather than SQL views. |
| **Change** | Consumer/provider from columns to **roles** | Message publisher/subscriber, BDCT provider-contract-vs-consumer, Janus shape-vs-contract all fit "party plays role R in artifact A". |
| **Change** | Pact content from a DB column to a **content-addressed blob** | Removes TOAST bloat, allows streaming upload, dedups identical content across publishes and across pacticipants. |
| **Change** | One file per publish to a **staged, multi-document publish** | Lets teams stage per-test-suite files, and lets Janus publish contract + shape as linked documents. Per-document versioning as in the quilt prototype. |
| **Change** | The matrix query to **materialised decision state** maintained on write | The query that overloads the DB becomes a read of precomputed rows. |
| **Change** | `clean` from an operator-run task to **retention policy evaluated continuously** | Lifecycle is part of the model (problem 6, and the "configurable lifecycle" requirement). |
| **Change** | Webhooks from the integration mechanism to **one subscriber of an event log** | Pull becomes possible; storms become preventable. |
| **Throw away** | The 60-odd SQL views and materialised `*_matrix` tables | They are the Pact model baked into SQL and the reason nothing else fits. |
| **Throw away** | Storing verification *results* in full in the primary DB | Result documents become blobs like everything else; only the verdict and summary are indexed. |
| **Throw away** | Postgres as the only possible store | The index needs a small, boring relational schema (or a KV store, §13); the bytes need an object store. |
| **Throw away** | Synchronous webhook triggering inside the publish transaction | Publishing writes facts and an outbox row; everything else is asynchronous. |

## 3. The problems, restated as design forces

Each problem in broker-mkii.md becomes a force the design must satisfy, so §4–§11 can be checked against
them.

| # | Problem | Design force |
|---|---|---|
| P1 | Model built for Pact, PactFlow/BDCT/Janus bolt on tables and paths | **F1 — Open artifact types.** Adding an artifact type must not add tables or top-level routes. |
| P2 | Consumer/provider hardcoded | **F2 — Roles, not columns.** The relationship between party and artifact is data. |
| P3 | Content in Postgres; 50–256 MB files; 900 GB of TOAST | **F3 — Bytes out of the index.** Upload must stream, dedup, and complete inside a load balancer's timeout without processing the content inline. |
| P4 | One JSON file per publish | **F4 — Multi-document, staged publish** with per-document versioning. |
| P5 | Matrix query overloads DB | **F5 — Decisions are reads.** `can-i-deploy` must not join at request time. |
| P6 | Accumulation degrades DB | **F6 — Retention is policy** and runs continuously; the hot set stays small. |
| P7 | Postgres only | **F7 — Narrow storage interface.** No engine-specific SQL in the core. |
| P8 | Webhook thundering herd | **F8 — Fan-out is throttled and deduplicated.** |
| P9 | Push-only webhooks | **F9 — Pull is first class.** |
| — | Modular; auth/RBAC pluggable | **F10 — Module boundary** for types, auth, storage, notifications. Team-level visibility (PactFlow Teams) must be enforceable below the API layer. |
| — | Branch-centric CI | **F11 — Branch is a coordinate**, not a tag with a naming convention. |
| — | Existing users must be able to move | **F12 — Facade and import.** |

## 4. Core model: parties, artifacts, roles, facts

Seven concepts. Everything else is a module's interpretation of them.

```
Party ──< Version ──< Publication ──< Document (typed, versioned, blob-backed)
  │           │              │
  │           │              └──< Role (party, role-name)           e.g. consumer / provider / publisher / subscriber
  │           └──< Coordinate (branch, tag, environment membership, deployed, released)
  │
  ├──< Fact (subject publication, object publication, kind, verdict, summary-blob)   e.g. verification, subsumption
  │
  └──< Scope membership (entity, scope)          visibility grouping; given meaning by an infrastructure module (§4.6)
```

### 4.1 Party

What `pacticipant` is today: a named thing that has versions. Renamed only because the word "pacticipant"
assumes pacts; keep the old name in the facade. Parties have display name, repository URL, labels, and
nothing else that is Pact-specific.

### 4.2 Version and coordinates

Unchanged in intent. A version belongs to a party and has an ordering (creation order, as today).
Coordinates attach to versions:

- **branch** — a version belongs to one or more branches; each branch has a head (latest version).
- **tag** — legacy, kept for the facade and for teams that use it.
- **environment membership** — deployed / released records with `currently` flags, as today.

These are the selectors every question resolves against, so they live in the index, not in blobs.

### 4.3 Publication and Document

A **publication** is the act of a party publishing a set of documents about one of its versions. A
publication has:

- the publishing party and version (the *publisher* — not to be confused with the message-publisher role);
- a **type** (`pact`, `oas`, `janus-contract`, `janus-provider-shape`, `verification`, …) registered by a
  module (§10);
- one or more **documents**, each with a name, media type, and content-addressed blob reference;
- a set of **roles** (§4.4);
- metadata the type's module extracted at publish time (e.g. for pact: interaction count, has-messages,
  spec version; for a shape: provenance).

Publications are immutable. Publishing the same documents again for the same version creates a new
publication whose documents point at the same blobs (dedup is free). Publishing with only one document
changed creates a new publication with one new blob and N−1 shared ones — the quilt prototype's model,
where the *index* is what moves and unchanged documents keep their version.

### 4.4 Role

A role relates a **party** to a **publication** with a **name**. Names come from the type's module:

| Type | Roles |
|---|---|
| `pact` (HTTP) | `consumer` (the publisher), `provider` |
| `pact` (messages) | `subscriber`/`consumer` (the publisher), `publisher`/`provider` — the module can present both vocabularies against the same stored roles |
| `oas` provider contract (BDCT) | `provider` (the publisher); no counterpart until compared |
| `janus-contract` | `consumer` (publisher), `provider` |
| `janus-provider-shape` | `provider` (publisher), *no consumer role at all* — the exact case the current schema cannot express |
| `verification` | `verifier` (publisher), points at the publication verified via a Fact |

An **integration** (today's consumer↔provider pair) becomes a *derived* view: two parties that appear in
complementary roles on the same publication or on the two sides of a fact. It does not need its own table
beyond a cache.

This resolves P2 without inventing a third role model for messages: the UI and the facade decide what to
call the roles for a given type; the core stores names.

### 4.5 Fact

A fact is a typed, immutable assertion **from one publication about another**:

```
Fact {
  kind:      "verification" | "subsumption" | "compatibility" | ...   (registered by a module)
  subject:   publication id      (e.g. the verification result publication, or the provider-shape publication)
  object:    publication id      (e.g. the pact publication it verified / the contract it was compared to)
  verdict:   pass | fail | unknown                              (Kleene; the only value the decision engine reads)
  summary:   small JSON (counts, engine version, exercised variants)   — indexed
  detail:    blob reference                                     — the full result document, not indexed
  produced_by: party version (the provider version that ran the verification, etc.)
}
```

Today's `verifications` table is a fact of kind `verification` whose subject is a result publication and
whose object is the pact publication. The Janus subsumption report is a fact of kind `subsumption`
between a shape publication and a contract publication, keyed exactly as the integration notes recommend
(contract SHA, shape SHA, engine version) — that key is just the two publication blob hashes plus a
summary field.

The decision engine (§7) reads verdicts. Nothing in the core reads `detail`.

### 4.6 Scope

A **scope** is a visibility group. The core owns two small tables, `scopes` and
`scope_memberships(entity_kind, entity_id, scope_id)`, and attaches no meaning to them. The entities that
can be placed in a scope are the ones an operator assigns to a team today: **parties, environments,
webhook subscriptions, secrets and retention policies**. Publications, facts, decision-state rows and
events are never scoped directly; their visibility is derived from the parties that hold roles on them.

The core enforces scope in exactly one place: an **access filter hook** in the repository layer.
Every dataset query for a scopable entity passes through `access.filter(principal, kind, query)` before it
executes, so a resource that forgets to check fails closed rather than open. The OSS default is the
identity filter over a single implicit scope. What a scope *is* (a team, a tenant, a business unit) and
who is in it is the job of an infrastructure module (§10.2), which is why the core stores scopes but
never users.

Derived visibility, per read:

| Read | Visible when |
|---|---|
| Party, environment, subscription, secret, policy | the entity is in a scope the principal can see (or the instance runs with the identity filter) |
| Publication | any party holding a role on it is visible. A pact between team A's consumer and team B's provider is visible to both. |
| Fact | both its subject and object publications are visible |
| Decision-state row | its left and right party ids are in the principal's visible set. The rows already denormalise party ids (§7.2), so this is an `IN` over a per-request set that is small for any real team, and the read stays indexed. |
| Event | the party ids the event carries are visible, so a team's pull client (§9.2) sees only its own integrations |
| Blob | a visible publication references the hash (§5.4) |

Scope exists so that visibility can be enforced below HAL. Link omission (§11) handles *actions* a
principal may not take; it cannot hide rows from a collection, and hiding rows is what team-level
separation requires. Multi-tenancy is the same mechanism with cross-scope visibility switched off: a
tenant is a scope that nothing crosses.

## 5. Storage: content-addressed blobs, a small relational index

### 5.1 Two stores, one interface

```
┌──────────────────────────────────────────────────────────────┐
│ Core                                                         │
│   BlobStore   put(stream) -> hash · get(hash) -> stream ·    │
│               exists(hash) · delete(hash) · size(hash)       │
│   Index       parties, versions, coordinates, publications,  │
│               documents(hash refs), roles, facts,            │
│               decision_state, outbox, retention_marks        │
└──────────────────────────────────────────────────────────────┘
      │                              │
      ▼                              ▼
 S3 / GCS / Azure Blob /        Postgres / MySQL / SQLite /
 filesystem / (DB fallback)     SQL Server / (or a KV store, §13)
```

**BlobStore** is the quilt prototype's file system layout generalised: the key is the content hash
(SHA-256 of the canonical bytes), the value is the bytes. Storing the same pact twice a day costs one
index row per publish and zero new blob bytes. A filesystem backend serves single-node and dev
installations; an object store serves production. A "blob in the index database" backend exists only so a
zero-config install works, and it is not the default.

**Index** holds identity, relationships and small metadata only. No column in the index holds a document.
Its schema is small enough (a dozen tables, no views) that supporting several SQL engines is a matter of
avoiding vendor features, not of maintaining parallel view definitions (F7).

### 5.2 The publish path for a 256 MB file

The current failure is that the request has to be read, parsed, sorted, hashed, diffed and stored inside
one HTTP request under a 60 s load balancer timeout. MkII separates *receiving* from *processing*:

1. Client `POST`s the document as a stream. The server streams it to the BlobStore while hashing it,
   returns `201` with the hash. No parsing. Bounded by network throughput only.
2. Client `POST`s the publication manifest: version coordinates, type, roles, document list (names →
   hashes). This is a few kilobytes and completes in milliseconds. The publication exists.
3. A background worker runs the type module's `analyse(blobs)` step: parses, extracts metadata (interaction
   SHAs, has-messages, provenance), computes "content that affects verification" hashes, and appends the
   outbox events. Its output is written to the index as publication metadata.

Clients that want the old one-shot behaviour use the facade, which does steps 1–2 for them and returns
when 2 completes. The 256 MB customer's publish succeeds because the only synchronous work is streaming
bytes to storage. Optional: the client sends the hash first (`HEAD /blobs/{hash}`) and skips the upload
when the blob exists — a republish of unchanged content becomes one small request.

### 5.3 What this does to the 900 GB

- Identical content published repeatedly: stored once.
- Content of expired publications: deleted from the blob store by the retention process (§8) when no
  publication references the hash. Blob deletion is a cheap object-store operation, not a Postgres vacuum.
- The index no longer has TOAST tables; backups, replication and failover become routine.

### 5.4 Blob access under scoping

Content addressing deduplicates across parties, so two scopes can legitimately share a blob. A raw
`GET /blobs/{hash}` therefore cannot be authorised by the hash alone. Blob reads are served through a
publication's document link, and the check is "does a publication visible to this principal reference
this hash". The `HEAD /blobs/{hash}` upload shortcut has the same problem in the other direction: it
reveals whether anyone in the instance has published those exact bytes. The core offers two policies,
chosen by the access module: scope the existence check to the caller's visible publications, or disable
the shortcut and always accept the upload. The second costs bandwidth and never leaks, and is the right
default for an instance that runs team separation at all.

## 6. Publishing: staged, multi-document, resumable

The two-step publish in §5.2 already gives *staging* for free: uploaded blobs are inert until a manifest
references them. Making it a first-class workflow:

```
POST   /blobs                                   stream a document, get its hash (idempotent)
HEAD   /blobs/{hash}                            does the broker already have it?
POST   /parties/{p}/versions/{v}/publications   manifest: type, roles, documents [{name, mediaType, hash}], metadata
GET    /publications/{id}                       HAL: documents, roles, facts, decision links
```

Behaviours:

- **Partial staging across CI jobs.** Each test-suite job uploads its own blob(s). A final job posts the
  manifest listing all of them. If a job re-runs and produces identical bytes, the hash matches and no
  new blob is stored. No client-side merge step (today's `pact-broker publish` directory merge) is
  required, although the module may still *merge* documents at analyse time if the type wants a single
  logical contract (pact does).
- **Linked documents.** A Janus publication lists `contract.json` and, optionally, `shape.json` in one
  manifest. A shape published on its own is a publication of type `janus-provider-shape` with a single
  document and a single `provider` role.
- **Per-document versioning.** Because documents are identified by hash, "which documents changed since
  the last publication for this branch" is a set difference over hashes — the quilt prototype's behaviour
  without the per-document version counters, which are derivable.
- **Unreferenced blobs** are garbage-collected after a grace period (§8).

Media type is free. JSON, YAML, protobuf descriptors, HAR files, and the Janus formats are all just bytes
with a media type; the type module decides what it can analyse.

## 7. The decision engine: can-i-deploy without the matrix

### 7.1 What the matrix actually answers

Every `can-i-deploy` question reduces to: *for a set of selected versions, for every relationship each
selected version participates in, what is the verdict of the most relevant fact between the selected
version and the version of the other party that the target selector resolves to?* Today that is computed
by joining pacts, publications, verifications, tags, branches and deployments at query time, and it is
the query that overloads the database (P5).

### 7.2 Maintain decision state on write

MkII stores a **decision state** table with one row per *(publication A, publication B)* pair that has
ever had a fact between them, plus one row per publication that has never had a counterpart
(the "never verified" rows today's left outer join produces):

```
decision_state (
  left_publication, right_publication,       -- ordered by role so the pair is unique
  relationship_kind,                         -- pact-verification | subsumption | ...
  verdict,                                   -- pass | fail | unknown  (the latest fact's verdict)
  latest_fact,                               -- for the link
  left_version, right_version, left_party, right_party   -- denormalised for selector lookup
)
```

Every write that can change an answer updates only the rows it touches:

- new **fact** → update the one pair row's verdict;
- new **publication** → insert its "unknown" rows against the other party's publications it will be
  compared with (the module's `expected_counterparts` hook decides; for pact, that is "verifications by
  the provider", and the row starts as *unknown*);
- **coordinate change** (branch head moves, deployed record created) → nothing changes in
  decision_state; selectors resolve *versions* first, then read the rows.

`can-i-deploy` then becomes:

1. resolve selectors to version ids (small indexed lookups on coordinates — the same resolution the
   broker does today, minus the matrix join);
2. read `decision_state` rows where `left_version ∈ selected ∧ right_version ∈ resolved-target`
   (one indexed range read per selected version);
3. apply the three-valued summary and the reason vocabulary.

Step 2 is O(rows returned), never O(history). Pending/WIP logic and "latest by" collapsing operate on the
returned rows in memory, as they do today, but on tens of rows instead of the full matrix.

### 7.3 Time-dependent parts stay computed on read

The Janus notes are right that **reports cache and decisions do not**, because exemptions and policies
expire. The decision state stores verdicts, which are timeless. Policy (ignore rules, subsumption
exemptions with reasons and expiry) is applied in step 3 with the current clock. This is the same split.

### 7.4 Rebuildable

`decision_state` is a **projection** of publications and facts. It can be dropped and rebuilt from the
index in a background job, which is also how the importer (§12) populates it and how a bug in the update
logic is recovered from. Nothing that cannot be recomputed lives in it.

## 8. Lifecycle: retention as policy, not as a rake task

`clean` today is a large delete run by operators, with a selector language that is hard to reason about
and expensive to execute. MkII makes retention part of the model:

- A **retention policy** is a set of *keep rules*, each a selector plus a duration, attached at three
  levels: instance default, per party, per branch pattern (e.g. `feature/*` keeps 14 days, `main` keeps
  forever, anything deployed or released keeps while it is deployed plus 90 days). The defaults mirror
  the current `CleanIncremental` keep selectors so behaviour on upgrade is unchanged.
- A **marker process** runs continuously in small batches: it evaluates rules against versions and stamps
  an `expires_at` on versions and publications. The UI shows it; a `retention` link on a version shows
  which rule is keeping it.
- A **collector** deletes expired publications, then facts and decision rows that reference them, then
  blobs no publication references. Each step is an indexed delete of a bounded batch. The hot working set
  stays proportional to the number of *live* versions, so performance no longer decays with age (P6).
- **Everything a CI publishes carries the branch it came from**, and branch-level rules are the primary
  knob, which is the "configurable lifecycle for anything published from a CI" requirement.
- Rules are data with an API and a UI; a `dry-run` link shows what a rule change would expire.

Deleting an old version never changes a `can-i-deploy` answer for a live one, because decision state rows
are keyed by publication and a live publication's rows survive.

## 9. Events: an outbox, pull first, push second

### 9.1 The outbox

Every state change (publication analysed, fact recorded, verdict changed, coordinate changed, retention
expiry) appends an **event** to an append-only, ordered log in the index, in the same transaction as the
change. Events carry the ids and enough denormalised fields for a subscriber to filter without a second
request. Events are retained for a configurable window.

### 9.2 Pull

```
GET /events?after={cursor}&types=verdict_changed,publication_analysed&party=orders-api
```

A CI system, a GitOps controller, or a customer's own relay polls this with a cursor. Nothing in the
customer's network needs to be reachable from the broker (P9). Long-polling and, where the deployment
allows it, server-sent events, are optimisations over the same cursor. A per-party or per-branch cursor
lets a pipeline ask "has anything happened to my integrations since my last build" in one request — the
question webhooks were always a proxy for.

### 9.3 Push (webhooks) as a subscriber

Webhooks become one built-in subscriber to the log, running in a worker, with:

- **Debounce/coalesce**: a webhook subscription defines a coalescing key (e.g. provider + branch) and a
  quiet period; N events within the period produce one delivery carrying the batch. This alone removes
  most of the herd (P8): ten consumers publishing in a minute produce one provider build, not ten.
- **Rate limits and budgets** per subscription and per target host.
- **Loop detection**: deliveries carry a causation id; a publication made by a build that a webhook
  triggered records that id, and a subscription can be configured not to fire again within a chain
  depth or window.
- **Verdict-change semantics**: the most useful event is *"the answer to a deploy question changed"*
  (`verdict_changed`), which is derived from decision state and fires far less often than
  "something was published". Today's `contract_content_changed` was a step in this direction.

The template/rendering language for webhook bodies is kept (it is widely used), reimplemented against
the event payload.

## 10. Modules and extension points

The core knows about parties, versions, coordinates, publications, documents, roles, facts, scopes,
decision state, retention and events. Modules supply behaviour at defined points, and they come in two
kinds with different rules.

### 10.1 Artifact modules: no tables, no routes

| Extension point | Provided by | Examples |
|---|---|---|
| **Artifact type** | `type` module: role names, `analyse(blobs) → metadata + derived documents`, `expected_counterparts`, `content_hash_for_change_detection`, renderers, diff | `pact`, `oas`, `janus-contract`, `janus-provider-shape`, `verification`, `openapi-diff-report` |
| **Fact kind** | `fact` module: how a fact is submitted, what its verdict means, summary schema | `verification`, `subsumption`, `bdct-compatibility` |
| **Comparator** | runs inside the broker to *produce* facts from two publications when no CI has | the Janus engine as a WASM component (`subsumption`), an OAS-vs-pact comparator (BDCT) |
| **UI plugin** | pages/renderers for a type | pact HTML renderer, shape view, subsumption pair view |

**An artifact module never adds a table or a top-level route.** It adds a type name, metadata (a JSON
document on the publication), documents (blobs) and facts. If that constraint is too tight for a real
artifact, the core model is wrong and should be fixed centrally; that is the test of F1.

Comparators deserve a note: the Janus notes concluded that the broker should embed the engine as a WASM
component rather than reimplement the walk, and that the walk is a pure function so its results cache by
input hashes. A comparator module is exactly that: given two publications, produce a fact. Running them in
a worker, keyed by input hashes, gives the N×M cross-product without N consumer builds. It also gives
BDCT the same home instead of a separate subsystem.

### 10.2 Infrastructure modules: own tables, keyed by core ids

| Extension point | Provided by | Examples |
|---|---|---|
| **Blob store** | `put/get/exists/delete` | filesystem, S3-compatible, GCS, Azure, in-index |
| **Index store** | the repository interface | Postgres, MySQL, SQLite, SQL Server; a KV option (§13) |
| **Authentication** | request → principal | basic, API token, OIDC/JWT, mTLS, a header from a proxy |
| **Access** | `filter(principal, kind, query)` for visibility (§4.6) and `permitted?(principal, action, resource)` for actions; the HAL renderer omits links `permitted?` denies | allow-all (OSS default); Teams + RBAC; tenancy; audit |
| **Notification sink** | consumer of the event log | webhooks, message queue publisher, Slack, email |

**An infrastructure module may own its own tables, keyed by core ids and scope ids, but it may not add
columns to core tables and may not issue its own queries against them.** Everything it needs from the
core arrives through the hooks. The thing that stays out of modules is *interpreting core tables*, not
*having tables*.

Teams + RBAC is the worked example, because it is the case that forced the split. The module owns users,
teams, team membership, roles and permissions. It maps a team to a core scope, implements `filter` as
"parties whose scope is in the principal's teams", implements `permitted?` for actions, and decides the
one policy the core cannot: whether an entity in no scope is visible to everyone or to nobody. The core
never learns what a team is. The same module with cross-scope visibility disabled is multi-tenancy.

Postgres row-level security would be a tidy implementation of `filter` for the Postgres index backend, but
it is not portable across the engines F7 asks for, so it can only ever be an optimisation behind the
hook, never the mechanism. A separate index per team is too coarse, because cross-team integrations are
the point of a broker.

### 10.3 Discoverability

**A module is discoverable through HAL.** The index resource lists the types and fact kinds the
instance supports, so a client (or the Pact CLI) can ask "can this broker take a provider shape"
before trying, the same pattern as today's `pb:publish-contracts` link. Access modules are not
advertised; their effect is simply which links and rows a principal sees.

## 11. API: HAL kept, resources rethought

Resource families, all HAL, all link-discoverable from `/`:

```
/parties, /parties/{p}, /parties/{p}/versions/{v}
/parties/{p}/branches/{b}, /environments/{e}
/blobs/{hash}
/publications/{id}, /publications/{id}/documents/{name}
/facts/{id}
/decisions?selector=...            (can-i-deploy, can-i-merge; GET with the selector language)
/events?after=...
/retention/policies, /retention/preview
/subscriptions (webhooks and other sinks)
```

Principles carried over: unknown links are ignored by clients; actions the principal cannot perform have
no link; new capabilities appear as new links. Principles added: **every large payload is a blob with a
hash** and every list is paginated with a cursor; nothing returns unbounded collections.

## 12. Migration path for existing brokers

Three layers, adoptable independently:

1. **Import.** A one-shot (and re-runnable, incremental) importer reads an existing broker database and
   writes parties, versions, branches/tags/environments, one `pact` publication per pact publication (blob
   = `pact_version_contents`), one `verification` fact per verification, then rebuilds decision state.
   It is idempotent on (party, version, content hash), so it can run repeatedly while the old broker is
   still live, and the cut-over is a DNS change.
2. **Facade.** A module that serves today's URL space and payload shapes — `/pacts/provider/.../consumer/...`,
   `/contracts/publish`, `/matrix`, `/pacticipants`, `/webhooks`, `pb:*` link relations — on top of the
   core. Existing `pact-broker` CLI, `pact_broker-client`, the Pact SDK verifier fetches and PactFlow's
   own integrations work unchanged. The facade is where "pacticipant", "consumer" and "provider" continue
   to exist as words.
3. **New clients.** The Pact CLI and the Janus CLI look for the MkII links on `/` and use the streaming
   publish, the events cursor and the shape resource when present, falling back to the facade links when
   not, exactly as the publish change was rolled out before.

Verification results and webhook execution logs are imported as blobs, so history is not lost, but they
immediately fall under the retention policy.

## 13. Implementation choices

Language and storage are prototype questions, but there are constraints that narrow them:

- **Language.** The engine the broker will embed for comparison is Rust and builds to WASM. A Rust broker
  can link it natively; anything else embeds it via wasmtime. Both are viable. Rust gives one toolchain
  across the Pact MkII estate, a single static binary for distribution, and the performance headroom the
  publish path needs. Go would be the alternative for the same reasons; Ruby is ruled out only because
  the streaming publish path and the worker model are exactly where it is weakest, and because the
  existing code is what we are replacing, not porting. Recommendation: **Rust**, revisit at the first
  prototype gate if the web/ORM ecosystem cost is too high.
- **Web layer.** HAL rendering, content negotiation and conditional requests are the surface area that
  matters. webmachine-style resource dispatch (as in the quilt prototype) maps well to HAL resources;
  axum/actix are fine too. Not a deciding factor.
- **Index store.** The schema is deliberately small and relational-shaped (many-to-one, unique keys,
  range reads by version order). A SQL abstraction (`sqlx`/`sea-orm`) over Postgres, MySQL, SQLite and
  SQL Server satisfies F7 as long as no view or engine-specific feature is used. An embedded KV store
  (e.g. RocksDB/redb) is attractive for a single-binary install but pushes secondary indexes into
  application code; keep it as an option, not the baseline.
- **Blob store.** S3-compatible API as the production baseline (covers MinIO, GCS via interop, Ceph),
  filesystem for dev and small installs, in-index as the zero-config fallback.
- **Workers.** In-process job queue backed by the index (a `jobs` table with leases) so the single-binary
  install has no extra infrastructure. The interface allows an external queue later.
- **UI.** Server-rendered pages over HAL as today, or a small SPA consuming HAL. Not a design driver.

## 14. Drawbacks

- **Two stores instead of one** means two things to back up, and a consistency story (blob written before
  manifest; orphan blobs collected later). It is the standard pattern for artifact stores, but it is more
  than one Postgres.
- **A generic model is harder to explain** than "consumer, provider, pact". The facade and the type
  modules carry the vocabulary, but contributors to the core need to think in parties and roles.
- **Materialised decision state is code that can be wrong.** A bug produces wrong `can-i-deploy` answers
  until rebuilt. Mitigation: it is a rebuildable projection, and the importer/rebuild path is exercised
  constantly.
- **Async analyse** means a publication is visible before its metadata is; the API must expose an
  `analysis: pending` state and clients must tolerate it. The facade waits for analysis for one-shot
  clients.
- **Rewriting is a large investment**, and PactFlow's layers (BDCT, RBAC, teams, secrets) have to be
  re-expressed as modules. The upside is that they become modules rather than forks.
- **Scope is a core concept the OSS broker never uses.** It exists for commercial separation and adds a
  filter hook on every collection query. The cost is one `IN` over a small set; the risk is that the
  identity filter path masks bugs in the scoped path, so the test suite has to run both.
- **Two brokers exist for a while.** The facade limits it, but there will be a period of "which one has
  the truth", exactly as with the Janus dual-embedding period.

## 15. Alternatives considered

- **Incrementally fix the current broker** (blobs out of Postgres, materialise the matrix, add a shape
  table). Each fix is possible, but P1/P2 are structural: every new artifact still needs its own tables
  and paths, which is what PactFlow's experience with BDCT demonstrated. The Janus notes' §10 PR series is
  exactly this path and is right *as a stopgap*.
- **A generic artifact repository (Artifactory/Nexus/OCI registry) plus a thin decision service.**
  Tempting for §5 and §6 — an OCI registry is a content-addressed multi-document store with a manifest.
  It does not give branches/environments/facts/decisions, and forcing them into OCI annotations makes the
  decision service the real broker anyway. Worth stealing the manifest/blob shape, not the product.
- **Event-sourced everything** (the outbox as the source of truth, all state as projections). Elegant,
  and decision state already is a projection, but it makes the import story and ad-hoc queries harder
  than they need to be. MkII keeps the index as the truth and the event log as a durable feed.
- **Keep the matrix but cache it.** Caching a query whose inputs change on every publish is what the
  existing materialised matrix tables attempt; it did not solve P5 because invalidation is the hard part.
  Maintaining per-pair rows on write is the same idea done at the right granularity.

## 16. Unresolved questions and what a prototype must prove

The bets, in the form the Janus plan used, each with a falsifiable claim:

| # | Bet | Falsifiable claim |
|---|-----|-------------------|
| B1 | Party/role/publication/fact is sufficient | Pact (HTTP + messages), OAS provider contracts with BDCT comparison, Janus contracts and provider shapes can all be modelled **without a module adding a table or a top-level route**. |
| B2 | Blobs out of the index fixes the publish path | A 256 MB pact publishes in under the LB timeout on a modest instance; a day of republishing an unchanged 50 MB pact adds zero blob bytes. |
| B3 | Decision state makes can-i-deploy a read | On a dataset the size of PactFlow production (imported), `can-i-deploy --to-environment` completes in tens of milliseconds and does not scale with history. |
| B4 | Retention policy keeps the hot set bounded | Query latency after simulating a year of CI publishing under default policy is flat. |
| B5 | Coalesced events tame the herd | A burst of N consumer publishes produces one provider build trigger; a pull client sees the same information with no inbound connectivity. |
| B6 | The facade is good enough | The existing `pact-broker` CLI, `pact_broker-client` gem, and at least two Pact SDK verifiers run their full broker-related test suites against MkII unchanged. |
| B7 | Import is lossless where it matters | An existing broker imported into MkII returns identical `can-i-deploy` answers for a recorded corpus of real queries. |
| B8 | Scope is sufficient for Teams + RBAC | PactFlow's team-level visibility and permissions can be implemented as an access module over §4.6 with no change to core tables, and the per-request `IN` filter adds no measurable latency to B3's can-i-deploy read. |

Open questions for the RFC:

- **Roles for messages**: expose `publisher`/`subscriber` as first-class names or as display aliases of
  `provider`/`consumer`? (Storage is the same either way; this is a UX and CLI question.)
- **Publication identity across staged uploads**: is a manifest the only way to complete a publication, or
  should a "auto-complete after quiet period" mode exist for teams that cannot add a final CI step?
- **Where analyse runs for very large documents**: worker in the broker, or a client-side `analyse` step
  that submits metadata with the manifest (trusting the client)?
- **Decision state for many-to-many relationships**: a Janus shape compared against N contracts is N
  rows; is per-pair the right unit for every fact kind, or do some kinds want a per-publication summary?
- **Scope granularity**: §4.6 scopes parties, environments, subscriptions, secrets and policies. Does
  PactFlow's Teams model need anything finer (a single branch, a single integration) that would force
  scoping onto publications directly?
- **Unscoped entities**: visible to all or to none by default? The design leaves it to the access module;
  the OSS default and the PactFlow default may differ, and the importer (§12) needs an answer.
- **KV vs SQL for the index**: worth a spike before B3, since the range reads by version order are the
  only query shape that matters.
- **Naming**: "party"/"publication"/"fact" are working words. The facade keeps "pacticipant"; the new API
  needs words that read well in a CLI.

A prototype in the style of the Janus one — spikes for B2 and B3 first, since they are cheap to test and
decide the storage choices; then a thin vertical slice (publish a pact through the facade, import a real
database, answer `can-i-deploy` from decision state) — would settle the storage and language questions
within the first two phases.
