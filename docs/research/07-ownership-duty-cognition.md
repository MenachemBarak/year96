# 07 — Ownership & Duty cognition

Scope: This report designs the Year96 Ownership and Duty layers: how a 2027 agentic OS represents a human or organization's mental model, records the WHY behind decisions/preferences/bugs/processes, maintains strategy and sensors, converts cognition into Duties and Builder requests, and improves itself without the Owner ever executing. I treat Ownership as the organization's executive cognition and Duty as delegated sub-cognition that converts a mental model into actionable insight and execution contracts.

## TL;DR for the Year96 architect

- Model Ownership as **BDI + CoALA + org-role governance**, not as a task queue: Beliefs are state/why/memory; Desires are strategies and optimal vectors; Intentions are Duties and BuilderRequest contracts. CoALA's memory/action/decision split maps cleanly to Year96's state, internal thoughts, and external Builders [1].
- The Owner must be an **explicit non-executor**: it can create/update/delete Duties, set charters, request verification, and decide strategy, but cannot call execution tools. Enforcement belongs in #06 identity gates and #05 runtime policies.
- Use a **why graph** as first-class state: every goal, preference, decision, process, incident, and Builder result is a typed node with causal/rationale edges. ADR/MADR/IBIS/QOC-style rationale becomes queryable graph data, not Markdown folklore [19].
- Ambient ownership is now plausible: LangChain defines ambient agents as event-stream listeners that act on multiple events and use notify/question/review human-in-the-loop patterns [4]; OpenAI Pulse productizes proactive overnight briefs [6]; Letta sleep-time compute explicitly turns raw context into learned context during idle time [5].
- Long-horizon autonomy is improving but not solved. METR's current TH1.1 reframes horizon as task difficulty measured by human completion time; original 2025 trend was about 7-month doubling, but 2026 TH1.1 says trend estimates are sensitive to task composition and long-task coverage [2][3]. Year96 should plan for reliable **minutes-hours autonomous ownership**, with days-long loops only under verification and checkpointing.
- Multi-agent Ownership should be **orchestrator-worker** only for breadth-first cognition. Anthropic reports a 90.2% internal eval improvement for multi-agent research, mostly by using parallel context windows/tokens, but at high coordination and token cost [7].
- Mental-model capture should combine **active elicitation** (GATE asks informative questions/edge cases) [8], **passive edit learning** (PRELUDE learns latent preferences from edits) [14], interview agents, and behavioral telemetry. The model must store confidence/provenance and ask only when expected value is high.
- "Optimal vector" should be a concrete multi-objective target: weighted utility dimensions plus constraints, Pareto front candidates, evidence, and review cadence. It is not a vibe; it is the scoring function Duties use to choose what to propose.
- Duty is not a worker. Duty owns a bounded sub-domain: knowledge base, settings, ongoing processes, insight backlog, SLAs, and BuilderRequest queue. Duties discuss sibling CRUD but request Builders rather than execute.
- Use provider interfaces everywhere: OwnershipStore, MentalModelProvider, WhyGraphProvider, StrategyPlanner, SensorProvider, ObjectiveProvider, DutyManager, BuilderRequestor, VerificationProvider, LearningProvider. Logic is stateless; state lives behind stores.
- Adopt permissive core components where useful: LangGraph (MIT verified) for stateful/ambient orchestration, Letta Code (Apache-2.0 verified) for memory/sleep-time patterns, Temporal TypeScript SDK (MIT visible in README) for durable cadences. Avoid AI-Scientist-v2 as core because its custom use-restricted license is excluded for core.
- The hard product problem is **authority calibration**: when should Ownership silently learn, notify, ask, review, create a Duty, or request execution? This must be learned per human/thread and constrained by #02 scope effect, #06 permissions, and #09 proofs.

## Landscape

### Cognitive architectures and planning

**CoALA — Cognitive Architectures for Language Agents (09/2023, still central in 2026).** CoALA defines a language agent by memory modules, action spaces, and decision processes, and explicitly discusses internal vs external actions and continuous autonomous learning [1]. For Year96, CoALA is the right vocabulary for separating Ownership cognition from Duty/Builder actions: internal thought updates mental model/why graph; external grounding/request actions go through gates.

**SOAR/ACT-R lineage and BDI.** Classical cognitive architectures give Year96 a warning: long-lived cognition needs symbolic structure, goals, procedures, and traceability, not just prompt history. BDI is the best direct mapping: Beliefs = state plus why graph, Desires = strategies/objectives, Intentions = active Duties and BuilderRequest plans. Jason/JaCaMo remains a useful BDI reference system, but its Java/academic ecosystem should be a design reference rather than Year96 core unless #11 finds a modern maintained provider.

**ReAct (10/2022), Reflexion (03/2023), Tree of Thoughts (05/2023), LATS (10/2023).** These are still baseline agent-planning moves: interleave reasoning/actions [15], store verbal lessons after failures [16], deliberate over trees [17], and unify reasoning/action/planning/search [18]. For Year96 they belong as StrategyPlanner algorithms and Builder-side tactics, not as top-level architecture. Ownership uses them to evaluate options and create Duties; Builders use them to execute.

**HTN/PDDL/LLM planning and hierarchical multi-agent planning (2024-2026).** The trend is hybrid: LLMs propose/decompose, symbolic planners validate constraints, and task networks provide stable hierarchy. Year96 should use HTN-like decomposition for Ownership → Duty → Builder boundaries, with PDDL-style constraints for permissions, budget, deadlines, and proof criteria. Pure LLM planners are too brittle for authority decisions.

### Long-horizon evidence

**METR time horizons (03/2025, TH1.1 update 01/2026).** METR defines task-completion time horizon as the human-expert task duration at which an AI agent reaches a chosen success probability [3]. The original 2025 post reported roughly 7-month doubling of the 50%-success horizon [2], but it now warns that parts are out of date and points to TH1.1. TH1.1 adds tasks, moves infrastructure from Vivaria to Inspect, doubles long tasks from 14 to 31, and notes trend sensitivity [3]. For Year96, this means: do not assume stable exponential extrapolation; instrument our own horizon benchmark per Duty class.

**Anthropic multi-agent research system (06/2025).** Anthropic's Research feature uses a lead agent to spawn parallel search agents; it works because subagents get separate context windows, separation of concerns, and enough token budget. They report 90.2% improvement over a single-agent baseline on internal research evals and state that token use explained most variance on BrowseComp-style performance [7]. Year96 should apply this selectively: Ownership can spawn Duties or research subagents for breadth, but avoid multi-agent theater for linear tasks.

**AI Scientist v2 and Agent Laboratory (01-04/2025).** Agent Laboratory automates literature review, experimentation, and report writing for research assistance [11]. AI Scientist-v2 uses agentic tree search to generate hypotheses, run experiments, analyze data, and write papers; its repo warns that LLM-written code must run in a sandbox [10][13]. These systems prove multi-stage scientific ownership is possible but fragile. AI-Scientist-v2's license is custom use-restricted, therefore excluded from Year96 core.

### Mental-model and preference elicitation

**GATE / generative elicitation (paper 2023, ICLR 2025 proceedings).** The GitHub README states that Generative Active Task Elicitation asks open-ended questions or generates edge cases to infer intended behavior, and that users report less effort than prompting or labeling [8]. This should be the default MentalModelProvider for cold-start: ask a few high-information questions, synthesize edge cases, and store answers as preference/rationale nodes.

**PRELUDE (NeurIPS 2024, still relevant).** PRELUDE learns latent user preference from user edits in tasks such as summarization and email writing [14]. For Year96, edits are not corrections to a single artifact; they are evidence about a user's values, style, risk tolerance, and constraints. Every edit should update the why graph with provenance and confidence.

**Personalized LLM agents / digital twins (2026).** The 2026 trend is persistent per-user models with memory, profile, tool history, and preference learning. Year96 should avoid pretending to simulate the person; instead it should maintain an auditable *operational mental model* with explicit confidence, last-reviewed time, and allowed use.

### WHY and rationale capture

**IBIS/QOC/design rationale/ADRs/MADR.** MADR describes an architectural decision as a justified choice and emphasizes recording context, options, and outcome in lightweight Markdown [19]. Year96 should generalize this to all domains: every decision has context, options, chosen outcome, rejected alternatives, evidence, owner, expiry/review trigger, and consequences. The storage should be graph-first, with ADR/MADR generated as a view.

**5 Whys and causal why graphs.** 5 Whys is too linear alone, but useful as a UI move for incident Duties. Year96's why graph should support multiple causes, conflicting rationales, temporal validity, and counterfactuals: "we do X because Y, unless Z changes." This is essential for dormant threads that reactivate years later (#03).

### Strategy, sensors, and optimal vectors

**OKRs, GQM, Opportunity Solution Trees, Impact Mapping, Wardley Maps, North-Star metrics, OODA.** These are not interchangeable. OKRs express commitments; GQM converts goals to questions and metrics; Opportunity Solution Trees map customer opportunities; Impact Mapping links actors/impacts/deliverables; Wardley Maps locate evolution/commoditization; OODA is the control loop. Year96 should store all as strategy artifacts under a common `StrategyFrame` type and let Duties choose the useful lens.

**Optimal vectors.** The concrete formalization: an `OptimalVector` is a multi-objective utility vector over dimensions such as revenue, learning, risk, time, cost, human energy, reversibility, trust, quality, and optional values constraints. Candidate strategies are scored into vector space; dominance/Pareto filters remove bad options; a human-reviewed weight profile chooses tradeoffs. Embedding-space goal vectors can help retrieval, but final authority should be numeric/symbolic and explainable.

### Org-design analogs

**Amazon single-threaded owner and Apple DRI.** These analogs matter because they distinguish ownership from doing: one accountable identity keeps context, tradeoffs, and outcomes coherent. Year96 Ownerships need a named accountable human/agent charter even when thousands of Builders run.

**Holacracy and Sociocracy.** Holacracy encodes roles with purpose/accountabilities/domains and governance processes; its constitution has Circle roles, Circle Reps, Facilitator, Secretary, and governance records [20]. The useful pieces for Year96: charters, domains, accountabilities, governance-vs-tactical split, and explicit role changes. The Communicator (#04) is closest to facilitator/secretary: process guardian, not decision-maker.

**Microsoft Work Trend Index / Frontier Firm (2025).** Microsoft's Frontier Firm messaging frames organizations around humans directing agents. Treat it as product/market evidence that "agent boss" and AI-managed workflows are becoming mainstream, but do not use it as technical proof; the fetched page exposed only high-level guidance [21].

### Proactive ambient ownership

**LangChain ambient agents (2025).** Ambient agents listen to event streams, can handle multiple events at once, and use notify/question/review patterns [4]. This is directly Year96's sensor-to-action UX.

**Letta sleep-time compute (2025).** Letta argues stateful agents should reason while idle, turning raw context into learned context [5]. This is exactly how Ownership improves the why over time without waiting for a user prompt.

**ChatGPT Pulse (09/2025).** OpenAI Pulse runs background research overnight, uses chat history and opt-in app context, and produces proactive daily cards [6]. This validates proactive ownership UX: small, reviewable, opt-in insights rather than surprise execution.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| CoALA | Paper/framework | Memory/action/decision vocabulary for Ownership cognition | Paper, no code license | 2023 paper; still cited; opened arXiv | Adopt |
| LangGraph | OSS | Durable stateful/ambient agent graph; human-in-loop; memory; deployment | MIT verified from LICENSE via GitHub MCP | GitHub page opened; stars/commit unverified due API rate limit; backed by LangChain | Adopt |
| Letta Code / MemGPT lineage | OSS/product | Stateful memory, sleep-time compute, learned context | Apache-2.0 verified via `letta-ai/letta-code` LICENSE | Active source moved from `letta-ai/letta` to `letta-ai/letta-code`; stars/commit unverified | Trial |
| Temporal TypeScript SDK | OSS | Durable cadences, workflows, retries for Ownership/Duty loops | MIT visible in GitHub README [22] | Mature Temporal-backed SDK; stars/commit unverified | Adopt with #05 |
| GATE / generative-elicitation | OSS + paper | Active mental-model/preference elicitation via questions/edge cases | MIT verified via LICENSE | Research repo; stars/commit unverified | Trial |
| PRELUDE | OSS + paper | Learns latent preferences from user edits | MIT verified via LICENSE | NeurIPS 2024 research repo; stars/commit unverified | Trial |
| Anthropic multi-agent research pattern | Product/architecture article | Lead/subagent breadth-first research and token-scaling lessons | Proprietary product; article only | Production feature; Anthropic-backed | Adopt pattern, not component |
| METR time horizon methodology | Evaluation methodology | Measures realistic autonomous task horizon per Duty class | Paper/site; eval repo not license-verified here | Current TH1.1 page; METR-backed | Adopt metric |
| AI Scientist-v2 | OSS + paper | Agentic tree-search ownership over research pipeline | Custom use-restricted license verified; not SPDX | 2025 repo/paper; sandbox warning; Sakana-backed | Excluded-license (core) |
| Agent Laboratory | OSS + paper | Research assistant pipeline example | MIT verified via LICENSE | 2025 repo/paper; stars/commit unverified | Watch/Trial external |
| MADR/ADR | Standard/template | Lightweight decision/rationale records as views over why graph | Site/template; license not verified | Established architecture practice | Adopt concept |
| Holacracy Constitution | Standard/org design | Roles, domains, accountabilities, governance process | Legal text; not software | Version 5.0 site opened | Trial concepts |
| LangChain Open Agent Platform | OSS/product | No-code agent builder lineage | License not verified in this pass | Fetched README says repository deprecated | Avoid for core |

## How I would build this part of Year96

### Core object model

```ts
type ID = string;
type ISODate = string;
type Confidence = number; // 0..1

type WhyNodeKind =
  | 'goal' | 'preference' | 'decision' | 'strategy' | 'process' | 'bug'
  | 'incident' | 'assumption' | 'constraint' | 'metric' | 'observation'
  | 'rationale' | 'counterfactual' | 'lesson';

type WhyEdgeKind =
  | 'because' | 'enables' | 'blocks' | 'causes' | 'contradicts'
  | 'refines' | 'replaces' | 'measures' | 'evidencedBy'
  | 'ownedBy' | 'delegatedTo' | 'verifiedBy';

interface WhyNode {
  id: ID;
  kind: WhyNodeKind;
  statement: string;
  scope: { orgId: ID; ownershipId?: ID; dutyId?: ID; threadId?: ID };
  provenance: Array<{ source: 'human'|'agent'|'sensor'|'builder'|'import'; ref: string; at: ISODate }>;
  confidence: Confidence;
  validFrom: ISODate;
  validUntil?: ISODate;
  reviewBy?: ISODate;
  privacy: 'public-org'|'restricted'|'private-human'|'secret';
}

interface WhyEdge { id: ID; from: ID; to: ID; kind: WhyEdgeKind; weight: number; evidence?: ID[]; }
interface WhyGraph { nodes: WhyNode[]; edges: WhyEdge[]; version: string; }

interface MentalModel {
  humanIds: ID[];
  values: WhyNode[];
  preferences: WhyNode[];
  antiPreferences: WhyNode[];
  style: Record<string, unknown>;
  riskTolerance: Record<string, number>;
  knownUnknowns: string[];
  elicitationState: { nextQuestions: string[]; fatigueBudget: number; lastInterview?: ISODate };
}

interface OptimalVector {
  id: ID;
  dimensions: Array<{
    key: 'revenue'|'learning'|'risk'|'time'|'cost'|'quality'|'trust'|'humanEnergy'|'reversibility'|string;
    direction: 'maximize'|'minimize'|'target';
    weight: number;
    hardConstraint?: boolean;
    target?: number;
  }>;
  paretoPolicy: 'human-weighted'|'lexicographic'|'constraint-first'|'portfolio';
  embeddingGoal?: number[];
  lastCalibratedAt: ISODate;
}

interface StrategyFrame {
  id: ID;
  kind: 'okr'|'gqm'|'ost'|'impact-map'|'wardley'|'ooda'|'custom';
  horizon: 'short'|'quarter'|'year'|'multi-year';
  claims: ID[];
  metrics: ID[];
  reviewCadence: string;
}

interface Sensor {
  id: ID;
  name: string;
  provider: string;
  query: unknown;
  cadence: string;
  triggerPolicy: 'notify'|'question'|'review'|'auto-duty-proposal'|'auto-builder-request';
  scopeEffectHints: string[];
  permissionRef: ID;
}

interface Ownership {
  id: ID;
  title: string;
  humanOwners: ID[];
  accountableOwner: ID;
  whyGraphRef: ID;
  mentalModelRef: ID;
  strategies: { long: StrategyFrame[]; short: StrategyFrame[] };
  sensors: Sensor[];
  optimalVector: OptimalVector;
  memory: { semanticRefs: ID[]; episodicRefs: ID[]; proceduralRefs: ID[] };
  duties: ID[];
  charter: { purpose: string; domains: string[]; accountabilities: string[]; permissions: ID[]; forbiddenActions: string[] };
  reviewCadence: string;
  createdAt: ISODate;
  updatedAt: ISODate;
}

interface Duty {
  id: ID;
  ownershipId: ID;
  title: string;
  scope: { domain: string; boundaries: string[]; outOfScope: string[] };
  knowledgeBaseRefs: ID[];
  settings: Record<string, unknown>;
  ongoingProcesses: Array<{ id: ID; cadence: string; sla: string; status: 'healthy'|'blocked'|'degraded' }>;
  insightBacklog: Array<{ id: ID; whyNodeId: ID; priority: number; proposedAction?: string }>;
  builderRequests: ID[];
  slas: Array<{ metric: string; target: string; escalationThreadId: ID }>;
  permissions: ID[];
}

interface BuilderRequest {
  id: ID;
  dutyId: ID;
  goal: string;
  why: { summary: string; whyNodeIds: ID[] };
  references: Array<{ kind: 'thread'|'doc'|'state'|'decision'|'artifact'; ref: string }>;
  acceptanceCriteria: string[];
  proofCriteria: Array<{ level: 'unit'|'integration'|'mocked-integration'|'e2e'|'mocked-e2e'|'agentic-verifier'; required: boolean; oracle: string }>;
  capabilities: string[];
  permissions: ID[];
  tools: string[];
  deadline: ISODate;
  budget?: { money?: number; tokens?: number; wallClock?: string };
  escalation: { ifBlockedAfter: string; threadId: ID };
}
```

### Provider interfaces

```ts
interface OwnershipStore {
  get(id: ID): Promise<Ownership>;
  listByHuman(humanId: ID): Promise<Ownership[]>;
  save(next: Ownership, proof: { actor: ID; reason: string }): Promise<void>;
}
interface MentalModelProvider {
  infer(evidence: unknown[], current: MentalModel): Promise<{ model: MentalModel; deltas: WhyNode[] }>;
  proposeQuestions(model: MentalModel, budget: number): Promise<string[]>;
}
interface WhyGraphProvider {
  upsert(nodes: WhyNode[], edges: WhyEdge[]): Promise<void>;
  explain(query: string, scope: unknown): Promise<{ answer: string; path: ID[]; confidence: Confidence }>;
}
interface StrategyPlanner {
  propose(ownership: Ownership, observations: WhyNode[]): Promise<StrategyFrame[]>;
  score(options: StrategyFrame[], vector: OptimalVector): Promise<Array<{ option: ID; score: number; paretoRank: number; rationaleNodeIds: ID[] }>>;
}
interface SensorProvider { poll(sensor: Sensor): Promise<WhyNode[]>; subscribe(sensor: Sensor, cb: (n: WhyNode)=>void): Promise<void>; }
interface ObjectiveProvider { calibrate(owner: Ownership, feedback: unknown[]): Promise<OptimalVector>; }
interface DutyManager {
  planCrud(owner: Ownership, strategyDeltas: StrategyFrame[]): Promise<Array<{ op:'create'|'update'|'delete'; duty: Partial<Duty>; why: ID[] }>>;
  apply(plan: unknown, approval: ID): Promise<Duty[]>;
}
interface BuilderRequestor { request(req: BuilderRequest): Promise<{ accepted: boolean; builderId?: ID; threadId: ID }>; }
interface VerificationProvider { verify(req: BuilderRequest, artifactRefs: string[]): Promise<{ passed: boolean; proofs: string[]; whyUpdates: WhyNode[] }>; }
```

### Main loops

1. **Sense.** #01 state fabric and #02 scope-effect deliver events: timers, thoughts, user edits, market changes, incidents, Builder results, dormant-thread reactivations. Sensors annotate scope and authority.
2. **Interpret.** Ownership updates the why graph: classify observations, link causes/evidence, detect contradictions, decay stale confidence, and ask #03 thread memory for relevant context.
3. **Strategize.** StrategyPlanner runs OODA/OKR/GQM/OST frames, scores options against OptimalVector, and produces strategy deltas. Human review is required when authority or values thresholds are crossed.
4. **CRUD Duties.** Owner creates/updates/deletes Duties only. It never opens tools, writes code, buys ads, or edits assets. A policy gate rejects execution calls from Ownership identities.
5. **Request Builders.** Duties convert insight backlog into BuilderRequest contracts with why, references, acceptance/proof criteria, permissions, tools, deadline, and escalation. #06 attenuates permissions; #05 runs durable workflows; #04 communicates.
6. **Verify outcomes.** #09 verifies artifacts and outcomes against proof criteria, not against agent self-report. Failed proofs become why nodes and lessons.
7. **Learn.** #08 promotes improvements: update mental model, recalibrate optimal vector, revise strategies, change Duties, and schedule sleep-time consolidation.

Cadences: fast triggers for safety/revenue incidents; hourly/daily for sensors; weekly tactical Duty review; monthly strategy review; quarterly mental-model interview; continuous sleep-time consolidation within budget. #02 thought generation can create synthetic prompts such as "What would make this Ownership obsolete?"; #05 timers guarantee checks.

### Worked decomposition: "start Facebook ads for my business"

Ownership: `growth-acquisition` with purpose "profitably acquire customers while protecting brand trust." Initial why graph captures business model, target customer, margin, brand constraints, previous marketing beliefs, and risk limits. OptimalVector weights revenue and learning high, risk/cost bounded.

Duties created/updated:
- `audience-market-duty`: owns ICP hypotheses, competitor ads, offer angles, exclusions.
- `tracking-duty`: owns pixel/CAPI, consent, analytics, attribution, dashboards.
- `creative-duty`: owns ad concepts, brand voice, landing page message match.
- `budget-experiment-duty`: owns test design, spend caps, stop-loss rules, learning agenda.
- `compliance-duty`: owns Meta policies, privacy, claims review.

BuilderRequests: install/verify tracking; draft campaign structure; produce three creative variants; build landing page experiment; create dashboard; launch a low-budget campaign after human review. Each request includes why, proof criteria (pixel event test, policy review, budget cap, dashboard screenshot, expected CPA range), permissions (Meta Ads sandbox/live with spend ceiling), tools, and deadline. The Owner does not create ads; Duties request Builders and review verified outcomes.

### Worked decomposition: "build an Unreal Engine game"

Ownership: `game-studio-ownership` with purpose "ship a playable, distinctive Unreal game without bankrupting the studio or losing creative coherence." Mental model captures desired genre, player fantasy, art constraints, hardware target, monetization ethics, team skill, and tolerance for prototype churn. OptimalVector balances fun, feasibility, novelty, cost, schedule, and learning.

Duties:
- `creative-direction-duty`: pillars, references, tone, player promise.
- `technical-architecture-duty`: Unreal version, plugins, performance budgets, CI, asset pipeline.
- `prototype-duty`: core loop, controls, combat/interactions, greybox levels.
- `production-duty`: milestones, backlog, vendor/asset sourcing.
- `qa-playtest-duty`: playtest plans, bug taxonomy, telemetry.
- `business-duty`: platform, budget, funding, store requirements.

BuilderRequests: create a one-page GDD from why graph; prototype core movement/combat; set up Unreal CI and version control; implement telemetry; run 5 playtest sessions; produce vertical-slice acceptance checklist. Proof criteria include packaged build, FPS budget, crash-free smoke test, player task success, source control reproducibility, and #09 agentic review.

### Scaling path

Laptop: SQLite/event log + local vector DB + LangGraph/Temporal dev server + file-backed why graph. Team server: Postgres/pgvector or graph DB provider, Temporal cluster, object store, policy engine, observability. Cluster: sharded OwnershipStore by organization/thread, append-only event sourcing from #01, multi-tenant permission gates from #06, queue-based sensor fanout, budget-aware model routing from #05/#11. State remains behind interfaces so providers can swap.

### Testing and proof plan

Apply the 70% rule. Unit-test scoring, Pareto filters, why-graph migrations, permission denials, and BuilderRequest schemas. Integration-test OwnershipStore/MentalModel/WhyGraph providers against seeded histories. Mocked E2E: feed synthetic business/game desires and assert the Owner creates Duties but cannot execute. E2E: run a sandbox BuilderRequest and verify proof capture. Agentic verifier: ask a separate #09 verifier to reconstruct why from graph paths and detect unsupported decisions. Regression benchmarks: METR-style task horizons per Duty class, mental-model precision/recall against human-labeled preferences, and authority-calibration tests for notify/question/review/act choices.

## What is still unsolved (late 2026)

- **Authority calibration.** There is no settled science for when an agent should silently learn, ask, notify, review, or act. Year96 must learn this per human and prove it does not drift into overreach.
- **Faithful mental models.** Preferences are contextual, contradictory, and change under reflection. The system must represent uncertainty and ask good questions without exhausting the human.
- **Why-graph truth maintenance.** Rationale graphs will contain stale, false, political, or self-serving explanations. Year96 needs contradiction detection, provenance, expiration, and adversarial review.
- **Long-horizon reliability.** METR's 2026 update shows measurement itself is hard. Ownership should not assume days-long autonomy is safe; it needs checkpoints, simulation, rollback, and proof gates.
- **Multi-agent cost and coordination.** Anthropic's results imply performance often comes from more tokens/context. Year96 needs budget-aware spawning and anti-duplication controls.
- **Optimal vector legitimacy.** Multi-objective scoring can smuggle values into weights. Humans must review weights, constraints, and tradeoffs; Duties should expose Pareto alternatives, not hide them.
- **Privacy and governance.** Mental models and why graphs are highly sensitive. #06 must enforce purpose limitation, clone attenuation, audit, and deletion/retention policy.
- **Evaluation of proactive benefit.** Pulse-like briefings can become noise. Year96 needs outcome metrics for ambient ownership: saved time, avoided risk, learning produced, false alarms, and trust impact.

## Sources

1. https://arxiv.org/html/2309.02427
2. https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/
3. https://metr.org/time-horizons/ and https://metr.org/blog/2026-1-29-time-horizon-1-1/
4. https://www.langchain.com/blog/introducing-ambient-agents
5. https://www.letta.com/blog/sleep-time-compute/
6. https://openai.com/index/introducing-chatgpt-pulse/
7. https://www.anthropic.com/engineering/multi-agent-research-system
8. https://github.com/alextamkin/generative-elicitation and https://arxiv.org/abs/2310.11589
9. https://proceedings.iclr.cc/paper_files/paper/2025/hash/c9867d5a22653ce98b02595061e40f12-Abstract-Conference.html
10. https://arxiv.org/abs/2504.08066
11. https://arxiv.org/abs/2501.04227
12. https://arxiv.org/abs/2504.13171
13. https://github.com/SakanaAI/AI-Scientist-v2 and its LICENSE fetched via GitHub MCP
14. https://github.com/gao-g/prelude and its LICENSE fetched via GitHub MCP
15. https://arxiv.org/abs/2210.03629
16. https://arxiv.org/abs/2303.11366
17. https://arxiv.org/abs/2305.10601
18. https://arxiv.org/abs/2310.04406
19. https://adr.github.io/madr/
20. https://www.holacracy.org/constitution
21. https://www.microsoft.com/en-us/worklab/work-trend-index/frontier-firm
22. https://github.com/Temporalio/sdk-typescript
23. https://github.com/langchain-ai/langgraph and LICENSE fetched via GitHub MCP
24. https://github.com/letta-ai/letta and `letta-ai/letta-code` LICENSE fetched via GitHub MCP
25. https://github.com/SamuelSchmidgall/AgentLaboratory and LICENSE fetched via GitHub MCP
26. https://github.com/langchain-ai/open-agent-platform
