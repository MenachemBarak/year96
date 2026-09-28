# 03 — Thread Memory

Scope: Year96 Threads are endless process representatives: every process has one thread, and every thread preserves chats, memory, meta-memory, milestones, clone branches, listeners/hangers, and dormancy/reactivation state. This report owns the thread data model, memory tiers, rehydration, clone fork/merge, fact invalidation, and the context assembly policy for any identity entering an old or active thread.

## TL;DR for the Year96 architect

- Treat a Thread as an append-only, event-sourced process envelope, not as a chat room. Raw chat spans live in #01 state fabric; thread memory stores bounded indexes, summaries, temporal facts, milestones, and context recipes.
- Adopt a hybrid memory architecture: Letta-style explicit memory blocks and tiered memory [5], Mem0-style extraction/consolidation/retrieval [6], and Graphiti/Zep-style temporal knowledge graph facts with validity windows [7]. Do not rely on one vector store.
- Meta-memory is first-class: `purpose`, `originatingDesire`, `successCriteria`, `constraints`, `owner`, `openQuestions`, and revision history. It is the “why” layer that prevents years-old reactivation from becoming arbitrary retrieval.
- Milestones are the unit of long-lived compaction. Detect them from explicit markers, event segmentation, decision/action boundaries, scope-effect changes from #02, artifacts, and “why changed” revisions; store raw span pointers, not only summaries.
- Growth must be bounded by design: context assembly uses recent tail caps, top-k evidence caps, per-tier budgets, recursive rollups, and cold storage. “Never closed” does not mean “ever-growing prompt.”
- Use “never delete” for authoritative raw events and audit records, but allow memory decay by demotion, lower retrieval priority, and supersession. MemoryBank’s Ebbinghaus-style forgetting is useful only as ranking, not deletion [18].
- Use bi-temporal semantic facts: `validTime` for the world claim and `transactionTime` for when Year96 learned/updated it. This is mandatory for conflict resolution and for answering “what did we believe then?” [7].
- Clone memory should be implemented as copy-on-write branches over thread+identity memory, then merged through a three-way memory diff: base, clone branch, current target. Merge summaries and facts separately; never silently merge private notes.
- Rehydration is a product feature, not a search query. It should produce a signed “rehydration brief”: why the thread exists, last stable milestone, decisions and rationale, unresolved questions, relevant changes since dormancy from #01/#02, current risks, and exact provenance.
- Context engineering research says long context rots; even million-token windows do not remove retrieval, compression, and just-in-time loading [13][14]. Year96 must optimize tokens like scarce working memory.
- Current vendor memory (ChatGPT, Claude, Vertex/Gemini, Bedrock) is useful as external integration but too user/project-scoped and opaque for core Year96 thread memory [15][16][17].
- Benchmarks to gate this subsystem: LongMemEval and LongMemEval-V2 for long-term interactive memory and environment experience [20], LoCoMo/LoCoMo-Plus for cognitive constraints [21], plus dormant-thread replay tests written from Year96 event logs.

## Landscape

### Agent memory systems and papers

**MemGPT / Letta (10/2023 paper; active platform in 2026).** MemGPT framed LLM agents as OS-like context managers with virtual memory tiers and interrupts, moving information between limited context and external recall/archival stores [5]. Letta is the maintained platform for stateful agents; GitHub reports Apache-2.0, ~24,953 stars, and a 2026-09 push [27]. Why it matters: Year96 Threads need explicit memory blocks and editable state, but should not inherit Letta’s whole agent runtime as a hard dependency.

**Mem0 (04/2025 paper; active OSS/product).** Mem0 proposes production long-term memory that dynamically extracts, consolidates, and retrieves salient conversational information, including a graph-memory variant; it reports better LoCoMo performance, 91% lower p95 latency, and >90% token-cost savings versus full-context replay [6]. GitHub reports Apache-2.0, ~66,221 stars, pushed 2026-09 [28]. Why it matters: good default `FactExtractor`/`MemoryProvider` trial, especially for user/thread facts, but its public design is not enough for Year96’s cross-thread/provenance/clone model.

**Zep / Graphiti (01/2025 paper; active OSS/product).** Zep’s paper describes Graphiti, a temporal KG for agent memory integrating conversational and business data while maintaining historical relationships; it reports DMR and LongMemEval gains and large latency reductions [7]. Graphiti GitHub reports Apache-2.0, ~31,264 stars, pushed 2026-09 [29]. Why it matters: this is the closest off-the-shelf pattern for Year96 semantic facts, fact invalidation, “what was true when,” and scope-effect over time.

**Cognee (active OSS).** Cognee positions itself as an open-source AI memory platform for agents, persistent long-term memory, graph databases, vector databases, and context engineering. GitHub reports Apache-2.0, ~31,125 stars, pushed 2026-09 [30]. Why it matters: promising trial for knowledge graph + vector memory pipelines, but Year96 should keep it behind providers.

**LangMem (01/2025 repo, active).** LangChain’s LangMem is a small library for agent memory on top of LangGraph/LangChain. GitHub reports MIT, ~1,687 stars, pushed 2026-09 [31]. Why it matters: useful for LangGraph-based executors and tests, but too framework-bound to be the canonical thread memory model.

**A-MEM (02/2025, NeurIPS 2025).** A-MEM applies Zettelkasten principles to agent memory: when adding memory, generate structured notes with contextual descriptions, keywords, tags, and dynamic links; new memories can update existing contextual representations [8]. Repo is MIT, ~974 stars, pushed 2026-03 [36]. Why it matters: excellent design inspiration for milestone-to-note linking and memory evolution; risk is LLM-driven link churn without auditable provenance.

**HippoRAG (2024/2025).** HippoRAG combines LLMs, knowledge graphs, and Personalized PageRank inspired by hippocampal indexing; the paper reports up to 20% better multi-hop QA, and 10-30x cheaper, 6-13x faster retrieval than iterative retrieval in some settings [9]. Repo is MIT, ~4,026 stars, pushed 2026-09 [35]. Why it matters: use graph-walk retrieval from milestones/facts to answer old-thread “why did we decide that?” questions.

**MIRIX (07/2025).** MIRIX proposes six memory types: Core, Episodic, Semantic, Procedural, Resource Memory, and Knowledge Vault, coordinated by multiple memory agents, with multimodal screen memory and LoCoMo gains [25]. Why it matters: its tier taxonomy maps well to Year96 (core/meta, episodic, semantic, procedural, resources), especially for multimodal state from #01. Treat as watch until code/license are verified.

**MemoryBank (05/2023).** MemoryBank uses long-term user memory, personality synthesis, memory updating, and an Ebbinghaus-inspired forgetting curve [18]. Why it matters: decay should affect retrieval priority and compaction cadence, not authoritative deletion.

**Generative Agents (04/2023).** Stanford’s generative agents used memory streams, reflection, planning, and dynamic retrieval over complete natural-language experience records [19]. Why it matters: reflections are early meta-memory/milestone rollups; Year96 needs stronger provenance, time validity, and permissions.

**RAPTOR (01/2024).** RAPTOR recursively embeds, clusters, and summarizes text into a tree, retrieving at multiple abstraction levels [23]. Why it matters: use recursive milestone rollups for very old threads: day/week/month/phase summaries with pointers to raw spans.

**LightRAG (10/2024, updated 2025).** LightRAG uses graph structures plus vector representations and incremental updates for simple/fast RAG [24]. GitHub reports MIT, ~39,902 stars, pushed 2026-09 [34]. Why it matters: strong candidate for lightweight graph+vector retrieval in laptop/small-cluster deployments.

**LongMemEval / LongMemEval-V2 (ICLR 2025; 2026 work-in-progress).** LongMemEval tests extraction, multi-session reasoning, temporal reasoning, knowledge updates, and abstention; V2 asks whether agents become “experienced colleagues” in specialized web environments using up to 500 trajectories and 115M tokens [20]. Why it matters: perfect regression gate for dormant-thread reactivation.

**LoCoMo-Plus (02/2026).** LoCoMo-Plus argues factual recall is insufficient and tests cognitive memory under cue-trigger semantic disconnect and latent user constraints [21]. Why it matters: Year96 meta-memory and ownership memory must preserve constraints/goals that are not later explicitly queried.

**Titans and Nested Learning (2025/2026).** Titans introduces neural long-term memory at test time [26]. Google-associated Nested Learning frames learning as nested context flows and memory systems [26]. Why it matters: watch for model-native memory; do not build core 2027 architecture assuming it replaces external auditable memory.

### Context engineering

**Anthropic context engineering (2025).** Anthropic defines context engineering as curating/maintaining the optimal token set during inference across system prompts, tools, external data, and message history; it explicitly cites context rot and recommends tight, high-signal context plus just-in-time retrieval [13]. Why it matters: Year96 `ContextAssembler` must be a policy engine, not “dump relevant history.”

**Chroma context rot (2025).** Chroma shows performance degrades as input length grows even on controlled tasks; simple needle benchmarks overstate long-context robustness [14]. GitHub repo is MIT, ~312 stars [37]. Why it matters: hard evidence that Year96 should use bounded context and external memory even with large-context models.

**Agentic Context Engineering / ACE (2025 paper, ICLR 2026).** ACE treats context as evolving playbooks updated through generation, reflection, and curation; it targets brevity bias and context collapse and reports gains on agent and finance benchmarks [22]. Why it matters: procedural memory/playbooks should evolve like audited documents, not only summaries.

### Vendor memory

**Claude memory (2025).** Anthropic’s Claude memory is optional, project-scoped, user-visible/editable, with incognito chats and enterprise controls [15]. Why it matters: project-scoped memory validates Year96’s per-thread scoping, but vendor memory lacks raw provenance and clone/fork/merge semantics.

**OpenAI ChatGPT memory (2026 help page).** OpenAI documents improved memory, project-only memory, temporary chats that do not update memory, and regulated-workspace limitations [16]. Why it matters: confirms industry convergence on scoped/controllable memory; still external integration only.

**AWS Bedrock AgentCore Memory (2026 docs).** AgentCore Memory provides fully managed short-term and long-term memory, extracting key insights across sessions for personalization and workflow agents [17]. Why it matters: good enterprise external provider; too opaque for Year96’s canonical thread memory.

**Google Gemini Enterprise Agent Platform Memory Bank (2026 docs).** Google’s platform includes Sessions and Memory Bank for persistent long-term memories, plus evaluation/tracing services [12]. Why it matters: validates memory as part of runtime+eval platform; Year96 should expose a compatible provider but keep core portable.

### Thread structures

**Zulip topics.** Zulip makes each conversation a first-class topic rather than a hidden side thread, letting conversations last hours/days and remain findable; threads are labeled by topic rather than first message [10]. Why it matters: Year96 should label threads by purpose/meta-memory, not by a starting message.

**Slack threads.** Slack API identifies threaded messages and replies via conversation history and thread retrieval [11]. Why it matters: common integration pattern, but insufficient for endless Year96 process threads.

**Matrix threads and email/JWZ.** Matrix’s event model and rooms support persistent distributed event history, while JWZ email threading uses Message-ID/In-Reply-To/References to reconstruct reply trees. My JWZ fetch failed; Matrix spec fetch hit the general spec page, so detailed claims remain partially verified. Lesson: use explicit graph links, not inferred subject lines.

**Steve Yegge’s Beads.** The lead’s “Beads” pointer could not be verified from `beads.land`, which is unrelated craft content [39]. Mark as unverified until #11/#12 find the correct source.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| Letta | OSS/product | Stateful agents, explicit memory blocks, virtual-context pattern | Apache-2.0 verified via GitHub API [27] | ~24,953 stars; pushed 2026-09; Letta AI | Trial |
| Mem0 | OSS/product/paper | Production extraction, consolidation, retrieval; graph option | Apache-2.0 verified [28] | ~66,221 stars; pushed 2026-09; Mem0 | Trial |
| Graphiti | OSS/paper | Temporal KG, fact history, relationship invalidation | Apache-2.0 verified [29] | ~31,264 stars; pushed 2026-09; Zep | Adopt |
| Zep | OSS/product | Hosted/OSS memory service around temporal KG | Apache-2.0 verified [32] | ~4,937 stars; pushed 2026-09; Zep | Trial |
| Cognee | OSS/product | Agent memory platform, KG + vector memory | Apache-2.0 verified [30] | ~31,125 stars; pushed 2026-09 | Trial |
| LangMem | OSS | LangGraph-compatible memory library | MIT verified [31] | ~1,687 stars; pushed 2026-09; LangChain | Watch/Trial |
| A-MEM | Paper/OSS | Zettelkasten-style dynamic notes/links | MIT verified [36] | ~974 stars; pushed 2026-03; NeurIPS 2025 | Trial |
| HippoRAG | Paper/OSS | Graph + PPR retrieval for multi-hop memory | MIT verified [35] | ~4,026 stars; pushed 2026-09 | Trial |
| LightRAG | Paper/OSS | Fast graph+vector RAG with incremental updates | MIT verified [34] | ~39,902 stars; pushed 2026-09; EMNLP 2025 | Trial |
| LlamaIndex | OSS/product | Connectors, indexes, retrievers, memory integrations | MIT verified [33] | ~52,337 stars; pushed 2026-09 | Trial |
| txtai | OSS | Semantic search/vector workflows | Apache-2.0 verified [33] | ~12,985 stars; pushed 2026-09 | Watch |
| Chroma context-rot repo | OSS/research | Reproducible context-length degradation tests | MIT verified [37] | ~312 stars; pushed 2025-09 | Adopt as regression idea |
| LongMemEval | Benchmark | Multi-session, temporal, update, abstention memory tests | Code license not verified | ICLR 2025; public repo noted in paper [20] | Adopt |
| LongMemEval-V2 | Benchmark | Environment-experience memory over trajectories | Code license not verified | 2026 WIP [20] | Adopt |
| LoCoMo-Plus | Benchmark | Latent constraint/cognitive memory evaluation | Code license not verified | 2026 paper [21] | Trial |
| Claude memory | Product | Project-scoped editable work memory | Proprietary | Anthropic, 2025 [15] | External integration |
| OpenAI memory | Product | Project-only/temporary memory controls | Proprietary | OpenAI, 2026 help [16] | External integration |
| AWS AgentCore Memory | Product | Managed short/long-term memory | Proprietary service | AWS, 2026 docs [17] | External integration |
| Google Memory Bank | Product | Enterprise sessions + memory bank | Proprietary service | Google, 2026 docs [12] | External integration |
| Beads | Unverified | Possible agent issue/memory graph lead | Unverified | Lead may have wrong URL [39] | Watch/unverified |

## How I would build this part of Year96

### Design headline

Build Thread Memory as **bounded projections over immutable state**, not as a mutable chat transcript. The canonical facts are events in #01 state fabric. Thread Memory owns typed projections: meta-memory, milestone index, retrieval indexes, identity notes, procedural runbooks, and context assembly policies. Every projection is reproducible from source spans or explicitly marked as human/agent-authored interpretation.

### TypeScript data model

```ts
type ID<T extends string> = string & { readonly __brand: T };
type ThreadId = ID<'thread'>;
type ChatId = ID<'chat'>;
type IdentityId = ID<'identity'>;
type CloneId = ID<'clone'>;
type SpanId = ID<'state-span'>;
type ArtifactId = ID<'artifact'>;
type MilestoneId = ID<'milestone'>;
type FactId = ID<'fact'>;
type MemoryId = ID<'memory'>;

type ThreadStatus = 'active' | 'dormant' | 'hanging-watched';
type ParticipantRole = 'participant' | 'listener' | 'hanger' | 'communicator' | 'owner';

type TemporalRange = {
  from: string;
  to?: string;
  precision?: 'instant' | 'day' | 'month' | 'year' | 'unknown';
};

type ProvenancePointer = {
  spanId: SpanId;
  eventIds?: string[];
  artifactIds?: ArtifactId[];
  quoteHash?: string;
  createdBy: IdentityId | 'system';
  createdAt: string;
};

type MetaMemory = {
  purpose: string;
  why: string;
  originatingDesire?: string;
  successCriteria: string[];
  owner: IdentityId;
  constraints: string[];
  openQuestions: string[];
  nonGoals: string[];
  scopeAssumptions: string[];
  revisionHistory: Array<{
    at: string;
    by: IdentityId | 'system';
    patch: string;
    rationale: string;
    provenance: ProvenancePointer[];
  }>;
};

type Thread = {
  id: ThreadId;
  orgId: string;
  title: string;
  createdAt: string;
  lastActiveAt: string;
  status: ThreadStatus;
  statusReason?: string;
  participants: Array<{
    identityId: IdentityId;
    role: ParticipantRole;
    joinedAt: string;
    leftAt?: string;
    permissionsRef: string;
  }>;
  parentThreadId?: ThreadId;
  childThreadIds: ThreadId[];
  linkedThreadIds: Array<{ threadId: ThreadId; relation: 'blocks'|'duplicates'|'derived-from'|'related'|'supersedes'|'forked-from' }>;
  chatIds: ChatId[];
  metaMemory: MetaMemory;
  memoryPolicy: ThreadMemoryPolicy;
};

type Chat = {
  id: ChatId;
  threadId: ThreadId;
  kind: 'self-talk' | 'two-party' | 'multi-identity' | 'clone-room' | 'external-import';
  participantIds: IdentityId[];
  rawSpanIds: SpanId[];
  startedAt: string;
  lastMessageAt: string;
};

type Milestone = {
  id: MilestoneId;
  threadId: ThreadId;
  boundary: { startSpanId: SpanId; endSpanId: SpanId; eventTime: TemporalRange };
  type: 'decision' | 'why-change' | 'artifact' | 'handoff' | 'risk' | 'scope-change' | 'periodic-rollup' | 'reactivation' | 'clone-merge';
  title: string;
  summary: string;
  decisions: Array<{ decision: string; why: string; alternativesRejected: string[]; provenance: ProvenancePointer[] }>;
  artifacts: ArtifactId[];
  openQuestions: string[];
  factPointers: FactId[];
  rawPointers: ProvenancePointer[];
  parentMilestoneIds: MilestoneId[];
  importance: number;
};

type TemporalFact = {
  id: FactId;
  subject: string;
  predicate: string;
  object: string | number | boolean | object;
  scope: { orgId: string; threadId?: ThreadId; identityId?: IdentityId; visibility: 'private'|'thread'|'org'|'external' };
  validTime: TemporalRange;
  transactionTime: TemporalRange;
  confidence: number;
  status: 'active' | 'superseded' | 'disputed' | 'retracted';
  supersedes?: FactId[];
  conflictsWith?: FactId[];
  provenance: ProvenancePointer[];
};

type MemoryRecord = {
  id: MemoryId;
  threadId?: ThreadId;
  identityId?: IdentityId;
  tier: 'working' | 'episodic' | 'semantic' | 'procedural' | 'meta' | 'private-note' | 'resource';
  content: string;
  embeddingRef?: string;
  factIds?: FactId[];
  milestoneIds?: MilestoneId[];
  provenance: ProvenancePointer[];
  createdAt: string;
  lastAccessedAt?: string;
  importance: number;
  decay: { halfLifeDays?: number; pinned?: boolean; retrievalBoost: number };
};

type ThreadMemoryPolicy = {
  recentTailMessages: number;
  recentTailTokenBudget: number;
  milestoneBudget: number;
  semanticFactBudget: number;
  privateNoteBudget: number;
  proceduralBudget: number;
  requireProvenanceForFacts: boolean;
  coldAfterDaysInactive: number;
};
```

### Provider interfaces

```ts
interface ThreadStore {
  getThread(id: ThreadId): Promise<Thread>;
  appendThreadEvent(threadId: ThreadId, event: ThreadEvent): Promise<void>;
  listDormantDueForWatch(now: Date): Promise<ThreadId[]>;
  linkThreads(a: ThreadId, b: ThreadId, relation: Thread['linkedThreadIds'][number]['relation']): Promise<void>;
}

interface MemoryProvider {
  upsert(record: MemoryRecord): Promise<void>;
  retrieve(q: MemoryQuery, budget: RetrievalBudget): Promise<MemoryRecord[]>;
  markAccessed(ids: MemoryId[], at: Date): Promise<void>;
  compact(scope: MemoryScope, policy: ThreadMemoryPolicy): Promise<CompactionReport>;
}

interface TemporalFactProvider {
  extract(input: FactExtractionInput): Promise<TemporalFact[]>;
  upsertFact(fact: TemporalFact): Promise<FactId>;
  query(query: FactQuery, asOf: { validAt?: Date; transactionAt?: Date }, budget: RetrievalBudget): Promise<TemporalFact[]>;
  resolveConflicts(facts: FactId[], policy: ConflictPolicy): Promise<ConflictResolution>;
}

interface MilestoneDetector {
  detect(events: ThreadEvent[], hints: DetectionHints): Promise<MilestoneCandidate[]>;
  materialize(candidate: MilestoneCandidate): Promise<Milestone>;
}

interface Compactor {
  summarizeSpan(span: SpanId[], target: 'chat'|'milestone'|'periodic'|'thread'): Promise<MemoryRecord>;
  rollup(milestones: Milestone[], level: 'day'|'week'|'month'|'phase'): Promise<Milestone>;
}

interface Rehydrator {
  buildBrief(req: { threadId: ThreadId; identityId: IdentityId; now: Date; reason: string }): Promise<RehydrationBrief>;
}

interface ContextAssembler {
  assemble(req: ContextRequest): Promise<AssembledContext>;
}

interface FactExtractor {
  extractFromSpan(span: SpanId, scope: MemoryScope): Promise<TemporalFact[]>;
}
```

### Algorithms

**Milestone detection.** Run a cheap streaming detector on every appended chat/event and a stronger batch detector at idle points. Signals: explicit markers (“decision,” “ship,” “blocked”), tool/artifact creation, participant handoff, meta-memory revision, #02 scope-effect delta above threshold, long silence followed by reactivation, clone fork/merge, and semantic event segmentation. The detector emits candidates; materialization requires a bounded evidence packet: relevant tail, candidate raw spans, extracted decisions/why, and artifact pointers. Milestones always include raw pointers so compaction is reversible.

**Progressive compaction.** Use a four-level pyramid: chat-span summaries -> milestones -> phase rollups -> thread rehydration briefs. Each summary stores (a) what happened, (b) why it happened, (c) decisions and rejected alternatives, (d) facts extracted/superseded, (e) raw span pointers, and (f) uncertainty. Never overwrite previous summaries; write a new rollup with parent pointers. This avoids context collapse and allows comparison across revisions.

**Forgetting/decay.** Raw events and audit records are never deleted by Thread Memory. “Forgetting” means lower default retrieval priority, move embeddings/objects to cold tier, and require stronger query similarity or explicit user/agent request. Pinned memory, legal/audit facts, success criteria, and safety constraints do not decay. Decay can be computed from importance, last access, contradiction status, and scope-effect frequency.

**Dormancy -> rehydration.** A dormant thread has watch predicates registered with #02 and #01: relevant state changes, new linked artifacts, owner desire updates, deadlines, or similar active threads. On reactivation, `Rehydrator` builds: identity-specific access check (#06), meta-memory, last milestone, active facts as-of-now, facts changed since lastActive, unresolved questions, #02 scope-effect report, recent linked-thread deltas, risk flags, and suggested next actions. The brief is itself a milestone of type `reactivation`.

**Clone fork/merge.** Fork creates a `CloneMemoryBranch` referencing base thread, base milestone/fact frontier, clone identity, attenuation policy from #06, and private memory namespace. It is copy-on-write: new notes/facts/milestones reference the branch. Merge uses three-way diff: reconstruct base memory frontier; compare clone branch and target current memory by tier; auto-merge non-conflicting public facts with provenance; mark conflicts as `disputed`; summarize clone chat into a milestone; keep private notes private unless clone identity explicitly publishes them; add a `clone-merge` milestone with decisions and conflict list.

**Fact invalidation/conflict resolution.** Facts are never updated in place. A correction writes a new fact with `supersedes` and closes old transaction time. Conflict policy ranks by authority (#06), recency, provenance quality, direct observation vs inference, confidence, and thread scope. Queries can ask `asOf.validAt` and `asOf.transactionAt` to reproduce old beliefs.

**ContextAssembler policy.** For any identity joining/resuming: check permissions and role (#04/#06); always include a concise thread header with purpose/why/owner/success criteria/constraints/status; include a rehydration brief if dormant or if identity is new; include recent tail capped by messages/tokens; include top milestones by relevance/importance/time diversity; include semantic facts with time labels and conflict flags; include procedural playbooks relevant to requested action; include identity-private notes only for that identity/clone lineage; include pointers, not bulk raw logs; emit a context manifest with every included item, reason, token cost, provenance, and excluded-over-budget counts.

### Typical flow

A human opens a thread for a desire. #04 Communicator creates `Thread` with participants and meta-memory. Chats append raw spans to #01. `FactExtractor` extracts candidate facts; `MilestoneDetector` identifies decision/artifact/why-change boundaries. The thread goes quiet; `status=dormant`, watch predicates remain. Two years later #02 notices a state change that may affect scope, or a human asks about the process. `Rehydrator` queries #01/#02/#06/#07, writes a reactivation milestone, and `ContextAssembler` gives the entering identity a compact but precise context. If clones brainstorm, their branch merges through a provenance-preserving diff.

### Plugging into other layers

- #01 state-fabric owns immutable raw spans, object storage, snapshots, and universal search.
- #02 scope-effect supplies watch predicates, relevance scores, and “what changed since last active.”
- #04 comm-hub owns communicator roles, listeners/hangers UX, and chat transport.
- #05 runtime schedules compaction, dormancy watches, rehydration jobs, and cold-tier migrations.
- #06 identity-gates enforces private notes, clone attenuation, and merge publish rights.
- #07 ownership-duty consumes meta-memory as mental-model material and writes back “why” revisions.
- #09 verification defines replay, trace, observability, and benchmark gates.
- #11 harness-stack supplies AGENTS/skills/context files as procedural memory surfaces.

### Scaling path

Laptop: SQLite/DuckDB for metadata, LanceDB/pgvector for embeddings, Kuzu/SQLite graph or NetworkX for temporal facts, local files for raw spans. Team server: Postgres + pgvector, object storage, Neo4j/FalkorDB/Kuzu, queue workers for extraction/compaction. Cluster: event bus from #01, sharded thread projections by org/thread, vector/graph stores with tenant isolation, cold object tiers, and batch compaction jobs. The interface stays stable; only providers change.

### Testing and proof

Unit tests cover schema validation, caps, three-way merge, bi-temporal queries, decay ranking, and context manifests. Integration tests replay known chats into memory providers and assert deterministic milestones/facts with mocked LLM extractors. Mocked E2E tests simulate a thread dormant for years, external state changes, and reactivation. Real E2E tests run against selected providers (Graphiti/Mem0/Cognee/LightRAG). Agentic verifier tests ask adversarial questions: stale facts, private clone notes, conflicting decisions, “what did we believe in 2025,” and latent constraints. Benchmarks: LongMemEval/LongMemEval-V2, LoCoMo-Plus, Chroma context-rot-style long-context degradation, plus Year96-specific dormant replay corpora. Promotion gate: no regression in recall, temporal accuracy, abstention, provenance citation, token budget, p95 rehydration latency, and privacy leakage.

## What is still unsolved (late 2026)

- **Reliable extraction of “why.”** Facts are easy compared to motives, rejected alternatives, and evolving success criteria. Year96 needs human/agent review loops for high-impact meta-memory revisions.
- **Conflict resolution across identities.** A system can rank evidence, but political/organizational truth may require explicit owner decisions. The memory layer must surface conflicts, not pretend to solve them.
- **Clone psychology and privacy.** Fork/merge mechanics are clear; social semantics are not. When is a clone’s private note publishable? #06/#07 must define identity continuity and consent.
- **Milestone detection quality.** Event segmentation is promising, but false milestones create clutter and missed milestones damage rehydration. Need long-running evals with human labels.
- **Context rot under tool use.** We can cap context, but agents may still retrieve too much or over-trust summaries. Need verifier-driven context manifests and retrieval audits.
- **Cross-thread memory without contamination.** Organization-wide learning is powerful but can leak irrelevant assumptions between threads. Use explicit scope, confidence, and provenance on every memory item.
- **Vendor lock-in.** The best managed memory services are proprietary and opaque. Year96 core must remain provider-based and auditable.
- **Benchmarks still underfit Year96.** Public benchmarks test chat assistants and web agents, not never-closed process threads with clones, hangers, and scope-effect watches. Build a Year96 benchmark suite from day one.

## Sources

1. Year96 team brief, `C:\projects\year96_!\year96\docs\research\00-TEAM_BRIEF.md`.
2. Year96 intro, `C:\projects\year96_!\year96\docs\YEAR96_INTRO.md`.
3. Year96 spec, `C:\projects\year96_!\year96\docs\YEAR96_SPEC.md`.
4. Year96 vision, `C:\projects\year96_!\year96\docs\YEAR96_Vision.md`.
5. https://export.arxiv.org/api/query?id_list=2310.08560
6. https://export.arxiv.org/api/query?id_list=2504.19413
7. https://export.arxiv.org/api/query?id_list=2501.13956
8. https://export.arxiv.org/api/query?id_list=2502.12110
9. https://export.arxiv.org/api/query?id_list=2405.14831
10. https://zulip.com/help/introduction-to-topics
11. https://docs.slack.dev/messaging/retrieving-messages
12. https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale
13. https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
14. https://www.trychroma.com/research/context-rot
15. https://claude.com/blog/memory
16. https://help.openai.com/en/articles/8590148-memory-in-chatgpt
17. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html
18. https://export.arxiv.org/api/query?search_query=ti:%22MemoryBank%22+AND+all:Ebbinghaus&max_results=5
19. https://export.arxiv.org/api/query?id_list=2304.03442
20. https://export.arxiv.org/api/query?search_query=ti:%22LongMemEval%22&max_results=3
21. https://export.arxiv.org/api/query?search_query=ti:%22LoCoMo%22&max_results=3
22. https://export.arxiv.org/api/query?id_list=2510.04618
23. https://export.arxiv.org/api/query?id_list=2401.18059
24. https://export.arxiv.org/api/query?search_query=ti:%22LightRAG%22&max_results=5
25. https://export.arxiv.org/api/query?search_query=ti:%22MIRIX%22&max_results=5
26. https://export.arxiv.org/api/query?search_query=ti:%22Titans%22+AND+all:memory&max_results=5 and https://export.arxiv.org/api/query?search_query=ti:%22Nested%20Learning%22&max_results=5
27. https://api.github.com/repos/letta-ai/letta
28. https://api.github.com/repos/mem0ai/mem0
29. https://api.github.com/repos/getzep/graphiti
30. https://api.github.com/repos/topoteretes/cognee
31. https://api.github.com/repos/langchain-ai/langmem
32. https://api.github.com/repos/getzep/zep
33. https://api.github.com/repos/run-llama/llama_index and https://api.github.com/repos/neuml/txtai
34. https://api.github.com/repos/HKUDS/LightRAG
35. https://api.github.com/repos/OSU-NLP-Group/HippoRAG
36. https://api.github.com/repos/WujiangXu/A-mem
37. https://api.github.com/repos/chroma-core/context-rot
38. https://spec.matrix.org/v1.15/client-server-api/#threading
39. https://beads.land/
