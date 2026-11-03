# Graph and vector storage decision

Status: Ladybug selected for Frontier; integration pending. The working
memory foundation currently uses Qdrant for embeddings and SQLite for its
graph. The brainstem/workspace learning loop can use that foundation through
the memory service while storage integration develops.

## Memory in the cognitive loop

The [architecture](2026-10-29-frontier-architecture.md) describes how current
input and retrieved experience reach the brainstem and workspace together.
EmbeddingGemma 2 provides multimodal retrieval; graph relationships connect
encounters, intentions, actions, dependencies, consequences and practices.
Retrieval returns content and provenance that the deciding model can use.

The memory service preserves source identities across record, graph and
vector views. Its responses include retrieval distance, relevant graph
context, source/practice revisions and projection readiness. It distinguishes
a completed search with little relevant evidence from unavailable or stale
memory. Outcome-grounded beliefs and their evidence support growing habits;
the [learning design](identity-and-learning.md) describes their revision.

Qdrant and SQLite have separate commit boundaries. Persist source records
and pending projection work durably, apply updates by stable identity, and
expose which graph/vector revisions are ready. Recovery catches up pending
projections without counting replay as a new experience. A unified Ladybug
backend must preserve the same observable memory contract.

## Storage requirements

Storage selection follows workload requirements. Graph queries
need a real graph capability; this does not require a separate graph server
if a unified engine proves suitable. Do not implement hand-written recursive
SQL traversal as our general-purpose graph engine.

## Selected engine: Ladybug outside the main BEAM

Ladybug is the selected engine. It provides both
graph operations and vector indexing, so a separate Qdrant service is not
a default dependency. Compare an external native worker with the
managed-WASM-on-BEAM alternative; never load the native C++ engine directly
as a NIF in Frontier's main VM. See the complete
[isolation and recovery contract](ladybug-isolation.md).

Use a typed Elixir memory API over this service for records, graph paths,
vectors, revisions and provenance. Before migration, validate the engine's
transaction/operation-receipt behavior, vector updates, recovery and bounded
resource use on copied data. The reported indexed-vector mutation issue
is an explicit regression test, not assumed fixed by a recent release.
[Vector support](https://docs.ladybugdb.com/extensions/vector/),
[open issue](https://github.com/LadybugDB/ladybug/issues/377).

The following PostgreSQL-first evaluation is retained as a fallback
analysis. Ladybug supersedes it as the selected engine; PostgreSQL remains
an alternative rather than a second implementation track.

## Historical alternative: managed PostgreSQL with AGE and pgvector

Evaluate one PostgreSQL database with logical schemas for operational
records, source experience, knowledge/code graphs and vectors. Apache AGE
supplies Cypher graph operations; pgvector supplies similarity search; SQL
accesses source records and job state. Elixir owns the domain API and
supervision, rather than reimplementing storage and numerical engines.
[AGE](https://age.apache.org/age-manual/master/intro/overview.html),
[pgvector](https://github.com/pgvector/pgvector).

The first version candidate is stable PostgreSQL 18 with the PG18 AGE 1.8.0
artifact and pgvector 0.8.7. See the dated
[project verification](../research/2026-10-29-storage-project-status.md).
This combination has not been built or tested locally. Refresh supported
minor versions and pin artifacts before the experiment.

Prefer it if it passes correctness, recovery and resource gates: a common
transaction domain, IDs and SQL inspection could reduce integration work.
But Cypher support alone does not establish efficient physical traversal.
Inspect plans, indexes, variable-depth behavior and concurrent writes.
AGE creates PostgreSQL tables for graph storage; it is not an independent
native graph engine. [Graph storage](https://age.apache.org/age-manual/master/intro/graphs.html).

### PostgreSQL lifecycle under Elixir

Use bounded Postgrex pools and explicit query timeouts. Initialize AGE
session state on every connection and reconnect; verify `agtype` and vector
encoding. Parameterize queries; select graph/schema names from trusted
configuration. [Postgrex](https://hexdocs.pm/postgrex/readme.html).

A storage manager verifies the configured local database's data-directory
identity, extensions, schema and readiness. PostgreSQL is an OS service,
not an Elixir process. Exactly one owner starts/stops it: use a dedicated
OS service unit, with Elixir managing readiness, admission and recovery
coordination. Do not launch a second postmaster from an OTP child.
Application/faculty restarts leave the database running. Explicit database
maintenance uses supported lifecycle controls.
[pg_ctl](https://www.postgresql.org/docs/current/app-pg-ctl.html).

Control, graph and vector workloads have separate bounded pools and query
budgets so research cannot exhaust control connections. They still share
CPU, I/O and database failure exposure; measure contention and reserve
resources. New effects stop when authoritative storage is unavailable.
STOP can revoke in-memory admission immediately; a failed write cannot
receive a durable acknowledgment.

### PostgreSQL data ownership and consistency

| Workload | Authority or query facility |
|---|---|
| Accepted events, obligations, work ownership and effect intents | Transactional ledger tables |
| Experience, identity versions and provenance | Source records; interpretations cannot overwrite observations |
| Typed code/knowledge relationships and traversal | AGE graphs with coherent revisions |
| Similarity search | pgvector, scoped by domain and embedding-model revision |
| Hot workspace views and routing caches | Rebuildable ETS, never sole durable state |
| Images/audio and large artifacts | Bounded files/buffers with explicit retention; no default raw-media archive |

All writes use typed domain APIs. Default direct inspection is read-only;
models cannot use database functions to rewrite ownership or grant scope.

Test that one explicit transaction can commit or roll back a source record,
AGE mutation and supplied vector together. Do not promise this until the
mixed-extension test passes. Never generate embeddings while holding a
transaction open. If a vector is unavailable, commit the source and a
pending-index job together, then index idempotently by content/model
revision. The state honestly remains unembedded until that job commits.
Notifications are wake hints; durable cursors drive recovery.

Nodes/edges carry stable IDs, source events, observed/inferred status,
validity and supersession. Code entities also carry repository, file/symbol
identity and content revision. Publish coherent graph revisions. Similarity
retrieves candidates; it is not proof of an assertion.

The query API takes namespace, source revision, relation types, depth,
result limit and execution budget. Results contain paths, provenance,
actual revision and completeness/truncation. Incomplete or timed-out search
does not prove a relationship absent.

## Other alternatives considered

Earlier options included unified PostgreSQL and a specialist graph engine
plus Qdrant. Ladybug's own graph/vector capability means the latter split
is not required. Existing Mirai stores remain intact pending migration;
Ladybug selection does not by itself authorize moving either current store.

For split engines, commit a mutation intent/outbox in the operational
ledger; apply its stable ID and graph changes in one graph transaction;
then acknowledge the committed revision. Recover a crash between commit
and acknowledgment by looking up that ID. Vector writes have independent
cursors. There is no cross-engine atomic transaction; readers must know
which revision is ready or stale. Proving this machinery is part of the
cost of choosing separate engines.

An Elixir-native memory service is recommended whichever engine wins. A
custom graph/vector engine is a separate research option only if the
candidates expose a concrete unmet requirement. Compare its restricted
query subset, vector recall, crash safety, backup, upgrades and maintenance
cost. Native indexing primitives are compatible with an Elixir API;
rewriting two engines is not a prerequisite for Frontier.

Neo4j or Memgraph can enter a later comparison for a specific gap; do not
start a broad engine contest by default. Kuzu is archived and excluded
from new deployment.

## Integration and migration gate

Use copies of actual code graphs and Mirai evidence/association data, plus
synthetic growth, cycles and high-degree nodes. Establish small-graph
reference answers independently of the candidates. Test:

- symbol lookup, typed neighbors and transitive callers/change impact;
- evidence/contradiction paths with validity and repository filters;
- updates during reads and coherent graph revision publication;
- exact vector ground truth, ANN recall@k, filtered recall, updates/deletes
  and embedding-model isolation;
- mixed record/graph/vector rollback, duplicate mutations and process kills
  around commit; backup/restore and extension upgrade rehearsals;
- cancellation, read-only credentials and session setup after reconnect.

Measure cold/warm p50/p95, peak resident memory, disk size, ingestion/build
time, update cost and inference interference. Approximate pgvector indexes
can filter after scanning: test filtered recall rather than assuming LIMIT
returns the right neighbors. [Filtering](https://github.com/pgvector/pgvector#filtering).

Correctness, recoverability and bounded resources remain mandatory after
selection. Record the pinned Ladybug engine and extension versions,
configuration, corpus, results and license review before migration. A
blocking integration failure is reported for human review; it does not
silently select another engine. Documenting Ladybug does not pass this gate.
