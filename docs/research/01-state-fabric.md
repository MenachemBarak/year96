# 01 — Universal State Fabric

Scope: This report designs the Year96 State Fabric: the log, schemas, snapshots, projections, search indexes, provenance, privacy boundaries, and provider interfaces that make “the whole world is state” technically implementable while keeping logic stateless. I focus on 2025–2026 systems and licence posture, because Year96’s core must stay permissive while still learning from commercial and copyleft systems.

## TL;DR for the Year96 architect

- Make **CloudEvents the outer envelope**, not the whole model. Use CloudEvents `id/source/type/subject/time` plus Year96 extensions for bitemporal `validTime` and `txTime`, actor, identity, causation/correlation/thread IDs, schema refs, provenance, sensitivity labels, and blob pointers. CloudEvents is CNCF graduated and Apache-2.0 [1].
- Treat the **append-only event log as source of truth**, and every database/search/graph/vector store as a disposable projection. Logic is stateless code over `StateQuery` + emitted `StateEvent`s.
- Use **NATS JetStream or Apache Kafka** as permissive core logs. NATS is simpler for laptop/edge; Kafka/Pulsar are mature at cluster scale; Redpanda and KurrentDB/EventStoreDB are useful but licence posture is not core-permissive enough for default adoption [2][3][4].
- Prefer **Apicurio Registry** over Confluent Schema Registry for core because Apicurio is Apache-2.0, while Confluent Schema Registry is under Confluent Community License, a use-restricted source-available licence [5][6].
- For time travel, use **event replay + immutable lake/table snapshots** as the universal mechanism. XTDB v2 is technically excellent for bitemporal SQL but MPL-2.0, so mark excluded for core; Apache Iceberg/Delta/Dolt/DuckLake are permissive candidates for projections and historical analytics [7][8][9][10].
- For “views update the moment state changes,” trial **Feldera/DBSP** and **RisingWave**. Feldera’s arbitrary incremental SQL over changes is very aligned to scope-effect; RisingWave packages ingestion + materialized views + serving SQL for agentic AI [11][12]. Exclude Materialize and Pathway as core where BSL applies [13][14].
- For unified search, use **Postgres + pgvector + VectorChord + Tantivy** on a laptop; use **Vespa** or OpenSearch + Qdrant/Milvus at cluster scale. Vespa is the strongest single permissive hybrid serving/ranking engine [15][16][17][18][19]. ParadeDB is attractive but AGPL in 2026, so not core [20].
- For graphs/ontology, adopt **LinkML + JSON-LD/RDF** for schemas and **Oxigraph/Jena** for RDF where needed; trial **Graphiti** for temporal agent memory but coordinate with #03. Kuzu is now archived, so do not make it a pillar [21][22][23][24][25].
- “Thoughts are state” means every intention, reasoning trace, tool call, critique, and plan becomes a `StateEvent` with W3C PROV lineage and OpenTelemetry trace/span IDs. OpenTelemetry GenAI conventions moved into a dedicated semantic-conventions repo, so Year96 should track that actively [26][27][28].
- “The system is state” means Git commits, prompts, policies, harness configs, IaC plans, schema migrations, evaluations, and release decisions are also addressed by `y96://` URIs and emitted as events. OpenTofu is MPL-2.0 per assignment memory and needs direct verification; Pulumi/CUE/KCL/Pkl are preferable for permissive core where possible [29][30][31][32].
- World freeze = a **consistent cut** across log partitions plus projection checkpoints plus blob content hashes. The frozen world is identified by a `WorldSnapshot` manifest, not by copying every byte.
- Privacy cannot be solved by immutable logs alone. Use envelope encryption per sensitivity domain, key hierarchy + crypto-shredding, redaction tombstones, projection rebuilds, retention classes, and proof logs.

## Landscape

### Log-centric architecture and universal event envelope

Jay Kreps’ “log as the heart of data infrastructure” pattern remains the right mental model: state is the fold of an ordered fact stream, and derived databases are caches. In Year96, this gives the invariant that every change, including agent thought and self-modification, is reproducible if the log and blobs survive. Apache Kafka is still the benchmark event log, with Apache-2.0 licence and very high maturity; the GitHub API check during this task showed about 33.9k stars and active pushes on 2026-09-28 [2].

CloudEvents matters because Year96 needs an event envelope that every connector and internal service can emit without bespoke adapters. The spec says it exists because event producers describe events differently, limiting libraries/tooling; it is CNCF graduated as of Jan 2024 and Apache-2.0 [1]. Year96 should extend it rather than invent a new outer transport shape.

NATS JetStream is the best lightweight permissive log for laptop/single-node/edge. The NATS server is Apache-2.0, about 20.8k stars in the GitHub API check, and was actively pushed on 2026-09-28 [3]. Its simpler operational surface fits Year96’s “many identities, many local processes” MVP.

Apache Pulsar remains relevant for multi-tenant streams, geo-replication, and tiered storage; Apache-2.0 and active. It is heavier than NATS and Kafka but worth a cluster provider [4]. Redpanda is technically strong and Kafka-compatible, but GitHub reported no SPDX licence and the project historically uses BSL/source-available terms; mark Excluded-license for core, possible external integration. AutoMQ is important because diskless Kafka on S3/object storage changes the cost model: its README says “Diskless Kafka on S3” and Table Topic integration with Iceberg/S3 Tables; GitHub API reported Apache-2.0 and about 10.9k stars [33]. It should be trialed for cluster cost reduction.

KurrentDB/EventStoreDB is purpose-built for event sourcing. The docs show enterprise licence-key-gated features for KurrentDB 24.10+ [34]. That is acceptable as an external adapter, not the default core. Confluent Schema Registry is explicitly under Confluent Community License for Schema Registry, ksqlDB, REST Proxy, etc., while Apache Kafka remains Apache-2.0 [6]. Therefore Apicurio Registry, Apache-2.0, is the default schema provider [5].

### Time travel and versioned data

XTDB v2 is the most directly aligned bitemporal database: all tables track system time and valid time automatically, with SQL and immutable Apache Arrow/object storage. Its launch post says v2 reached stable SQL/storage and is open source under MPL-2.0 [7]. MPL is not permissive enough for Year96 core preferences, but XTDB’s model should inspire the canonical event schema.

Dolt is “Git for Data”: SQL tables with clone/branch/merge/push/pull semantics, Apache-2.0, about 24.5k stars, and useful for proof datasets and human-inspectable state diffs [8]. Apache Iceberg and Delta Lake are both Apache-2.0 table formats for time-travel analytic projections; Delta’s repo was active on 2026-09-28 [9]. DuckLake is new but notable: its spec requires a SQL catalog database and Parquet data storage, and the GitHub API reported MIT with about 3k stars [10]. It could become the easiest local lakehouse snapshot format. Neon’s Postgres branching is relevant for isolated task worlds and replay verification; GitHub API reported Apache-2.0 and about 23k stars. It is a provider candidate for #09 deterministic test branches, but a hosted service should not become a core dependency.

### Incremental computation and live projections

Feldera/DBSP is the most interesting 2025–2026 incremental compute option for Year96. Its README claims arbitrary SQL programs can be evaluated incrementally, including joins, aggregates, windows, UDFs, and recursive queries, with millions of events/sec on a laptop and consistency with batch results [11]. This maps directly to subscriptions, scope-effect triggers (#02), and “hang until relevant state changes” thread listeners (#03/#04).

RisingWave rebranded its value proposition around “event streaming for agentic AI”: ingest from databases, event streams, webhooks, batch stores; maintain materialized views incrementally; serve SQL at low latency [12]. It may be the pragmatic cluster projection provider when Year96 needs SQL materialized views without assembling Debezium+Kafka+Flink+serving DB. Materialize remains a strong technical reference but source-available/BSL, so not core [13]. Pathway is also explicitly BSL in its GitHub badge [14]. ElectricSQL is Apache-2.0 and useful for Postgres read-path sync to devices [35]. Convex is interesting as a reactive database with TypeScript functions and self-hosting, but its full licence must be checked before core adoption [36]. Rocicorp Zero/Replicache are useful local-first references; terms need verification [37].

### CDC/connectors and external world sensors

Debezium is the default CDC engine: Apache-2.0, low-latency database change capture, and a large connector ecosystem [38]. Airbyte is valuable for SaaS/API ELT and AI-agent data movement, but its connector licensing must be checked per connector; the README positions it as open-source data movement [39]. dlt is an excellent lightweight Python ingestion library for agents: it can load APIs, iterables, SQL, and files into DuckDB/warehouses and is easy for executors to embed [40]. Redpanda Connect/Benthos gives declarative YAML pipelines, Bloblang transforms, and many connectors, with a free Apache-2.0 API surface plus enterprise bundle separation [41]. changedetection.io should be an external-web sensor provider, but I did not fully verify its current licence in this time box.

### Unified search over everything

OpenSearch is Apache-2.0 and a mature text/log/time-series engine; its repo is Linux Foundation/OpenSearch Foundation backed [15]. Vespa is the strongest permissive “one engine” for text, vectors, tensors, structured data, ranking, and low-latency serving; its README says all content is Apache-2.0 and it releases every weekday [16]. Tantivy is MIT and embeddable Rust full-text search; Quickwit is Apache-2.0 in the currently opened README, correcting the common outdated assumption that it is AGPL after acquisition [17][18]. Qdrant is Apache-2.0 and production-ready for vector search with payload filtering [19]. Milvus is Apache-2.0 and best for billion-scale multimodal vectors [42]. Weaviate is open-source vector DB; licence needs direct SPDX verification before core but is often BSD-3 [43]. pgvector keeps vectors inside Postgres with ACID/PITR/JOINs [44]. VectorChord is a newer Postgres extension for compressed large-scale vector search [45]. ParadeDB is very attractive for “just use Postgres” hybrid BM25+vector but the opened README badge shows AGPL-3.0, so mark Excluded-license core [20]. Elasticsearch’s modern ELv2/SSPL/AGPL mix is not default-core material.

### Knowledge graphs and ontologies

Graphiti is the most relevant agent-memory KG in 2026: it builds temporal context graphs that track fact changes, source provenance, prescribed/learned ontology, and historical queries without full recomputation [21]. This overlaps #03; State Fabric should provide raw events, URIs, and provenance, while #03 decides memory compaction and retrieval policy. Kuzu was promising as an embeddable graph DB with vector/full-text indices, but its README now says the KuzuDB project is being archived and prior releases remain usable; do not make it strategic [22]. FalkorDB targets low-latency LLM knowledge graphs and sparse-matrix property graph queries [23], but licensing needs verification. SurrealDB’s README badge shows BSL 1.1, so exclude from core [24]. TypeDB is conceptually strong but licence posture must be verified; assume non-permissive until checked [25]. Apache Jena and Oxigraph are safer RDF/SPARQL providers for standards-based graphs [46][47]. LinkML gives schemas that can emit JSON Schema/RDF/OWL/SQL-ish artifacts and should anchor ontology-as-code [48].

Palantir Foundry Ontology/AIP remains the enterprise reference pattern: objects + links + actions + functions. Year96 should copy the separation of object state, semantic model, actions, and policy, not the product.

### Digital twins, snapshots, and “freeze the world” analogues

DTDL is a JSON-LD/RDF-based digital twin modeling language for physical/logical entities, with v4 supported in Azure IoT Operations [49]. It is a good external ontology input. Eclipse Ditto is EPL-2.0 and provides open digital-twin state management APIs [50]; because EPL is weak copyleft, use as integration. OpenUSD is Apache-2.0 and describes time-sampled 3D scene state, useful for simulation/digital twin blobs [51]. Bevy ECS and flecs show how game/simulation worlds maintain entity-component state and can snapshot/replay; Bevy is MIT/Apache-2.0 and flecs is MIT [52][53].

### Thoughts, provenance, and the system as state

W3C PROV gives the vocabulary Year96 needs: entities, activities, agents, derivation, attribution, and association [27]. OpenTelemetry gives trace/span context and language ecosystem; GenAI semantic conventions moved to a dedicated repository, which means Year96 should pin versions and ingest OTel spans as first-class state, not as separate observability exhaust [28].

For system-as-state, GitOps and configuration-as-data are the pattern. CUE validates and unifies config [31]; KCL is a constraint/record/functional language for configuration [32]; Pkl is useful but the GitHub page failed to render in this time box, so verify separately. OpenTofu is OSS IaC but GitHub page did not show the licence in the fetched range; the assignment memory says MPL-2.0, so mark Excluded-license core until reverified [29]. Pulumi has a visible licence badge and is usually Apache-2.0; use as permissive IaC provider if LICENSE verification passes [30].

### Local-first sync

Automerge, Yjs, Loro, LiveStore, Jazz, and ElectricSQL matter because humans and agents will operate on laptops, phones, browsers, and offline sandboxes. Automerge provides CRDTs, compression, and sync; Yjs is MIT and supports offline editing, snapshots, and shared cursors [54][55]. Loro adds version-controlled JSON CRDTs and imports Git DAG history [56]. LiveStore is Apache-2.0 and explicitly event-sourced local SQLite with sync and materializers [57]. Jazz 2.0 alpha is a local-first relational DB with durable streams/files and auth/sync specs [58]. These should feed edge events into the global fabric rather than replace the canonical log.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| CloudEvents | Standard | Universal event envelope | Apache-2.0 | CNCF graduated Jan 2024; ~5.9k stars; active 2026 | Adopt |
| NATS JetStream | OSS | Laptop/edge event log | Apache-2.0 | ~20.8k stars; active 2026 | Adopt |
| Apache Kafka | OSS | Cluster log backbone | Apache-2.0 | ~33.9k stars; active 2026 | Adopt |
| Apache Pulsar | OSS | Multi-tenant/geo log provider | Apache-2.0 | ~15.3k stars; active 2026 | Trial |
| AutoMQ | OSS/product | Diskless Kafka on S3/object storage | Apache-2.0 (GitHub API) | ~10.9k stars; active 2026 | Trial |
| Redpanda | OSS/product | Kafka-compatible fast broker | BSL/NOASSERTION | ~12.6k stars; active | Excluded-license |
| KurrentDB/EventStoreDB | OSS/product | Purpose-built event store | NOASSERTION/licensed features | ~5.9k stars; enterprise licence keys | Watch |
| Apicurio Registry | OSS | Schema registry | Apache-2.0 | ~939 stars; active | Adopt |
| Confluent Schema Registry | OSS/product | Mature Kafka schemas | Confluent Community License | ~2.5k stars; use-restricted | Excluded-license |
| XTDB v2 | OSS | Bitemporal SQL model | MPL-2.0 | ~3.1k stars; stable v2 | Excluded-license |
| Dolt | OSS | Git-like SQL datasets | Apache-2.0 | ~24.5k stars; active | Trial |
| Apache Iceberg | OSS | Lakehouse time travel tables | Apache-2.0 | ~9.3k stars; active | Adopt |
| Delta Lake | OSS | Lakehouse ACID/time travel | Apache-2.0 | ~9k stars; active | Trial |
| DuckLake | OSS/spec | Local SQL catalog + Parquet lake | MIT | ~3k stars; new | Trial |
| Feldera/DBSP | OSS/product | Incremental SQL projections | MIT (verify before prod) | Rapidly growing; active | Trial |
| RisingWave | OSS/product | Streaming SQL materialized views | Apache-2.0 (verify) | Active; agentic AI positioning | Trial |
| Materialize | Product | Strong live views reference | BSL | Mature but source-available | Excluded-license |
| Pathway | OSS/product | Python streaming/RAG pipelines | BSL (badge) | Active | Excluded-license |
| Debezium | OSS | CDC | Apache-2.0 | Mature; Red Hat ecosystem | Adopt |
| dlt | OSS | Agent-friendly ELT | Apache-2.0 (verify) | Active Python ecosystem | Adopt |
| Redpanda Connect/Benthos | OSS/product | Declarative connectors/transforms | Apache-2.0 free API + enterprise | Active | Trial |
| Vespa | OSS | Hybrid search/ranking | Apache-2.0 | Daily releases; production proven | Adopt |
| OpenSearch | OSS | Text/log/time search | Apache-2.0 | Foundation-backed | Trial |
| Tantivy | OSS library | Embedded full-text | MIT | Rust ecosystem | Adopt |
| Quickwit | OSS/product | Object-storage log search | Apache-2.0 (opened README) | Active | Trial |
| Qdrant | OSS/product | Vector DB | Apache-2.0 | Production-ready | Trial |
| Milvus | OSS | Billion-scale vectors | Apache-2.0 | Large ecosystem | Watch/Trial |
| pgvector | OSS extension | Simple vectors in Postgres | PostgreSQL | Mature extension | Adopt |
| VectorChord | OSS extension | Compressed Postgres vector search | License verify | New | Trial |
| ParadeDB | OSS/product | Postgres BM25+vector | AGPL-3.0 badge | Active | Excluded-license |
| Graphiti | OSS | Temporal agent KG | Apache-2.0 (verify) | Active Zep project | Trial with #03 |
| Kuzu | OSS | Embedded graph DB | Permissive but archived | Archived notice | Avoid |
| SurrealDB | OSS/product | Multi-model DB | BSL-1.1 badge | Popular | Excluded-license |
| LinkML | OSS | Ontology/schema-as-code | MIT (verify) | Mature bio/linked-data use | Adopt |
| Apache Jena | OSS | RDF/SPARQL | Apache-2.0 | Mature Apache project | Trial |
| Oxigraph | OSS | Embedded RDF/SPARQL | MIT/Apache? (verify) | Active Rust | Trial |
| OpenTelemetry | Standard/OSS | Traces/logs/metrics as state | Apache-2.0 | CNCF standard | Adopt |
| W3C PROV | Standard | Provenance model | W3C spec | Mature | Adopt |
| LiveStore | OSS | Local event-sourced SQLite | Apache-2.0 | Active | Trial |
| Automerge/Yjs/Loro | OSS | CRDT local-first states | MIT/permissive | Mature/active | Trial |

## How I would build this part of Year96

### 1. Canonical model

Every durable fact becomes a `StateEvent`. The envelope is CloudEvents-compatible, but the payload is governed by Year96 schemas.

```ts
type Y96Uri = `y96://${string}`;
type Instant = string; // RFC3339/UTC
type Interval = { from: Instant; to?: Instant };
type Sensitivity = 'public' | 'internal' | 'confidential' | 'secret' | 'regulated' | 'personal' | 'thought-private';

type StateEvent<T = unknown> = {
  specversion: '1.0';
  id: string;
  source: Y96Uri;
  subject: Y96Uri;
  type: string;
  time: Instant;
  datacontenttype: 'application/json' | 'application/cbor' | 'application/protobuf';
  dataschema: Y96Uri;
  data?: T;
  blob?: { uri: Y96Uri; mediaType: string; sha256: string; bytes: number };
  y96: {
    validTime: Interval;
    txTime: Instant;
    observedTime?: Instant;
    actor: Y96Uri;
    tenant: Y96Uri; org: Y96Uri;
    thread?: Y96Uri; message?: Y96Uri; thought?: Y96Uri;
    causationId?: string; correlationId?: string; traceparent?: string;
    sequence?: { partition: string; offset: string };
    provenance: ProvRef[];
    sensitivity: Sensitivity[];
    retention: { class: 'ephemeral'|'normal'|'audit'|'legal-hold'; deleteAfter?: Instant };
    acl: { readers: Y96Uri[]; writers?: Y96Uri[]; policyRefs: Y96Uri[] };
    hash: { canonical: string; previousForSubject?: string };
  };
};
```

The URI scheme must address any state:

- `y96://org/acme/identity/human/alice`
- `y96://org/acme/thread/customer-onboarding/meta-memory`
- `y96://org/acme/thought/agent-17/2026-09-28T16:00:00Z/0004`
- `y96://org/acme/system/git/year96/commit/abc123`
- `y96://org/acme/blob/sha256/...`
- `y96://org/acme/world-snapshot/2026-09-28T16:00:00Z`

Thoughts are not hidden logs. They are private/sensitive state events: `y96.identity.thought.started`, `y96.identity.intention.set`, `y96.agent.plan.revised`, `y96.agent.critique.recorded`. Communications (#04) emit messages and thread events. Memory (#03) is a projection over messages, thoughts, artifacts, and milestones. Identity (#06) emits permission/capability/policy events. Self-improvement (#08) emits code/config/prompt/evaluation/release events. Verification (#09) emits before/after snapshots, expected state predicates, traces, and proof verdicts.

### 2. Provider interfaces

```ts
interface EventLogProvider {
  append(events: StateEvent[], opts: { expectedSubjectVersion?: string }): Promise<AppendReceipt>;
  subscribe(query: EventSubscription, handler: (e: StateEvent) => Promise<void>): Promise<Subscription>;
  read(range: LogRange): AsyncIterable<StateEvent>;
  getHighWatermarks(): Promise<Record<string, string>>;
}

interface SchemaRegistryProvider {
  register(schema: StateSchema): Promise<Y96Uri>;
  resolve(uri: Y96Uri): Promise<StateSchema>;
  validate(event: StateEvent): Promise<ValidationResult>;
  compatibility(oldUri: Y96Uri, next: StateSchema): Promise<CompatibilityReport>;
}

interface ProjectionProvider<T = unknown> {
  name: string;
  rebuild(from: WorldSnapshotRef): Promise<void>;
  apply(event: StateEvent): Promise<void>;
  checkpoint(): Promise<ProjectionCheckpoint>;
  query(q: unknown, auth: AuthContext): Promise<T>;
}

interface SearchIndexProvider {
  index(event: StateEvent, docs: SearchDoc[]): Promise<void>;
  hybridSearch(q: SearchQuery, auth: AuthContext): Promise<SearchResult[]>;
  deleteOrRedact(subject: Y96Uri, policy: RedactionPolicy): Promise<void>;
}

interface SnapshotProvider {
  freeze(opts: FreezeRequest): Promise<WorldSnapshot>;
  checkout(snapshot: Y96Uri, opts: CheckoutRequest): Promise<CheckoutHandle>;
  diff(a: Y96Uri, b: Y96Uri): Promise<WorldDiff>;
}

interface ConnectorProvider {
  spec(): ConnectorSpec;
  start(cursor?: unknown): AsyncIterable<StateEvent>;
  health(): Promise<Health>;
}

interface BlobStoreProvider {
  put(bytes: AsyncIterable<Uint8Array>, meta: BlobMeta): Promise<BlobRef>;
  get(ref: BlobRef, auth: AuthContext): Promise<ReadableStream>;
  seal(ref: BlobRef): Promise<{ sha256: string; size: number }>;
}

interface StateQuery {
  get(uri: Y96Uri, asOf?: { validTime?: Instant; txTime?: Instant }): Promise<StateObject>;
  events(q: EventQuery): AsyncIterable<StateEvent>;
  sql<T>(query: string, params: unknown[], asOf?: TimePoint): Promise<T[]>;
  vector(q: VectorQuery): Promise<SearchResult[]>;
  graph(q: GraphQuery): Promise<GraphResult>;
  search(q: SearchQuery): Promise<SearchResult[]>;
}
```

### 3. Log topology

Use topic families rather than one global topic:

- `state.raw.external.*`: CDC, SaaS, web, email/calendar, market/GDELT/RSS, browser capture.
- `state.raw.internal.*`: messages, thoughts, runtime events, permissions, prompts, code/config.
- `state.validated.*`: schema-valid canonical events after policy and dedupe.
- `state.projection.commands`: rebuild/checkpoint/freeze instructions.
- `state.audit.proofs`: verifier and snapshot proofs.

Partition by `(org, sensitivity-domain, subject-hash)` so per-subject order is stable while multi-tenant isolation remains possible. Every append path runs: authenticate actor (#06), validate schema, attach OTel trace, calculate canonical hash, write blob first if needed, append event, fan out to projections.

### 4. Projections and query stack

Laptop MVP:

- NATS JetStream log.
- SQLite/DuckDB/DuckLake files for analytic snapshots.
- Postgres if available, with pgvector/VectorChord for vectors.
- Tantivy for embedded full text.
- Oxigraph or simple RDF files for semantic graph.
- Apicurio in container or local JSON Schema registry.
- LiveStore/ElectricSQL for UI/device sync.

Cluster scale:

- Kafka or Pulsar log; trial AutoMQ for object-storage-backed Kafka.
- Object store (S3-compatible) for blobs and lakehouse tables.
- Iceberg as canonical analytic table format; Delta optional.
- Feldera/RisingWave for live SQL materialized views.
- Vespa as primary hybrid retrieval/ranking engine; OpenSearch for logs if org already uses it; Qdrant/Milvus when vector scale exceeds Vespa/Postgres ergonomics.
- Jena/Oxigraph/Graphiti-backed temporal KG for semantic and memory projections.
- Debezium, dlt, Redpanda Connect, Airbyte-compatible connectors.

SQL + vector + graph + time querying should not be one magic database. `StateQuery` composes results: SQL narrows candidates by tenant/thread/time/policy; vector/text ranks content; graph expands relationships/provenance; bitemporal filters apply at event/projection layer. Vespa can serve the main online hybrid ranker; DuckDB/Iceberg handles offline time-travel analytics; Postgres remains the transactional control plane.

### 5. World freeze and replay

A `WorldSnapshot` manifest contains:

```ts
type WorldSnapshot = {
  uri: Y96Uri;
  createdAt: Instant;
  requestedBy: Y96Uri;
  logCuts: Record<string, string>;
  projectionCheckpoints: ProjectionCheckpoint[];
  blobMerkleRoot: string;
  schemaVersionSet: Y96Uri[];
  codeConfigRefs: Y96Uri[];
  encryptionKeyEpochs: Record<string, string>;
  proof: { hash: string; signatureRefs: Y96Uri[] };
};
```

Freeze algorithm: pause/mark writes per partition, collect high-watermarks, flush projections to checkpoints, seal in-flight blobs, record schema/config refs, emit `y96.world.freeze.created`. For large clusters, use Chandy-Lamport-style markers: inject barrier events into each partition; projections checkpoint when all input barriers arrive. Replay for #09 checks out a snapshot into an isolated environment, replays events to a target cut, runs stateless functions with deterministic clocks/model mocks, and compares emitted events/projections to expected predicates.

### 6. Privacy, retention, and immutable logs

Do not promise physical deletion from immutable backups. Instead: encrypt payload/blob keys per subject/sensitivity/retention class; store only ciphertext in logs; crypto-shred by destroying data encryption keys; emit redaction events; rebuild projections and search indexes; keep minimal tombstone/proof metadata. Sensitivity labels drive routing: `thought-private` thoughts may be visible only to the owning identity and approved verifiers; regulated data may have regional topics and retention. Multi-tenant isolation is topic/account/bucket/key separation plus policy checks in every provider.

### 7. Testing and proof

For the 70% rule, State Fabric needs:

- Envelope schema golden tests and compatibility tests.
- Property tests for canonical hashing and URI normalization.
- Event-log contract tests run against NATS, Kafka, Pulsar providers.
- Projection determinism tests: rebuild from log must equal incremental apply.
- Snapshot consistency tests with concurrent writers and injected failures.
- Replay tests with frozen clocks, model/tool mocks, and known expected state.
- Privacy tests: redacted subjects disappear from search/vector/graph projections after key shredding and rebuild.
- Agentic verifier tests (#09): before/after state predicates for every Year96 task.

## What is still unsolved (late 2026)

- **Consistent global cuts across arbitrary SaaS APIs and the physical world** are approximate. Year96 can freeze its observations and connectors’ cursors, not the actual universe.
- **Thought capture vs. privacy/safety** is unresolved. Persisting reasoning traces improves debugging and scope-effect prediction but can expose secrets, sensitive mental models, and legally discoverable material.
- **Cross-modal canonicalization** is hard: video, browser DOM, 3D scenes, emails, code, and thoughts need stable identifiers, hashes, embeddings, and provenance without losing meaning.
- **One query language for SQL + vector + graph + bitemporal provenance** does not exist in a mature permissive package. Provider composition is safer than betting on a universal DB.
- **Schema evolution for agent-generated events** will be chaotic. Year96 must invest in schema linting, compatibility gates, upcasters, deprecation workflows, and quarantine topics.
- **Right-to-be-forgotten vs. audit logs** requires legal/product decisions, not only crypto-shredding. Some proofs must survive while content vanishes.
- **Deterministic replay of LLM agents** is not solved unless model calls are recorded/mocked or constrained to deterministic local models. Coordinate tightly with #09.
- **Cost control** for “everything is indexed every way” needs adaptive indexing: not every event deserves embeddings, graph extraction, and indefinite hot storage.

## Sources

1. https://github.com/cloudevents/spec
2. https://github.com/apache/kafka
3. https://github.com/nats-io/nats-server
4. https://github.com/apache/pulsar
5. https://github.com/apicurio/apicurio-registry
6. https://www.confluent.io/confluent-community-license-faq/
7. https://xtdb.com/blog/launching-xtdb-v2
8. https://github.com/dolthub/dolt
9. https://github.com/delta-io/delta
10. https://ducklake.select/docs/stable/specification/introduction.html and https://github.com/duckdb/ducklake
11. https://github.com/feldera/feldera
12. https://github.com/risingwavelabs/risingwave
13. https://github.com/MaterializeInc/materialize
14. https://github.com/pathwaycom/pathway
15. https://github.com/opensearch-project/OpenSearch
16. https://github.com/vespa-engine/vespa
17. https://github.com/quickwit-oss/tantivy
18. https://github.com/quickwit-oss/quickwit
19. https://github.com/qdrant/qdrant
20. https://github.com/paradedb/paradedb
21. https://github.com/getzep/graphiti
22. https://github.com/kuzudb/kuzu
23. https://github.com/FalkorDB/FalkorDB
24. https://github.com/surrealdb/surrealdb
25. https://github.com/typedb/typedb
26. https://github.com/open-telemetry/opentelemetry-specification
27. https://www.w3.org/TR/prov-overview/
28. https://opentelemetry.io/docs/specs/semconv/gen-ai/
29. https://github.com/opentofu/opentofu
30. https://github.com/pulumi/pulumi
31. https://github.com/cue-lang/cue
32. https://github.com/kcl-lang/kcl
33. https://github.com/AutoMQ/automq
34. https://docs.kurrent.io/server/v25.0/quick-start/installation.html
35. https://github.com/electric-sql/electric
36. https://github.com/get-convex/convex-backend
37. https://github.com/rocicorp/mono
38. https://github.com/debezium/debezium
39. https://github.com/airbytehq/airbyte
40. https://github.com/dlt-hub/dlt
41. https://github.com/redpanda-data/connect
42. https://github.com/milvus-io/milvus
43. https://github.com/weaviate/weaviate
44. https://github.com/pgvector/pgvector
45. https://github.com/tensorchord/VectorChord
46. https://github.com/apache/jena
47. https://github.com/oxigraph/oxigraph
48. https://github.com/linkml/linkml
49. https://github.com/Azure/opendigitaltwins-dtdl
50. https://github.com/eclipse-ditto/ditto
51. https://github.com/PixarAnimationStudios/OpenUSD
52. https://github.com/bevyengine/bevy
53. https://github.com/SanderMertens/flecs
54. https://github.com/automerge/automerge
55. https://github.com/yjs/yjs
56. https://github.com/loro-dev/loro
57. https://github.com/livestorejs/livestore
58. https://github.com/garden-co/jazz
