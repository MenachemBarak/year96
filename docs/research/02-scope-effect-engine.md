# 02 — Scope-Effect Engine

Scope: this report owns Year96's Scope-Effect Engine: the system that turns #01 State Fabric deltas and self-generated thoughts into per-identity/per-thread predictions of relevance, magnitude, probability, and horizon; routes the result to #04 Communication Hub listeners/hangers; and learns from outcomes. The design assumes Year96 is an interface-first, provider-based OS where logic is separated from state, all relevant state is searchable, and proofs/observability consume most engineering effort.

## TL;DR for the Year96 architect

- Scope effect is not one model. Build it as a cascade: deterministic subscriptions and graph blast radius first, embedding/temporal-graph retrieval second, learned per-identity ranking third, and LLM/causal adjudication only for high-value candidates.
- Treat every delta as a forecastable proposition: `P(matter | identity, thread, delta, horizon)`, expected impact, uncertainty, explanation, and expiry. Store predictions so Year96 can score Brier/log loss later.
- Adopt proven infrastructure patterns: Nx/Bazel/Turborepo-style affected graphs generalize cleanly to “which org processes might be affected?” [15][16][17]; Feast gives training/serving feature consistency [6]; Flink/RisingWave/Feldera provide standing stream queries [7][8][9].
- Use DoWhy/EconML/causal-learn for offline causal analyses and counterfactual evaluation, but not as the real-time relevance engine. DoWhy’s first-class assumptions and validation are useful for “why did this matter?” audits [1]. Tigramite is strong for time-series causality but GPL-3.0, so exclude from core.
- Temporal graph learning is a Trial, not the foundation. TGB and PyG Temporal are good benchmarks/toolkits [2][3], but Year96 needs interpretable, replayable routing decisions; GNN embeddings become candidate generators/features.
- Relevance learning should be a contextual-bandit/ranking problem with off-policy replay. Vowpal Wabbit (BSD-style) and Open Bandit Pipeline (Apache-2.0, older activity) are useful, but Year96 should hide them behind providers.
- Watchlists are a first-class product: low-probability/high-impact items should decay, refresh on weak signals, and be periodically re-adjudicated rather than spamming humans.
- Thought generation is not “free autonomous brainstorming.” It is a budgeted synthetic delta source. Letta sleep-time compute [10], LangChain ambient agents [19], OpenAI Pulse [18], and ProactiveBench [20] converge on idle/ambient proactivity, but Year96 needs stricter accounting: budgets, anti-looping, novelty checks, and outcome scoring.
- Forecasting research gives the calibration target. ForecastBench frames forecasts as dynamic, scored probability judgments [11][12]; Year96 should score scope-effect forecasts with Brier/log loss, lead time, regret, and alert fatigue.
- Use horizon scanning methods for “rocket company -> ingredient competitor”: weak-signal mining over patents/papers/news, technology-convergence embeddings, and option-value watchlists. JRC’s 2024/2025 weak-signal report found 221 signals via text mining over Scopus and PATSTAT [22].
- For #04, output routing decisions must distinguish `notify-listener`, `wake-hanger`, `spawn-thought`, and `escalate`; the Scope-Effect Engine should not own communication policy beyond producing auditable relevance decisions.
- Biggest innovation: learning long-horizon “mattered later” labels. Short-term notifications can be supervised; five-year scope effects require replay, synthetic counterfactuals, prediction markets/forecast labels, and human adjudication.

## Landscape

### Causality and counterfactuals

**PyWhy DoWhy and EconML.** DoWhy is the best fit for auditable causal inference because it separates identification from estimation, makes assumptions explicit, and includes refutation/sensitivity checks [1]. EconML contributes heterogeneous treatment-effect estimators, useful when asking whether an intervention (“notify owner”, “wake hanger”, “put on watchlist”) changed outcomes. Both are permissive after license-file verification: DoWhy MIT; EconML license file says MIT despite GitHub API `NOASSERTION`. For Year96: use offline for causal review of routing policies, not for every delta.

**causal-learn and Tigramite.** causal-learn gives constraint/score-based causal discovery in Python, MIT, active. It can infer candidate dependency edges among state features when #01 State Fabric has enough observational data. Tigramite focuses on time-series causal discovery and is active, but its GPL-3.0 license makes it excluded from Year96 core; use only as an external optional integration.

**LLM causal discovery and causal world models.** 2025–2026 papers corrected the lead’s rough memory: LLMs can assist causal graph discovery but are not reliable causal reasoners alone. The 2025 survey “Large Language Models for Causal Discovery” frames LLMs as domain-prior and graph-refinement tools [13]. Causal-LLM (EMNLP Findings 2025) proposes one-shot prompt/data-driven graph discovery [14]. “Language Agents Meet Causality” (ICLR 2025) uses causal representation learning as a queryable world model for LLM planning and outperforms LLM-only reasoners on long horizons [21]. For Year96: use LLMs to propose causal hypotheses and explanations, then validate with data/replay.

### Change-impact and affected-graph analogues

**Bazel, Nx, Turborepo.** Build systems already solve a narrow version of scope effect: given a changed file/target, compute affected downstream tasks. Bazel query exposes dependency/path analysis over target graphs [17]. Nx `affected` determines the minimum projects affected by a change to reduce CI work [15]. Turborepo now exposes package/task graph inspection and `turbo query` GraphQL [16]. Year96 should generalize this from code targets to organization entities: policies, identities, duties, threads, suppliers, markets, tools, memories, documents, and sensors.

**Blast-radius and supply-chain risk products.** Commercial SRE/security platforms do this with dependency maps and risk propagation. The reusable pattern is: normalize resources into a graph, annotate edge semantics/strength, propagate change over typed edges, score reachable nodes by context, and explain the path. Year96 should implement this internally, not buy the core, because the graph includes private thoughts and mental models.

### Temporal graph learning and future relevance

**TGB and PyG Temporal.** TGB provides reproducible temporal graph datasets, splits, and evaluators for temporal graphs, temporal KGs, and heterogeneous graphs; it exports NumPy, PyTorch, and PyG `TemporalData` objects [2]. PyTorch Geometric Temporal implements spatiotemporal/dynamic GNN models [3]. For Year96: benchmark link prediction such as “this delta later mattered to this thread,” train embeddings for candidate generation, and compare simple baselines (EdgeBank, graph proximity) against neural methods. Do not make a black-box temporal GNN the final router.

**Knowledge graph embeddings and spreading activation.** Embedding similarity and random-walk/spreading activation are the right cheap candidate-generation primitives. They catch non-obvious adjacency (rocket -> cryogenics -> protein crystallization -> food ingredient) before expensive LLM adjudication.

### Recsys, ranking, feature stores, and interruption management

**Feast.** Feast is an Apache-2.0 feature store with offline/online stores, low-latency serving, point-in-time correct training features, and Python/feature-server APIs [6]. This is directly aligned with per-identity relevance profiles: if serving features differ from training features, the engine will miscalibrate. Adopt.

**Contextual bandits and online learning.** Vowpal Wabbit is mature, fast, and designed for online/contextual learning; its license file is BSD-style permissive though GitHub does not classify it. Open Bandit Pipeline is Apache-2.0 for off-policy evaluation, but last push observed in 2024, so Trial/Watch. Year96 needs bandits because routing policies both exploit known relevance and explore uncertain watchlist paths without flooding humans.

**Alert fatigue and interruption research.** The 2025 literature and deployed ranking systems confirm the obvious product truth: the cost of an interruption must be modeled, not bolted on. Scope decisions need expected value minus attention cost, current cognitive load, and user-stated notification policy.

### Stream reasoning and complex event processing

**Flink CEP, Siddhi, Esper, RisingWave, Feldera.** FlinkCEP detects event patterns in endless streams [7]; Apache Flink is highly mature and Apache-2.0. Siddhi is Apache-2.0 and CEP-focused, but smaller. Esper is GPL-2.0 and excluded from core. RisingWave’s 2026 docs explicitly position it as an event streaming platform for agentic AI, unifying ingestion, stream processing, low-latency serving, and materialized views [8]. Feldera claims full incremental SQL, consistency/freshness guarantees, connectors, and laptop-scale high throughput [9]; license file says OSS edition MIT with separate enterprise portions. Recommendation: Adopt Flink/RisingWave for scalable streams; Trial Feldera for incremental standing queries; avoid Esper in core.

**Semantic pub/sub and LLM predicates.** Emerging semantic CEP papers are promising but young. Use a safer cascade: typed predicates and embeddings first; LLM-evaluated predicates only after deterministic filters, with caching, sampling audits, and bounded spend.

### Weak signals and horizon scanning

Government foresight practice is directly relevant to long-horizon scope. UK Defra’s case study uses horizon scanning for weak signals and emerging trends [23]. JRC’s 2025 report detected 221 weak technology signals across 12 clusters using text mining, clustering, scientometric indicators, Scopus, and PATSTAT [22]. For Year96, these methods become external sensors (#07) and watchlist refreshers: patents, papers, hiring, procurement, code repos, standards, regulation, and news.

### Forecasting, calibration, and active inference

**ForecastBench and LLM forecasting.** ForecastBench is a dynamic benchmark for AI forecasting capabilities [11][12]. Halawi et al.’s 2024 “Approaching Human-Level Forecasting with Language Models” (not opened directly in this timebox) is part of the lineage. Year96 should borrow the discipline: every `might matter` is a probability distribution over time, scored when resolved.

**Predictive processing/world models.** V-JEPA 2 (June 2025) learns latent video world models for understanding, prediction, and planning [24]; the GitHub repo reports V-JEPA 2.1 in March 2026 and is MIT [25]. This matters as architectural inspiration: predict in compact latent space and focus on variables that help future control. It is not directly adoptable for org-level scope except as a research analogy.

### Proactive and ambient agents / thought generation

**Generative Agents.** Park et al. 2023 established memory stream, reflection, and planning loops. Year96’s thought generator should borrow the reflection cadence, but harden it with budgets and outcome scoring.

**Letta sleep-time compute.** Letta argues agents should reason during idle time, transforming raw context into learned context before the user asks [10]. This is the closest direct analogy to Year96 “thoughts create state.”

**LangChain ambient agents.** LangChain defines ambient agents as agents that listen to event streams and act on multiple events simultaneously; it emphasizes human-in-the-loop patterns of notify, question, and review [19]. This maps to Year96 routing decisions.

**OpenAI ChatGPT Pulse.** OpenAI’s Pulse launch frames proactive updates from prior conversation/context, surfacing logical next steps without a prompt [18]. Product lesson: proactivity must be useful, brief, and preference-aware.

**ProactiveBench / ProAgentBench / ProAct.** ProactiveBench (ICLR 2025) collected 6,790 events and trained/evaluated proactive assistance with human accept/reject labels, reaching 66.47 F1 for a fine-tuned model [20]. 2026 ProAgentBench and ProAct/ProActEval (opened as arXiv stubs [26][27]) show the field moving to real-world sessions and idle-time anticipation. Use as evaluation templates, not production dependencies.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| PyWhy DoWhy | OSS | Auditable causal assumptions, identification/estimation separation, refutation | MIT, GitHub API | 8,332 stars; pushed 2026-09-28; PyWhy | Adopt |
| PyWhy EconML | OSS | Heterogeneous treatment effects for policy impact | MIT, LICENSE file verified | 4,801 stars; pushed 2026-09-25; PyWhy/Microsoft lineage | Adopt |
| causal-learn | OSS | Causal discovery candidate edges | MIT, GitHub API | 1,692 stars; pushed 2026-09-04; PyWhy | Trial |
| Tigramite | OSS | Time-series causal discovery | GPL-3.0, GitHub API | 1,726 stars; pushed 2026-01-14 | Excluded-license |
| TGB | OSS/benchmark | Temporal graph benchmark and evaluators | MIT, GitHub API | 264 stars; pushed 2026-07-27 | Trial |
| PyTorch Geometric Temporal | OSS | Dynamic GNN models/loaders | MIT, GitHub API | 2,991 stars; pushed 2026-05-30 | Trial |
| Feast | OSS | Offline/online feature consistency for relevance profiles | Apache-2.0, GitHub API | 7,311 stars; pushed 2026-09-28 | Adopt |
| Vowpal Wabbit | OSS | Online/contextual bandit ranking | BSD-style permissive, LICENSE file verified | 8,723 stars; pushed 2026-09-28 | Trial |
| Open Bandit Pipeline | OSS | Off-policy evaluation for bandit policies | Apache-2.0, GitHub API | 710 stars; pushed 2024-06-03 | Watch |
| Apache Flink/FlinkCEP | OSS | Event streams and CEP patterns | Apache-2.0, GitHub API | 26,372 stars; pushed 2026-09-28 | Adopt |
| Siddhi | OSS | Lightweight CEP engine | Apache-2.0, GitHub API | 1,593 stars; pushed 2026-05-05 | Trial |
| Esper | OSS/product | CEP, streaming SQL | GPL-2.0, LICENSE file verified | 876 stars; pushed 2024-04-26; page modified 2026 | Excluded-license |
| RisingWave | OSS/product | SQL event streaming, materialized views, serving | Apache-2.0, GitHub API | 9,352 stars; pushed 2026-09-28 | Adopt |
| Feldera | OSS/product | Incremental SQL, consistency/freshness, connectors | MIT OSS edition; enterprise portions, LICENSE file | 2,107 stars; pushed 2026-09-28 | Trial |
| Bazel | OSS | Dependency query/affected graph pattern | Apache-2.0, GitHub API | 25,899 stars; pushed 2026-09-28 | Adopt-pattern |
| Nx | OSS/product | Affected project graph and CI minimization | MIT, GitHub API | 29,380 stars; pushed 2026-09-28 | Adopt-pattern |
| Turborepo | OSS/product | Package/task graph query pattern | MIT, GitHub API | 31,153 stars; pushed 2026-09-28 | Adopt-pattern |
| Letta | OSS/product | Stateful agents and sleep-time compute | Apache-2.0, GitHub API | 24,953 stars; pushed 2026-09-10 | Trial |
| hermes-agent | OSS | Required harness option for repetitive monitored automations | MIT, LICENSE file verified | 249,726 stars; pushed 2026-09-28; anomalously high, verify again before procurement | Trial |
| V-JEPA 2 | OSS/paper | Latent predictive world-model inspiration | MIT, GitHub API | 4,696 stars; pushed 2026-03-23; Meta | Watch |

## How I would build this part of Year96

### End-to-end pipeline

1. **Delta ingest.** Subscribe to #01 State Fabric streams: entity changes, document changes, messages, sensor observations, tool traces, external weak signals, and synthetic thought events.
2. **Normalize.** Convert each raw delta into a typed `StateDelta` with entity refs, provenance, embeddings, textual summary, causality hints, permissions, and time.
3. **Candidate generation.** Run multiple cheap providers in parallel: exact subscriptions; hanger/listener constraints from #04; affected-graph traversal; temporal graph proximity; embedding similarity; standing CEP queries; watchlist triggers; identity/thread memory retrieval from #03; sensors from #07.
4. **Feature hydrate.** Feast-backed online feature retrieval builds `RelevanceFeatureVector`: actor/thread preferences, past outcomes, attention budget, current load, graph path features, novelty, source trust, horizon priors, and possible impact.
5. **Learned scoring.** A pairwise/listwise ranker plus contextual bandit estimates relevance probability, impact distribution, horizon buckets (`now`, `hours`, `days`, `months`, `years`), and uncertainty per identity/thread.
6. **LLM/causal adjudication for top-k.** For expensive candidates only, ask an adjudicator to produce an explanation, counterfactual, missing-evidence checklist, and route recommendation. It must cite state refs and cannot invent permissions.
7. **Routing.** Policy emits one of `{ignore, log-for-later, watchlist, notify-listener, wake-hanger, spawn-thought, escalate}`. Outputs include probability, expected impact, horizon, reason path, expiry, and feedback contract.
8. **Feedback/outcomes.** Capture whether humans/agents clicked, ignored, acted, marked useful/noisy, whether milestones changed, whether watchlisted risk materialized, and counterfactual replay labels from #09.
9. **Calibration.** Update Platt/isotonic/conformal calibration per identity, thread type, horizon, and source. Publish reliability curves and alert-fatigue budgets.

### Data model

```ts
type Horizon = 'now' | 'hours' | 'days' | 'weeks' | 'months' | 'years' | '5y_plus';
type Route = 'ignore' | 'log-for-later' | 'watchlist' | 'notify-listener' | 'wake-hanger' | 'spawn-thought' | 'escalate';

interface StateDelta {
  id: string;
  occurredAt: string;
  source: string;
  kind: 'entity' | 'doc' | 'message' | 'sensor' | 'external' | 'thought';
  actorIdentityId?: string;
  entityRefs: string[];
  threadRefs?: string[];
  textSummary: string;
  embeddingRef?: string;
  provenance: { uri?: string; stateVersion: string; trust: number; permissionScope: string[] };
}

interface RelevanceProfile {
  identityId: string;
  threadId?: string;
  goals: string[];
  antiGoals: string[];
  domains: string[];
  horizonWeights: Record<Horizon, number>;
  interruptionBudget: { dailyCost: number; maxEscalations: number; quietHours: string[] };
  explorationRate: number;
  positiveSignals: string[];
  negativeSignals: string[];
  calibrationBucket: string;
}

interface ScopePrediction {
  deltaId: string;
  identityId: string;
  threadId?: string;
  pMatters: number;
  impact: { expected: number; low: number; high: number; unit: 'money' | 'time' | 'risk' | 'goal' | 'attention' };
  horizon: Horizon;
  uncertainty: number;
  route: Route;
  explanation: string;
  evidenceRefs: string[];
  expiresAt?: string;
  watchlistId?: string;
}

interface WatchlistItem {
  id: string;
  ownerIdentityId: string;
  threadId?: string;
  thesis: string;
  triggerPredicates: string[];
  pCurrent: number;
  impactIfTrue: number;
  optionValue: number;
  decay: { halfLifeDays: number; lastRefreshedAt: string };
  nextReviewAt: string;
  evidenceRefs: string[];
}
```

### Provider interfaces

```ts
interface DeltaSource { subscribe(cursor: string, emit: (d: StateDelta) => Promise<void>): Promise<void>; }
interface CandidateGenerator { generate(delta: StateDelta, ctx: ScopeContext): Promise<Candidate[]>; }
interface RelevanceScorer { score(candidates: Candidate[], features: FeatureVector[]): Promise<Score[]>; }
interface Adjudicator { adjudicate(top: ScoredCandidate[], ctx: ScopeContext): Promise<Adjudication[]>; }
interface RoutingPolicy { decide(a: Adjudication, budgets: BudgetState): Promise<ScopePrediction>; }
interface OutcomeLabeler { label(prediction: ScopePrediction, window: TimeWindow): Promise<OutcomeLabel[]>; }
interface FeedbackProvider { record(label: OutcomeLabel): Promise<void>; }
interface ThoughtGenerator { tick(ctx: ThoughtContext): Promise<StateDelta[]>; }
interface CalibrationProvider { calibrate(raw: Score, bucket: string): Promise<CalibratedScore>; }
```

### WATCHLIST mechanics

A watchlist item is not an alert queue. It is an option-value contract: “spend small recurring budget to keep this possibility alive.” Score `optionValue = pCurrent * impactIfTrue * learningValue - carryingCost - attentionRisk`. Items decay unless refreshed by new evidence. Re-evaluation schedules are horizon-sensitive: daily for live operational risks, monthly/quarterly for strategic weak signals, yearly for five-year convergence bets. Expired watchlists are archived with reasons so replay can learn whether decay was too aggressive.

### THOUGHT GENERATOR

The Thought Generator is a `DeltaSource` that emits synthetic `kind:'thought'` deltas. It runs on schedules, idle windows, unresolved tension, stale ownerships, watchlist review dates, and explicit #07 sensors. It has four modes:

1. **Reflection:** summarize recent outcomes, update “why” and relevance profile candidates.
2. **Prospection:** simulate near-future needs and ask “what would I regret not checking?”
3. **Maintenance:** revisit dormant hangers/watchlists and decay or refresh them.
4. **Curiosity:** bounded exploration of weak signals connected to high-impact goals.

Anti-runaway controls: per-identity compute budgets, max thought-depth, duplicate/novelty filters, no recursive thought spawning without a new external/state predicate, spend caps by horizon, mandatory expiry, and negative reward for thoughts that produce no useful downstream outcome. Every thought carries `parentReason`, `budgetDebited`, and `stopCondition`. For starving Yossi, the world is frozen but `physiological_need: hunger` and memory/goals generate a thought delta: “seek food soon.” The scorer routes it to wake the food-seeking duty because probability and impact are high even without external change.

### Worked example 1: rocket company -> food-ingredient competitor

A patent sensor ingests a rocket-company patent on cryogenic microencapsulation. Candidate generation finds no direct subscription from the ingredient startup, but graph proximity links `cryogenic systems -> preservation -> protein stability -> food ingredients`; embedding similarity matches old thread memories about shelf-stable proteins; JRC-style weak-signal sensors see related papers and hiring. The learned scorer assigns `pMatters=0.08`, `impactIfTrue=0.9`, horizon `5y_plus`, high uncertainty. Routing chooses `watchlist`, not notify. Six months later a supplier partnership and food-science hires refresh the watchlist to `p=0.21`, triggering an LLM adjudication and `notify-listener` to the strategy thread. Two years later a product announcement labels the watchlist positive; calibration rewards weak-signal chain features.

### Worked example 2: starving Yossi

External state is frozen. The Ownership layer still has goals (`stay alive`), body-state memory (`hungry`), and internal time/need dynamics. The Thought Generator emits `thought:yossi:food-seeking-intention`. Candidate generators match the survival ownership and active self-thread. Scorer: `pMatters=0.99`, impact high, horizon `now`; route `wake-hanger` for the food acquisition duty and perhaps `escalate` if blocked. If the thought loops without action, anti-runaway turns it into a single escalation: “food-seeking blocked because execution cannot change frozen world,” rather than endless rumination.

### Scaling path

Laptop: SQLite/Postgres + pgvector, lightweight event bus, local Feast, simple graph traversal, small ranker, cached LLM adjudication. Team/org: RisingWave/Flink for streams, Neo4j/JanusGraph or Postgres graph tables, vector DB, Feast online store, batched scoring, replay harness. Enterprise cluster: Kafka/Pulsar, Flink/RisingWave, distributed feature store, temporal graph training, multi-tenant policy isolation, model gateway, and #09 deterministic replay.

### Testing and proof (70% rule)

- Unit: route policy thresholds, decay math, calibration transforms, permission filters.
- Integration: replay known deltas through candidate/scorer/adjudicator with golden predictions.
- Mocked E2E: synthetic org histories with known future mattered labels; verify precision/recall, lead time, Brier score.
- E2E: real historical project/market/company data replayed without future leakage.
- Agentic verifiers: adversarial “alert fatigue”, “missed black swan”, “thought loop”, and “permission leak” agents.
- Observability: every prediction has trace id, features, graph paths, model version, prompt hash, budget debit, and outcome labels.

## What is still unsolved (late 2026)

- Long-horizon labels are sparse, delayed, and ambiguous; Year96 must invent good replay/counterfactual labeling workflows with #09.
- Black-box temporal graph models and LLM adjudicators are hard to explain to humans; final routing must remain auditable.
- Alert fatigue creates hidden harm: even useful alerts can degrade trust if timing is bad.
- Thought generation can become self-referential. Budgets, stop conditions, and novelty checks are mandatory but not sufficient.
- Permission-aware relevance is difficult: the system may know something matters but not be allowed to reveal why.
- Cross-identity learning risks privacy leakage. Relevance profiles need differential access controls and aggregation.
- Five-year option-value tracking has no mature OSS product; Year96 must combine foresight, forecasting, weak-signal mining, and org memory.
- Causal discovery from text and logs remains research-grade. Use it for hypotheses, not autonomous truth.

## Sources

1. https://www.pywhy.org/dowhy/main/
2. https://tgb.complexdatalab.com/
3. https://github.com/benedekrozemberczki/pytorch_geometric_temporal
4. https://api.github.com/repos/py-why/dowhy and related GitHub repo API/license fetches
5. https://raw.githubusercontent.com/py-why/EconML/main/LICENSE
6. https://docs.feast.dev/
7. https://nightlies.apache.org/flink/flink-docs-master/docs/libs/cep/
8. https://docs.risingwave.com/
9. https://docs.feldera.com/
10. https://www.letta.com/blog/sleep-time-compute/
11. https://arxiv.org/abs/2409.19839
12. https://forecastbench.org/
13. https://arxiv.org/abs/2402.11068
14. https://aclanthology.org/2025.findings-emnlp.439/
15. https://nx.dev/ci/features/affected
16. https://turbo.build/repo/docs/crafting-your-repository/understanding-your-repository
17. https://bazel.build/query/language
18. https://openai.com/index/introducing-chatgpt-pulse/
19. https://www.langchain.com/blog/introducing-ambient-agents
20. https://proceedings.iclr.cc/paper_files/paper/2025/hash/75c37811e830bf029584b1c6fac17726-Abstract-Conference.html
21. https://iclr.cc/virtual/2025/poster/27748
22. https://publications.jrc.ec.europa.eu/repository/handle/JRC140959
23. https://www.gov.uk/government/publications/weak-signals-and-trend-analysis-horizon-scanning
24. https://arxiv.org/abs/2506.09985
25. https://github.com/facebookresearch/vjepa2
26. https://arxiv.org/abs/2602.04482
27. https://arxiv.org/abs/2605.25971
28. https://raw.githubusercontent.com/VowpalWabbit/vowpal_wabbit/master/LICENSE
29. https://raw.githubusercontent.com/feldera/feldera/main/LICENSE
30. https://raw.githubusercontent.com/espertechinc/esper/master/LICENSE
