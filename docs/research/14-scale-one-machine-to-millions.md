# 14 — One codebase from one machine to millions of agents

Scope: this report designs the deployability rule added to the spec on 2026-09-28: Year96 code must run on one machine and also scale across servers for millions of agents. I interpret "millions of agents" as mostly millions of **logical agents** (identities, threads, clones, sensors, listeners and dormant virtual actors) plus a much smaller set of **active sessions** that simultaneously hold model, sandbox, database, queue and human-attention capacity. The task is therefore not "one Kubernetes cluster with millions of pods"; it is a provider-based codebase whose topology is configuration, with cells, partition keys, leases, effect ledgers and proof gates making location transparent.

## TL;DR for the Year96 architect

- The key framing is correct: millions of Year96 agents are mostly **logical/dormant** records and virtual actors. Cloudflare says Durable Objects can be created in the millions, are implicitly created, hibernate idle, and have no hard namespace object-count limit [14]. Temporal says it has no concurrent workflow-execution count limit but each workflow history is capped at 51,200 events / 50 MB [7]. Infrastructure can hold logical agents; **active LLM+sandbox sessions bind first**.
- Build **one artifact**: `year96` with role flags (`--roles=all`, `--roles=kernel,actor-host`, etc.). Grafana Mimir and Loki validate the pattern: one binary can run `-target=all` for monolith/dev and component targets for microservices/scale [2][3]. Temporal validates process-role separation with frontend/history/matching/worker services and fixed history shards [8].
- Four run profiles are enough: `sim` (single process, in-memory/PGlite providers, virtual clock, mocked models, deterministic simulation), `solo` (one box, all roles plus Postgres/NATS/Temporal dev server), `cluster` (Kubernetes, same binary per role, Helm), `fleet` (many cells across regions with thin global control plane).
- Make the **deployability contract** enforceable: every handler declares a partition key; every state change enters through the kernel; no package may keep authoritative state in process memory; every port has in-process and network transports; actors are addressable only by `y96://` or `ActorRef`; every handler is idempotent and bounded; resource classes are scheduled and budgeted.
- Use **cells** as the primary scale unit. AWS Well-Architected states cells reduce blast radius by isolating workload partitions [10]. Slack moved critical services to a cell-based architecture after AZ gray failures and designed drains to remove as much traffic from an AZ in 5 minutes, incrementally [11]. Year96 should place orgs into cells, not spread one org's hot aggregates globally unless forced.
- `cluster` is not enough for "millions". Kubernetes v1.37 targets at most 5,000 nodes, 110 pods/node, 150,000 pods and 300,000 containers per cluster [22]. Millions of concurrent **sandboxes** require multiple clusters/cells or Google-like Agent Substrate, whose 2026 blog says normal Kubernetes is optimized for thousands of long-running services, not millions of sub-second agent tool calls [23].
- The MVP Postgres actor host scales by sharding orgs and actor IDs; it should use **single-writer leases per actor/aggregate** plus mailbox partitions. Consistent hashing chooses a host; the lease row proves ownership. On rebalance, hosts stop acquiring new leases, drain mailboxes, then release leases.
- Temporal is still **Builders only**. It is MIT [35], mature, and Cloud limits document namespace APS/RPS/OPS, schedule RPS and throttling [9]; but Workflow history limits [7] and fixed `numHistoryShards` [8] make it a bad backing store for never-ending Thread actors.
- Messaging path: NATS JetStream for solo/small cells, Kafka for large distribution/analytics. NATS supports clusters, RAFT/quorum stream replication and leaf nodes that dial out and bridge subject interest [12][13]. Kafka is excellent once partitions are planned; partitions are the scale dial and operational liability.
- Postgres scale-out: core should be **app-level sharding by `(org_id, aggregate_hash)`** over ordinary Postgres. PgBouncer mitigates connections but not failover/load-balancing by itself [21]. Citus is AGPL-3.0 and excluded from core [19]. YugabyteDB core is Apache-2.0 but management is Polyform Free Trial; trial as external provider only [20]. CockroachDB licensing is use-restricted/changed; avoid core.
- Identity: do **not** mint a SPIFFE SVID per logical agent. SPIFFE identifies software workloads and issues short-lived SVIDs to workloads through the Workload API [25]. Pods/services get SVIDs; logical agents get kernel-minted Biscuit-style capabilities bound to workload identity, thread, org, budget and revocation epoch.
- Capacity model: for 10M logical agents / 1% active = 100k active sessions. If each active session averages 0.02 LLM calls/s, 2k calls/s; at 2k input + 500 output tokens, ~5M tokens/s. At an estimated 1.5k-2k tok/s per H100-class 70B FP8 server, that is thousands of H100s unless most calls go to frontier APIs/smaller models and prefix caches. Model capacity and spend dominate.
- Observability must be per-cell: OTel says sampling is appropriate at 1000+ traces/s and high-volume systems often need 1% or lower representative samples [27]. ClickHouse positions its OTel-based observability stack from single node to multi-petabyte scale [28]. Enforce cardinality budgets; never tag every span with raw agent/thread ID in hot metrics.

## Landscape

**Role-flag modular monoliths (2024–2026).** Grafana Mimir documents monolithic mode (`-target=all`) and microservices mode with independently scalable components [2]. Loki explicitly says all microservices live in one binary, switched by `-target`, with monolithic, simple-scalable and microservices deployment modes [3]. Grafana's AGPL license excludes these projects from Year96 core, but the pattern is ideal: topology is config, not a forked codebase.

**Temporal server and Cloud (2026).** Temporal's open-source server license is MIT [35]. The History Service partitions executions into a fixed number of history shards chosen at cluster creation and owns workflow lifecycle/timer state per shard [8]. Temporal Cloud documents namespace action/request/operation rate limits, schedule RPS limits and throttling behavior [9]. It validates durable execution and role separation, but also validates why Year96 actors should not be infinitely-long workflows: 51,200-event / 50 MB history cap and pending-operation caps per workflow [7].

**Single-process / embedded precedents.** PGlite is Apache-2.0 [36] and gives a Postgres-compatible WASM/embedded provider for `sim`/`lite` tests; NATS Server is Apache-2.0 [37] and can be embedded or run as a local process for solo. FoundationDB's single-threaded deterministic simulation ran whole clusters with clocks, networks and disks modeled, tens of thousands of nightly simulations and an estimated trillion CPU-hours [29]. TigerBeetle's VOPR stubs clock/network/disk, uses deterministic seeds plus Git commit, accelerates time, injects dropped/reordered packets and corrupt disks, and replays exact failures [30]. These are stronger precedents for Year96 `sim` than a toy in-memory unit test.

**Cellular architecture.** AWS Well-Architected says cells are independent workload partitions that reduce failure impact and improve predictability/testability [10]. Slack's migration shows why: a single AZ gray failure caused user-visible errors due to hundreds of backend RPCs and strong-consistency primaries; their response was cell/zone drain with fast, incremental traffic movement [11]. Year96 should design cells around org placement, data residency and blast-radius budgets.

**Virtual actors.** Dapr Actors implement the virtual actor pattern with actor IDs, on-demand activation, single-threaded message processing and suitability for thousands or more independent isolated units [15]. Cloudflare Durable Objects provide globally named, single-threaded stateful instances with colocated durable storage, implicit first-access creation, idle hibernation, millions of objects and no hard per-namespace object-count limit [14]. Orleans/Halo validated virtual actors at game scale; primary papers and talks report Orleans powering Halo 4 services for millions of players, but Year96 should adopt the pattern, not necessarily the .NET runtime.

**Agent sandboxes at scale.** Kubernetes itself documents large-cluster limits: 5,000 nodes, 110 pods/node, 150,000 pods, 300,000 containers [22]. GKE Agent Sandbox provides Sandbox/SandboxClaim/SandboxTemplate/WarmPool CRDs, default-deny networking, gVisor/Kata isolation, Pod snapshots and sub-second provisioning [24]. Google's 2026 Agent Sandbox/Agent Substrate post says GKE warm pools can allocate 300 sandboxes/s/cluster with 90% in 200 ms, and Agent Substrate exists because millions of sub-second agent tool calls would overwhelm normal Kubernetes control planes [23]. This confirms the assignment's lead: Google Agent Substrate exists and should be watched/trialed.

**Messaging and buses.** NATS clustering explains routes, RAFT groups, quorum writes and placement [12]; leaf nodes bridge subject interest over outbound-only links, useful for edge/private cells [13]. NATS is Apache-2.0 [37]. Kafka remains the production choice for large retention/distribution, but partition count and cross-region replication planning become operational constraints; Year96 should hide bus choice behind `MessageBusProvider` and declare per-subject partition keys.

**Authorization and identity.** Zanzibar scaled to trillions of ACLs and millions of authz checks/s with p95 under 10 ms and >99.999% availability at Google [26]. OpenFGA is Apache-2.0 [38] and runs with Postgres [16]; SpiceDB is Apache-2.0 [39] and explicitly implements Zanzibar-style concepts [17]. SPIFFE is for workloads, not per-user/per-agent logical principals: SVIDs are short-lived cryptographic documents issued to workloads through the Workload API [25]. Year96 should bind logical agent capabilities to workload SVIDs.

**Postgres and scale-out.** App-level sharding over ordinary Postgres is the safest permissive path. Citus is AGPL-3.0 and excluded [19]. YugabyteDB core is Apache-2.0 but its management/orchestration platform is Polyform Free Trial, so it is trial/external rather than default [20]. PgBouncer is mandatory to avoid connection storms, but its FAQ is explicit: no internal multi-host load balancing or failover; use DNS/LVS/HAProxy/external failover and reconnect [21].

**Observability.** OTel sampling docs justify tail/head sampling for high-volume traces and explicitly mention 1000+ traces/s and 1% or lower samples for high-volume systems [27]. ClickHouse's observability stack claims single-node to multi-petabyte scale on OpenTelemetry data [28]. Year96 should put ClickHouse per cell, export sampled traces globally, and keep audit/proof events in the ledger.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| Year96 role-flag binary | internal design | One artifact; topology by `--roles` and providers | internal | Must be first-class architecture rule | Adopt |
| Temporal | OSS | Builder workflows, timers, activities, schedules; role-separated services | MIT verified [35] | Mature Temporal-backed; Cloud docs active 2026 | Adopt for Builders |
| Grafana Mimir/Loki pattern | OSS pattern | `-target=all` vs microservices precedent | AGPL family; Excluded-license (core) | Mature Grafana | Adopt pattern only |
| PGlite | OSS | Embedded Postgres-compatible provider for sim/lite | Apache-2.0 verified [36] | Active ElectricSQL | Trial |
| NATS/JetStream | OSS | Solo/cell bus, RAFT streams, leaf nodes | Apache-2.0 verified [37] | Mature CNCF ecosystem | Adopt |
| Kafka | OSS | Large retention/distribution bus | Apache-2.0 | Mature | Adopt cluster/fleet |
| FoundationDB Simulation | OSS technique | Deterministic single-process cluster testing | Apache-2.0 verified [40] | Apple-backed; proven | Adopt technique |
| TigerBeetle VOPR | OSS technique | Deterministic fault simulation with seed replay | Apache-2.0 (verified separately by repo) | Active | Adopt technique |
| AWS Cell-based guidance | Cloud guidance | Blast-radius/cell design | N/A | AWS Well-Architected | Adopt pattern |
| Slack cellular architecture | Engineering precedent | AZ/cell drain, gray-failure lessons | N/A | Production Slack | Adopt pattern |
| Dapr Actors | OSS | K8s virtual actor provider | Apache-2.0 verified [41] | CNCF | Trial/Adopt cluster |
| Cloudflare Durable Objects | Product + OSS SDK | Hosted millions of hibernating objects | Platform proprietary; SDK permissive | Cloudflare production | Trial external |
| GKE Agent Sandbox | OSS/product | Warm sandbox pools, snapshots, default deny | Apache-2.0 verified for k8s-sigs [42] | GA 2026 on GKE | Trial |
| Agent Substrate | OSS | Ultra-dense agent control plane beyond K8s | Apache-2.0 verified [43] | New 2026 | Watch/Trial |
| Kubernetes | OSS | Cluster substrate and limits | Apache-2.0 | Mature | Adopt, with cells |
| OpenFGA | OSS | ReBAC checks, Postgres backend | Apache-2.0 verified [38] | CNCF/Auth0 lineage | Adopt |
| SpiceDB | OSS | Zanzibar-compatible authz option | Apache-2.0 verified [39] | Authzed; 7k+ stars shown in docs [17] | Trial |
| SPIFFE/SPIRE | Standard/OSS | Workload identity, federation | Apache-2.0 (SPIRE) | CNCF graduated | Adopt |
| Citus | OSS | Postgres sharding extension | AGPL-3.0 verified [19] | Mature | Excluded-license |
| YugabyteDB | OSS/source-available mix | Distributed Postgres-like provider | Core Apache-2.0; Anywhere Polyform Free Trial [20] | Mature vendor | Trial external |
| CockroachDB | OSS/source-available | Distributed SQL | BSL/changed license (use-restricted; verify before use) | Mature vendor | Excluded-license core |
| PgBouncer | OSS | Connection pooling | ISC-style (known; not reverified here) | Mature | Adopt |
| vLLM | OSS | High-throughput local inference, prefix cache | Apache-2.0 verified [44] | Very active | Adopt |
| SGLang | OSS | Alternative high-throughput serving | Apache-2.0 verified [45] | Active | Trial |
| OpenTelemetry | Standard/OSS | Traces/metrics/logs and sampling | Apache-2.0 | CNCF | Adopt |
| ClickHouse | OSS/product | Per-cell observability store | Apache-2.0 | Mature | Adopt |

## How I would build this part of Year96

### 1. Run profiles for one artifact

`year96` is one service artifact plus workers. It always loads the same packages and provider interfaces; profiles select providers and roles.

| Profile | Shape | Providers | What differs |
|---|---|---|---|
| `sim` | One process, no external network by default | In-memory or PGlite event log; in-process bus; in-memory actor host; virtual clock; deterministic scheduler; mocked model gateway; fake sandbox provider; seeded RNG | Only providers. Used for 10M dormant actors, wake storms, replay, model/tool fault injection. FDB/TigerBeetle-style determinism. |
| `solo` | One machine | `year96 --roles=all`; local Postgres; local NATS; Temporal dev server; OpenFGA; LiteLLM; local Docker/gVisor; OTel collector | Real persistence/effects but all roles together. Good for developer and small org. |
| `cluster` | One Kubernetes cluster/cell | Same binary deployed multiple times with `--roles=kernel`, `--roles=actor-host`, `--roles=hub`, etc.; sharded Postgres; NATS/Kafka; Temporal cluster; Dapr Actors optional; KEDA; sandbox warm pools; Envoy AI Gateway | Roles become separate Deployments/StatefulSets. Providers are networked. Helm owns topology. |
| `fleet` | Many cells/clusters/regions | Cell-local stack plus global control plane | Global plane holds org directory, cell map, identity federation roots, model-routing policy, billing/value rollups and migration control. Org data stays cell-local. |

This follows Mimir/Loki's monolith-to-microservices shape without copying their licenses [2][3]. The artifact must boot with `--roles=all` forever; otherwise regressions will silently break `solo` and `sim`.

### 2. Deployability contract, enforceable rules

```ts
type PartitionKey = `${OrgId}:${ShardId}`;
type ActorRef = { uri: `y96://org/${string}/actor/${string}`; partition: PartitionKey };
type TransportKind = 'inproc' | 'grpc' | 'nats' | 'kafka';

interface Y96Port<Req, Res> {
  readonly name: string;
  readonly transports: readonly TransportKind[]; // every core port has inproc + one network transport
  handle(req: Req, ctx: HandlerContext): Promise<Res>;
}
interface HandlerContext {
  orgId: OrgId; partitionKey: PartitionKey; actor?: ActorRef; intentId: string;
  deadline: Deadline; idempotencyKey: string; traceparent: string;
  resources: ResourceBudget; // tokens, GPU, sandbox, DB writes, human attention
}
```

Rules and enforcement:

1. **Location transparency.** Packages may address agents, threads and state only through `y96://` URIs or `ActorRef`, never hostnames. Enforce with branded types, lint rules banning raw service URLs in domain packages, and conformance tests moving actors between hosts.
2. **Two transports per port.** Every core port ships an in-process transport for `sim/solo` and a network transport for `cluster/fleet`. Enforce by provider contract tests that run the same suite against both.
3. **No authoritative in-memory singletons.** In-memory caches must be reconstructable and TTL-bound. Enforce with static lint banning mutable module-level state outside approved cache modules, plus restart/replay tests.
4. **Partition key on every handler.** No command, event, message, timer, query or effect dispatch without `org_id` and partition key. Enforce in schemas and reject at kernel ingress.
5. **Single writer per aggregate.** Writes go through kernel commit service; actor host must hold a lease row for actor/aggregate before processing. Enforce with DB constraints, lease tests and concurrent writer property tests.
6. **Idempotent handlers.** Every effect/activity/message has `intentId` and `idempotencyKey`; consumers dedupe by event ID. Enforce by replaying duplicated messages in conformance tests.
7. **Bounded queues and admission control.** Every queue has max depth, priority, backpressure and overload behavior. Enforce through load tests that prove p95 and shed policy.
8. **Per-shard ordering only.** Code must not assume global ordering. Enforce by simulator reordering cross-partition events.
9. **Config as state.** Topology, cell map, role assignments, limits and routing policy are ledgered config, not code branches. Enforce by disallowing environment-only hidden topology in production.
10. **Resources are scheduled.** Tokens, GPUs, sandboxes, DB writes, queue slots and human attention are `ResourceClass` objects debited from budgets before execution. Enforce by kernel refusing missing reservations.

### 3. Scale-out table per component

| Component | Single-machine form | Scale-out mechanism | Partition key | Known limit | Bottleneck and mitigation |
|---|---|---|---|---|---|
| Kernel commit service | In-process role + Postgres | Stateless replicas per cell; serializable tx per shard | `(org, aggregate_hash)` | One primary's write IOPS | Shard orgs; batch inserts; keep tx small; sagas across aggregates |
| Ledger | Local Postgres | Postgres shards by org/hash; WAL archive; replicas | org + aggregate | Primary write throughput | More shards/cells; hot aggregate splitting; append-only narrow rows |
| Outbox/bus | NATS JetStream | NATS cluster/leaf; Kafka for large fanout | subject partition | RAFT quorum / partition count | Backpressure; compact topics; per-org subjects |
| Actor host | In-process leases | Consistent hash to host + lease table; Dapr provider later | actor_id | Hot actor serial throughput | Split actors by child aggregates; coalesce wakes; digest |
| Timers | DB table + virtual clock | Timer wheels per shard; jitter; Temporal schedules only for Builders | owner shard | Wake storms | Bucket timers; random jitter; token buckets |
| Temporal | Dev server | Temporal cluster; namespaces/task queues/cells | workflow_id/task_queue | 51,200 events/50 MB history; fixed shards [7][8] | Continue-as-new; Builders only; choose shards upfront |
| Authorization | OpenFGA local | HA OpenFGA/SpiceDB per cell; cached caveated checks | org/resource | Zanzibar target p95 <10ms at huge scale [26] | Cache tuples; colocate with ledger; snapshot model IDs |
| Scope-effect stages | Local workers | Stream processors per org/cell; ML batch jobs | org/thread/identity | LLM adjudicator cost | Cheap cascade first; budgets; top-k only |
| Model gateway | LiteLLM local | Envoy AI Gateway + vLLM/SGLang pools/API routes | org/identity/model | GPU/API quota | Prefix cache; smaller models; admission control |
| pi/sandbox workers | Docker local | Warm pools, GKE Agent Sandbox, Firecracker/Kata pools | org/session | 300 allocations/s/cluster cited by Google [23] | Pools per cell; snapshots; queue starts |
| Search | Postgres FTS/vector | Vespa/OpenSearch/Qdrant per cell | org/index shard | Index/storage cost | Adaptive indexing; cold tiers |
| World-feed fetchers | hermes/local cron | KEDA-scaled fetchers, leaf cells, global fetch dedupe | source/org | Provider rate limits | Shared public mirror where legal; jitter |
| Observability | OTel + local ClickHouse/Jaeger | OTel collectors, tail sampling, ClickHouse per cell | cell/service/tenant | cardinality/cost | Cardinality budgets; sample traces; audit in ledger |
| Identity | local keys/SPIRE dev | SPIRE federation per cell/workload | workload trust domain | SVID sprawl if per agent | SVID per workload only; Biscuit per logical agent |

### 4. Cell design

A Year96 **cell** is the smallest independently survivable production unit: kernel replicas, ledger shards, outbox/bus, actor hosts, hub/thread services, scope workers, authz, model gateway, sandbox pools, feed fetchers, observability and local proof workers. It owns a set of orgs and their data residency region.

**Sizing.** Start with conservative cells: e.g. 2k-20k active sessions, 0.5M-5M logical agents, 10k-50k events/s depending on Postgres and model mix. The exact number must be load-tested; cell size is a product of DB write capacity, LLM budget, sandbox warm-pool size and blast-radius appetite, not a fixed constant.

**Global control plane.** Keep it thin: org directory (`org -> cell`), identity federation root, global policy bundle references, model-routing policy, billing/value aggregation, license ledger, and migration controller. It must not sit on the hot write path for org events.

**Routing.** Requests enter a global router that resolves org to cell and forwards to cell ingress. Internal actor messages never cross cells except through federation. Cross-cell communication is treated as A2A/external federation: signed messages, explicit capabilities, no shared transactions.

**Migration.** Freeze org in source cell; fence new writes; export ledger cut, blobs, timers, actor leases and Temporal workflow refs; import to target; rebuild projections; run replay/verifier checks; flip org directory; drain source. For large orgs, use dual-write only for routing metadata; state remains single-writer with a cutover fence.

**Residency.** Cell placement is the residency boundary. Org ledger, blobs, keys, OpenFGA tuples, search indexes and observability raw logs stay in-region. Global plane sees summaries and signed aggregates.

### 5. Capacity model: 10M logical agents / 1% active

All numbers below are estimates unless marked sourced.

| Item | Assumption/arithmetic | Result |
|---|---:|---:|
| Logical agents | Given | 10,000,000 |
| Active sessions | 1% active | 100,000 |
| Dormant storage | 4 KB actor row + 20 KB thread/meta pointers + 100 KB average compressed recent memory = 124 KB | ~1.24 TB hot metadata; raw history/blobs much larger and tiered |
| Dormant timer/watch rows | 2 rows/agent × 200 B | ~4 GB |
| Wake rate normal | 0.1% dormant wakes/hour = 9,900/hour | 2.75 wakes/s |
| Wake storm drill | 10% logical agents receive same event over 10 min | 1M candidates / 600s = 1,667 wake candidates/s before dedupe/coalesce |
| Events per active session | 0.2 ledger events/s average | 20k events/s |
| Bus messages | 3 messages/event | 60k msg/s |
| LLM calls | 0.02 calls/s/active session | 2,000 calls/s |
| Tokens/call | 2,000 input + 500 output | 5M tokens/s total processed/generated |
| GPU capacity | 70B FP8 H100 estimates ~1.5k-2k tok/s from 2026 benchmark reports (not primary; treat as estimate); prefix cache reduces prefill only [31] | 2,500-3,400 H100-equivalent if all local 70B; far fewer with small models/API/prefix hits |
| Sandboxes active | 20% active sessions need sandbox lease | 20,000 sandboxes |
| Sandbox allocation surge | Google sourced 300 allocations/s/cluster and 90% in 200ms [23] | 20k cold starts needs many minutes on one cluster; prewarm and cells required |
| Kubernetes capacity | 150k pods/cluster sourced [22] | Concurrent sandboxes fit one large cluster on paper, but blast radius/control-plane argues cells |
| Human attention | 0.5% active sessions interrupt humans/hour | 500/h; must be budgeted/ranked |
| Authz checks | 10 checks/event | 200k checks/s; Zanzibar precedent millions/s [26] |

The binding constraint is not actor records. It is LLM tokens, GPU/API spend, sandbox starts and human attention. Therefore the architecture must optimize: cheaper cascade before LLM, prefix/prompt caching, small specialized models, admission control, and graceful degradation.

### 6. Proof plan: the 70% rule

- **Same suites in every profile.** Unit/conformance/E2E tests run in `sim`, `solo`, `cluster` and at least two-cell `fleet`. A provider cannot ship unless the same behavioral tests pass in in-process and network mode.
- **10M dormant actor sim.** `sim` seeds 10M actors with virtual clock and synthetic metadata; prove memory stays bounded, wake selection is deterministic, and replay by seed is exact.
- **Linear scale test.** Add cells from 1→2→4→8 with fixed active sessions/cell. p95 kernel commit, actor wake and cost/active-agent must stay flat within tolerance. If not, the global plane is leaking onto the hot path.
- **Wake-storm drill.** Fire one external event that matches 10% of actors. Expected: candidates are coalesced, per-org token buckets engage, high-score wakes proceed, low-score routes become digests, no DB/bus meltdown.
- **Chaos.** Kill actor hosts while leases are held; partition NATS/Kafka; throttle Temporal; exhaust model quota; revoke workload SVID; restart Postgres primary; force cell outage. Verify no duplicate effects and every ambiguous effect is `unknown` until observe-back.
- **Cell failover/migration drill.** Quarterly export/replay a test org between cells and prove signed ledger equivalence, projection rebuild and DNS/router flip.
- **Scale regression CI.** Microbenchmarks for handler allocations, per-event DB writes, authz cache hit rates, queue depth, trace cardinality and actor-passivation cost fail PRs on regression.

### 7. Concrete changes for the architecture document

1. Replace §9 with four explicit profiles: `sim`, `solo`, `cluster`, `fleet`; state that topology is provider config, not code.
2. Add a "Deployability Contract" section under §3 or §9 with the enforceable rules above.
3. Clarify "millions of agents" as logical/dormant vs active sessions; add the capacity model and say model/sandbox/human resources bind first.
4. Add cell architecture to §9: cell contents, global control plane, routing, migration, residency and blast-radius targets.
5. Keep Postgres ledger but change "Postgres sharded by org" to "by `(org_id, aggregate_hash)` with org placement in cells"; mention PgBouncer limits and external failover.
6. In §6.9, make the Postgres actor host's lease algorithm explicit: consistent hash for placement, lease rows for single writer, drain/rebalance protocol.
7. In §6.9, mark Temporal as Builders-only and cite history limits/fixed history shards as rationale.
8. In §8, mark Citus Excluded-license (AGPL-3.0); mark YugabyteDB Trial external due mixed Apache/Polyform; avoid CockroachDB core due use-restricted licensing.
9. In §6.1 identity, explicitly state: SPIFFE SVIDs are for workloads/pods, not logical agents; logical agents use kernel-minted attenuated capabilities bound to workload SVID.
10. In §10/§11, add scale proofs: 10M dormant sim, wake-storm, linear cells, chaos/cell failover, and cardinality-budget CI gates.

## What is still unsolved (late 2026)

- **Exact cell size.** It depends on workload mix, model choices, Postgres hardware and sandbox policy. Year96 needs load labs, not one claimed number.
- **Millions of concurrent sandboxes.** Google Agent Substrate confirms standard Kubernetes is not enough for ultra-dense, sub-second agent tool calls [23]. Core should support Agent Sandbox now and watch Agent Substrate, not bet the architecture on it.
- **LLM capacity economics.** 100k active sessions can imply millions of tokens/s. Without aggressive routing and caching, Year96 becomes a GPU procurement problem.
- **Hot orgs and hot threads.** Org-level sharding works until one org or thread dominates. Splitting aggregates without breaking the mental model needs explicit design.
- **Cross-cell transactions.** Avoid them. Federation/A2A plus sagas are slower but survivable.
- **Authorization cache invalidation.** Zanzibar proves scale is possible, but Year96's capabilities, revocation epochs and caveats need rigorous stale-cache proofs.
- **Deterministic simulation for LLM agents.** Model outputs must be recorded/mocked; otherwise sim cannot replay.
- **License volatility.** Citus/Cockroach-style relicensing risk means the license gate must run continually, not only during design.

## Sources

1. Temporal deployment/role model docs attempted; main usable Temporal sources are [7]-[9] and license [35].
2. https://grafana.com/docs/mimir/latest/references/architecture/deployment-modes/
3. https://grafana.com/docs/loki/latest/get-started/deployment-modes/
4. https://github.com/electric-sql/pglite/blob/main/LICENSE
5. https://github.com/temporalio/temporal/blob/main/LICENSE
6. https://docs.temporal.io/evaluate/cloud/limits
7. https://docs.temporal.io/workflow-execution/limits
8. https://github.com/temporalio/temporal/blob/main/docs/architecture/history-service.md
9. https://docs.temporal.io/evaluate/cloud/limits
10. https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html
11. https://slack.engineering/slacks-migration-to-a-cellular-architecture/
12. https://docs.nats.io/learn/clustering/
13. https://docs.nats.io/learn/topologies/leaf-nodes
14. https://developers.cloudflare.com/durable-objects/concepts/what-are-durable-objects/
15. https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/
16. https://openfga.dev/docs/getting-started/setup-openfga/docker
17. https://authzed.com/docs/spicedb/concepts/datastores
18. https://research.google/pubs/pub48190/
19. https://raw.githubusercontent.com/citusdata/citus/main/LICENSE
20. https://github.com/yugabyte/yugabyte-db/blob/master/LICENSE.md
21. https://www.pgbouncer.org/faq.html
22. https://kubernetes.io/docs/setup/best-practices/cluster-large/
23. https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate
24. https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox
25. https://spiffe.io/docs/latest/spiffe-about/overview/
26. https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/
27. https://opentelemetry.io/docs/concepts/sampling/
28. https://clickhouse.com/docs/en/observability
29. https://apple.github.io/foundationdb/testing.html
30. https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/internals/vopr.md
31. https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html
32. https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-production/
33. https://keda.sh/docs/latest/concepts/scaling-deployments/
34. https://docs.nats.io/running-a-nats-service/configuration/leafnodes
35. GitHub MCP opened temporalio/temporal LICENSE: MIT.
36. GitHub/web opened electric-sql/pglite LICENSE: Apache-2.0.
37. GitHub/web opened nats-io/nats-server LICENSE: Apache-2.0.
38. GitHub MCP opened openfga/openfga LICENSE: Apache-2.0.
39. GitHub MCP opened authzed/spicedb LICENSE: Apache-2.0.
40. GitHub/web opened apple/foundationdb LICENSE: Apache-2.0.
41. GitHub MCP opened dapr/dapr LICENSE: Apache-2.0.
42. GitHub/web opened kubernetes-sigs/agent-sandbox LICENSE: Apache-2.0.
43. GitHub/web opened agent-substrate/substrate LICENSE: Apache-2.0.
44. GitHub/web opened vllm-project/vllm LICENSE: Apache-2.0.
45. GitHub/web opened sgl-project/sglang LICENSE: Apache-2.0.
