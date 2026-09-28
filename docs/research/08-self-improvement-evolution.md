# 08 — Self-improvement & self-re-architecture

Scope: this report designs Year96's safe continuous self-improvement subsystem: how an Agentic OS can modify prompts, context packs, skills, tools, workflows, agent graphs, routing/fine-tunes, component code, and eventually architecture while preserving immutable identity, permission, audit, rollback, and evaluation invariants. I read the shared brief and the three Year96 concept/spec/vision docs first; the design below treats "the system is part of STATE" as an engineering requirement: every variant, trace, result, gate decision, and rollout is searchable state and can itself trigger scope effects for teammates #01, #02, #06, #09, and #11.

## TL;DR for the Year96 architect

- Build self-improvement as a **progressive-delivery control plane**, not as an unconstrained self-editing agent. The improver proposes variants; an immutable kernel decides what is allowed to run, where it runs, when it can see real traffic, and how rollback works.
- Use **levels L0-L6**: L0 prompt/context, L1 skill/playbook, L2 tool/config, L3 workflow/agent graph, L4 model routing/fine-tune/RL, L5 component code, L6 architecture. Each level has an Improvement Ownership Duty with owned benchmarks, allowed mutation surfaces, and promotion policy.
- Adopt **DGM's open-ended archive idea** but not DGM's safety posture as-is. DGM shows recursive self-modification can improve coding agents by validating variants on SWE-bench/Polyglot and keeping stepping stones [5][6]. Year96 needs stronger immutable gates, hidden evals, policy-signed logs, and production rollout discipline.
- Treat **HGM/clade metaproductivity** as a watch item: it optimizes for descendant promise, which is valuable for long-horizon self-improvement, but it is very new and coding-benchmark centered [7].
- For prompt/program evolution, **Adopt DSPy + GEPA**. DSPy gives modular LM programs and optimizer hooks [13]; GEPA uses full traces and textual feedback to evolve prompts, configs, code, and agent-architecture text with Pareto selection [12][14].
- For workflow/agent-graph search, **Trial AFlow and ADAS/Meta Agent Search**. AFlow uses MCTS over code-represented agentic workflow space [9]; ADAS frames a meta-agent that writes and evaluates child agents [8]. Restrict them to sandboxed offline search.
- For RL and reward-driven improvement, **Trial OpenPipe ART, Prime Intellect Verifiers/Environments Hub, Microsoft Agent Lightning, verl, and OpenRLHF** behind a TrainingProvider, not in the online kernel, because reward hacking and distribution shift are expected [19][20][21][22][23][24].
- Use **OpenFeature** for variant flags and targeting [25], and **GitOps** via Argo CD or Flux for declarative promotion/rollback in clusters [26][27].
- The **immutable kernel** must include identity/authz, capability attenuation, audit/event-log appenders, sandbox policy, eval harness integrity, benchmark secret management, promotion policy, kill switch, rollback controller, and the rule that the kernel cannot self-modify without human approval.
- Anti-reward-hacking is first-class: public/dev/hidden eval splits; rotating canaries; verifier diversity; invariant checks before metrics; tamper-evident eval logs; adversarial "pass without doing the job" tests.
- Personalization is not "one best agent". Maintain per-human/per-thread variants, bounded by org invariants: a variant may fit Alice's style but is forbidden if it weakens audit, privacy, or permission checks.
- The key Year96 innovation is an **Improvement Ledger**: every mutation, rationale, state snapshot, benchmark outcome, rollout cohort, rollback, and learned preference is queryable state for #01 and scope-effect input for #02.

## Landscape

### Surveys and taxonomies

**A Comprehensive Survey of Self-Evolving AI Agents** (Aug 2025) frames self-evolving systems as a loop over inputs, agent system, environment, and optimizers; it organizes evolution by what changes (models, prompts, memory, tools, workflows), when it changes (intra-task, episodic, lifelong), and how feedback is gathered (environment reward, self-reflection, human feedback, social learning) [11]. For Year96, this taxonomy becomes the L0-L6 mutation ladder and prevents the common mistake of treating "self-improvement" as only prompt optimization or only model fine-tuning.

**A Survey of Self-Evolving Agents: On Path to Artificial Super Intelligence** was reported in search results as a July 2025 survey with similar emphasis on memory, tool, prompt, workflow, and model evolution. I did not open a stable primary page within the time box, so I use it only as confirmation, not as a scored component. Verdict: watch/unverified.

### Open-ended self-modifying agents

**Darwin Gödel Machine (DGM)** (May 2025; Sakana/UBC authors) is the most relevant concrete prior. It is a self-improving coding agent that repeatedly edits its own code, evaluates on coding benchmarks, and stores successful variants in an archive so future variants can branch from any prior stepping stone [5][6]. Its README explicitly warns that it executes untrusted model-generated code and recommends Docker and awareness of destructive behavior [6]. Why it matters: Year96 needs the same archive and empirical gate, but with stronger production-grade safety.

**Huxley-Gödel Machine (HGM)** (2025 paper/repo, ICLR 2026 oral claim in README) extends DGM-like self-rewriting with estimates of the promise of entire subtrees or clades, not just immediate score [7]. Why it matters: Year96's improvement archive should not only select the current champion; it should keep diverse lineages that produce good descendants. Caveat: very new, coding-centric, and not yet a general OS improvement platform.

**AI Scientist / AI Scientist v2 lineage**. Sakana's AI Scientist repo is a system for automatic scientific discovery and has an explicit warning about executing LLM-written code, spawning processes, web access, and the need to containerize and restrict web access [18]. Why it matters: it demonstrates a full idea-experiment-review loop, but Year96 should use it as an offline research proposer, not a production improver.

**AlphaEvolve** (Google DeepMind, 2025) is a Gemini-powered coding agent for algorithm discovery [15]. The official page I opened was partly hard to extract, but it confirms Google positions it as algorithm-discovery work. Why it matters: evolutionary code search works when the objective is crisp and evaluation is cheap; Year96 should copy the shape for L5 bounded components, not for fuzzy human-preference architecture without richer evals.

**OpenEvolve** is an open-source evolutionary coding agent that evolves code against evaluator files, supports Pareto/multi-objective optimization and deterministic runs, and positions itself as an open AlphaEvolve-like tool [16]. Why it matters: a candidate L5 component-code optimizer for bounded functions and algorithms. Its README shows a license badge but I could not verify SPDX through GitHub API due rate limit in the later query; mark license unverified in scorecard.

**ShinkaEvolve / Sakana AI CUDA Engineer** (Sept 2025 technical report page) presents an agentic CUDA kernel discovery, verification, and optimization pipeline using robust-kbench, evolutionary meta-generation, and LLM-based verifiers [17]. Why it matters: excellent example of self-improvement under hard correctness/performance tests. It should inspire Year96's "only optimize what you can verify" rule.

**SICA, STOP, Gödel Agent, Agent0 self-play**. I could not verify stable primary 2025-2026 sources quickly enough. They should remain watch/unverified until teammate #12 or a later pass confirms names, dates, licenses, and claims. Do not base architecture on unverified acronyms.

### Automated agent design and workflow search

**ADAS / Meta Agent Search** (paper 2024, ICLR 2025) defines automated design of agentic systems: a meta-agent invents candidate agent designs, writes code for them, evaluates them, and keeps an archive [8]. Why it matters: this is the research version of Year96 L3-L6 search. Use the pattern, but require sandboxing and immutable gates because generated agents can write arbitrary code.

**AFlow** (ICLR 2025 oral; FoundationAgents repo) automatically generates and optimizes agentic workflows. It represents workflows as code/graphs of LLM nodes and operators, then uses a Monte Carlo Tree Search variant to select, expand, evaluate, and update workflows over benchmarks such as HumanEval, MBPP, GSM8K, MATH, HotpotQA, and DROP [9]. Why it matters: it maps directly to Year96's L3 workflow/agent-graph mutations. Trial it for offline graph search.

**AgentSquare** (Oct 2024 arXiv, used in 2025 literature) searches modular LLM agent designs over planning, reasoning, tool-use, and memory modules [10]. Why it matters: it suggests Year96 should express agent graphs in modular typed components rather than opaque prompts.

**MetaGPT / MGX** is an MIT multi-agent software-company framework with SOP-based roles and 2025 MGX product news [30]. Why it matters mostly for teammate #11; for this report it is a source of structured workflow templates that can be mutated at L1/L3.

**EvoAgentX, MaAS, Google MASS**. I found secondary search references but did not verify primary repo/license in time. Mark watch/unverified.

### Prompt/program optimization

**DSPy** is an MIT framework for programming rather than prompting foundation models. Its README says it supports modular AI systems and algorithms for optimizing prompts and weights across classifiers, RAG pipelines, and agent loops [13]. Why it matters: Year96 should express L0/L1 behavior as typed DSPy-like modules where prompts are mutable text parameters and metrics are explicit.

**GEPA** (July 2025) is the strongest recent prompt/program optimizer. The arXiv title is "Reflective Prompt Evolution Can Outperform Reinforcement Learning" [12]. The repo says GEPA optimizes prompts, code, agent architectures, and configurations by having LLMs read full execution traces, errors, profiling data, and reasoning logs, then mutate candidates with Pareto-aware selection [14]. Why it matters: it exactly matches Year96 Observe -> mine failures -> propose mutation using #09 traces.

**MIPROv2 and SIMBA** are DSPy optimizers (MIPROv2 for instruction/demo search, SIMBA for stochastic introspective mini-batch ascent). I could not open DSPy docs due DNS failure, but DSPy README points to its optimizer papers [13]. Use them as built-in strategies behind PromptOptimizerProvider, with GEPA preferred for trace-rich agent failures.

**TextGrad, Promptbreeder, OPRO** are important historical baselines for textual optimization. Use for literature comparison, not core adoption, unless a later benchmark shows a win on Year96 suites. **Microsoft Trace** was not verified as a mainstream 2025-2026 prompt optimizer with this exact name; Microsoft Agent Lightning is the verified Microsoft RL-for-agent framework [22].

### Skill/experience learning

Voyager-style skill libraries, Anthropic Agent Skills, ACE/Agentic Context Engineering, and hermes-agent learning loops are relevant, but #11 owns the harness/methodology deep dive. For this report, the key design rule is: skills are versioned, evaled, rollbackable state objects, not free-floating markdown.

### RL for agents

**OpenPipe ART** is an Apache-2.0 Agent Reinforcement Trainer for multi-step agents using GRPO; its README says it improves agent reliability by letting LLMs learn from experience and wraps RL training for existing Python applications [19]. GitHub API metadata opened during this run showed 10,783 stars, Apache-2.0, pushed 2026-09-28.

**Prime Intellect Verifiers and Environments Hub**. Verifiers is an MIT library for creating environments to train and evaluate LLMs, integrated with the Environments Hub and prime-rl [21]. The Environments Hub blog emphasizes open, shareable RL/eval environments, eval reports, RL training, and beta sandboxes for secure code execution [20]. GitHub API metadata showed verifiers at 4,655 stars, MIT, pushed 2026-09-28.

**Microsoft Agent Lightning** is an MIT, roughly 3,500-line agentic RL framework for training agents with real harnesses. Its README describes a trainer, API gateway, and rollout controller; native Kubernetes Jobs; and an end-to-end coding-agent example improving SWE-bench Verified while including reward-hacking prevention [22]. GitHub API metadata showed 18,521 stars, MIT, pushed 2026-09-28.

**verl** (ByteDance Seed/community; Apache-2.0 per GitHub API metadata opened) is a flexible, production-ready RL training library for LLMs with modular APIs and integration with FSDP, Megatron-LM, vLLM, and SGLang [23]. **OpenRLHF** is Apache-2.0 per API metadata and its README calls it a production-ready RLHF framework based on Ray + vLLM with an agent-based design paradigm [24]. Trial both for large-scale training, not as the primary eval abstraction.

**SkyRL, rLLM, Nous Atropos** were named in the assignment but not verified in this time box. Watch/unverified.

### Safe deployment, gates, and control literature

**OpenFeature** is an Apache-2.0 vendor-agnostic feature flag specification that separates SDK flag evaluation from any specific flag provider [25]. Why it matters: Year96 variant activation should be a policy-evaluated flag decision over identity, thread, cohort, risk level, and scope.

**Argo CD** and **Flux** are Apache-2.0 GitOps controllers. Argo CD is declarative continuous delivery for Kubernetes [26]; Flux keeps clusters synchronized with Git/OCI sources and is a CNCF graduated project with multi-tenancy and Prometheus ecosystem integration [27]. Why it matters: promotion should happen through signed manifests and reconciliation, not self-editing production boxes.

**Anthropic Responsible Scaling Policy** provides an organizational precedent for safety levels, stricter demonstrations at higher capability levels, pauses if safety procedures are not met, and board-approved procedural safeguards for policy changes [28]. **Redwood Research AI Control** studies protocols robust to intentional subversion by untrusted models, including detecting hidden backdoors in code [29]. Year96's evaluator must assume a challenger can try to pass evals while hiding bad behavior.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| DSPy | OSS | Typed LM programs, optimizers for prompts/weights, modular agent loops | MIT, verified by GitHub API/README | 38,399 stars; pushed 2026-09-28; Stanford NLP | Adopt |
| GEPA | OSS/paper | Trace-reading reflective evolution for prompts, configs, code, architecture text; Pareto frontier | MIT, verified by GitHub API/README | 6,781 stars; pushed 2026-09-28 | Adopt |
| OpenFeature spec | Standard/OSS | Provider-neutral feature flags for shadow/canary targeting | Apache-2.0, verified by GitHub API | 1,263 stars; pushed 2026-09-28 | Adopt |
| Argo CD | OSS | Declarative GitOps promotion and rollback on Kubernetes | Apache-2.0, verified by GitHub API | 24,265 stars; pushed 2026-09-28; Argo/CNCF | Adopt |
| Flux v2 | OSS | GitOps Toolkit, multi-tenant reconciler alternative/complement | Apache-2.0, verified by GitHub API | 8,429 stars; pushed 2026-09-25; CNCF graduated | Adopt |
| Prime Intellect verifiers | OSS | Common environment/evaluator abstraction for RL and evals | MIT, verified by GitHub API/README | 4,655 stars; pushed 2026-09-28 | Trial |
| OpenPipe ART | OSS/product | GRPO training for multi-step real-world agents | Apache-2.0, verified by GitHub API | 10,783 stars; pushed 2026-09-28 | Trial |
| Microsoft Agent Lightning | OSS | Train real agent harnesses via proxy/gateway/controller; K8s jobs | MIT, verified by GitHub API/README | 18,521 stars; pushed 2026-09-28; Microsoft | Trial |
| verl | OSS | Scalable RL post-training infrastructure | Apache-2.0, verified by GitHub API | 23,673 stars; pushed 2026-09-28; ByteDance Seed/community | Trial |
| OpenRLHF | OSS | Ray + vLLM RLHF/RLAIF/agent training at scale | Apache-2.0, verified by GitHub API | 10,047 stars; pushed 2026-09-17 | Trial |
| DGM | OSS/paper | Open-ended self-modifying code agent with archive and empirical benchmark gate | Apache-2.0, verified by GitHub API/README | 2,380 stars; pushed 2025-08-13; Sakana/UBC research | Trial |
| HGM | OSS/paper | Clade-level metaproductivity for self-improving coding agents | Apache-2.0, verified by GitHub API/README | 432 stars; pushed 2026-02-07; ICLR 2026 oral claim | Watch |
| AFlow | OSS/paper | MCTS search over code-represented agentic workflows | MIT, verified earlier by GitHub API; README opened | 605 stars in API check; pushed 2025-12-25 | Trial |
| AgentSquare | Paper | Modular agent search over planning/reasoning/tool/memory | N/A paper; repo not verified | arXiv 2024/2025 literature; primary repo unverified | Watch |
| OpenEvolve | OSS | Evolutionary code optimizer similar to AlphaEvolve | License badge present; SPDX not verified due rate limit | Active README; stars/commit unverified | Watch |
| AI Scientist | OSS/paper | Offline idea/experiment/review loop; code-execution warnings | License not verified in time box | Sakana repo; active lineage | Watch |
| AlphaEvolve | Product/research | Frontier example of evolutionary algorithm discovery | Proprietary/unknown | Google DeepMind-backed | Watch |
| Anthropic RSP | Policy | Safety levels, capability-triggered gates, procedural safeguards | N/A | Frontier-lab policy | Adopt concept |
| Redwood AI Control | Research | Protocols for safety despite intentional subversion | N/A | Research program | Adopt concept |
| Unverified leads: SICA, STOP, Gödel Agent, Agent0, MaAS, MASS, rLLM, SkyRL, Atropos | Mixed | Possible useful ideas but not verified | Unverified | Unverified | Watch |

## How I would build this part of Year96

### 1. Core architecture: Improvement Ownership + immutable kernel

Year96 should create a top-level **Improvement Ownership** in the Ownership layer (#07). It owns the long-term why: make the OS more capable and more tailored to each human without violating identity, permissions, audit, safety, or reversibility. Under it, create one Duty per mutation level:

- `L0PromptContextDuty`: prompt text, context packs, retrieval settings, memory summaries.
- `L1SkillPlaybookDuty`: skills, SOPs, checklists, AGENTS.md snippets, pstack/superpowers/skills playbooks (#11).
- `L2ToolConfigDuty`: tool schemas, model gateway config, timeouts, sandboxes, allowlists.
- `L3WorkflowGraphDuty`: agent graphs, communicator routing, executor decomposition, durable workflow definitions (#04/#05).
- `L4ModelTrainingDuty`: model routing, preference models, fine-tunes, RL/RFT jobs.
- `L5ComponentCodeDuty`: bounded code components, connectors, evaluators, simulators.
- `L6ArchitectureDuty`: interfaces, state model shifts, new layer boundaries, storage/routing architecture.

These Duties may propose and evaluate changes, but they do not own the root invariant set. That belongs to an **Immutable Improvement Kernel** maintained by identity-gates (#06), verification (#09), and human-approved governance. The kernel is implemented as small, heavily tested services and policy bundles: identity/authn/authz; audit appenders and tamper-evident ledger writers; sandbox policy; evaluation harness and benchmark registry; hidden-test secret manager; promotion/rollback controller; kill switch; and a kernel-change process requiring human approval. The kernel can read all system state but only exposes narrow provider interfaces to improvers.

### 2. Provider interfaces

```ts
type MutationLevel =
  | 'L0_PROMPT' | 'L1_SKILL' | 'L2_TOOL_CONFIG'
  | 'L3_WORKFLOW_GRAPH' | 'L4_MODEL_ROUTING_TRAINING'
  | 'L5_COMPONENT_CODE' | 'L6_ARCHITECTURE';

type VariantStatus =
  | 'DRAFT' | 'OFFLINE_PASSED' | 'SHADOW' | 'CANARY'
  | 'PROMOTED' | 'ROLLED_BACK' | 'REJECTED' | 'QUARANTINED';

interface ImprovementProposer {
  propose(input: {
    level: MutationLevel;
    objective: ImprovementObjective;
    evidence: TraceBundle;
    allowedSurfaces: MutableSurface[];
    constraints: InvariantRef[];
    archiveContext: VariantSummary[];
  }): Promise<VariantDraft[]>;
}

interface VariantArchive {
  put(draft: VariantDraft): Promise<VariantId>;
  get(id: VariantId): Promise<Variant>;
  lineage(id: VariantId): Promise<VariantLineage>;
  paretoFront(query: ArchiveQuery): Promise<VariantSummary[]>;
  quarantine(id: VariantId, reason: string): Promise<void>;
}

interface EvaluatorProvider {
  evaluate(input: {
    variant: VariantId;
    suites: BenchmarkSuiteRef[];
    replaySnapshots: StateSnapshotRef[];
    budget: CostLatencyBudget;
    verifierDiversity: VerifierRef[];
    hiddenEvalPolicy: HiddenEvalPolicyRef;
  }): Promise<EvaluationReport>;
}

interface BenchmarkSuiteProvider {
  resolve(level: MutationLevel, objective: ImprovementObjective, risk: RiskTier): Promise<BenchmarkSuiteRef[]>;
  reserveHiddenHoldout(scope: ScopeRef): Promise<HiddenEvalPolicyRef>;
}

interface RolloutProvider {
  shadow(input: RolloutRequest): Promise<RolloutRun>;
  canary(input: CanaryRequest): Promise<RolloutRun>;
  promote(input: PromotionRequest): Promise<DeploymentRef>;
}

interface PromotionPolicy {
  decide(input: {
    variant: VariantId;
    evals: EvaluationReport[];
    shadow: RolloutRun[];
    canary: RolloutRun[];
    invariants: InvariantCheckResult[];
    humanApprovals: Approval[];
  }): Promise<PromotionDecision>;
}

interface RollbackProvider {
  rollback(deployment: DeploymentRef, reason: RollbackReason): Promise<RollbackReport>;
  killSwitch(scope: RolloutScope, reason: string): Promise<void>;
}
```

### 3. Data model

```ts
interface ImprovementObjective {
  id: string;
  ownerIdentityId: string;
  threadId: string;
  humanScope?: HumanId;
  metricTargets: MetricTarget[];
  nonRegressionTargets: MetricTarget[];
  riskTier: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  why: string;
}

interface VariantDraft {
  parentVariantIds: string[];
  level: MutationLevel;
  patch: TextPatch | ConfigPatch | CodePatch | GraphPatch | ArchitectureProposal;
  rationale: string;
  expectedEffects: ScopeEffectPrediction[];
  requiredCapabilities: CapabilityRequest[];
}

interface EvaluationReport {
  id: string;
  variantId: string;
  suites: SuiteResult[];
  invariantResults: InvariantCheckResult[];
  cost: { usd: number; tokens: number; wallMs: number; gpuSeconds?: number };
  latency: LatencyStats;
  verifierDisagreements: VerifierDisagreement[];
  rewardHackingSignals: RewardHackingSignal[];
  tamperEvidenceHash: string;
  decisionRecommendation: 'PASS' | 'FAIL' | 'NEEDS_HUMAN';
}
```

All objects are stored in #01's state fabric as append-only events plus queryable projections. Every patch has a content hash; every evaluation references exact state snapshots, model versions, tool versions, prompts, seeds where possible, and cost/latency. This lets #09 replay failures and lets #02 learn which system changes create scope effects for which identities and threads.

### 4. The improvement loop

**Observe.** Consume traces from #09: task outcomes, verifier failures, human corrections, latency/cost anomalies, rollback incidents, model/tool errors, and communication friction from #04. Also consume #01 historical state snapshots so evaluation can replay the world as it was when a bug happened.

**Mine failures/opportunities.** Cluster symptoms: repeated prompt misunderstanding, missing skill, brittle tool schema, over-expensive model route, workflow deadlock, permission-denied pattern, or architecture bottleneck. Each cluster becomes an `ImprovementObjective` with scope, risk tier, metrics, and owner Duty.

**Propose variants.** For L0/L1, use DSPy/GEPA over traces. For L3, use AFlow/ADAS-style search in sandbox. For L4, create candidate routing policies or offline RL/fine-tune jobs through ART/Verifiers/Agent Lightning/verl. For L5, use OpenEvolve/DGM-like code search only on bounded components with deterministic tests. L6 proposals are design RFCs and simulator prototypes; they cannot auto-promote.

**Evaluate offline.** The EvaluatorProvider runs: unit/integration/e2e/mocked-e2e/agentic verifiers; replay against #01 snapshots; adversarial tests; permission invariant checks from #06; cost/latency budgets; and comparison against champion and baseline variants. Evaluation uses public/dev suites and hidden held-out suites. Hidden suite access is mediated by the kernel, and proposers receive only aggregate feedback to reduce overfitting.

**Shadow.** A passing variant sees copied production traffic or replayed live state but cannot act. Shadow compares decisions, tool calls, costs, memory writes, and would-have-escalated messages against the champion. Any permission drift, missing audit event, or unexplained high-impact action quarantines the variant.

**Canary.** OpenFeature targets a tiny cohort: internal test identities, low-risk threads, or users who opted into experimentation. For personalized variants, canaries are per human and per thread type. The RolloutManifest sets max blast radius, duration, rollback thresholds, and observability requirements.

**Promote or rollback.** Promotion requires invariant pass, metric improvement, no serious verifier disagreement, and cost/latency within budget. Rollback is automatic on hard invariant failure and manual/automatic on soft degradation. GitOps reconciles the chosen manifest to runtime via Argo/Flux; the improver never writes directly to production.

**Archive.** Rejected variants are still useful. The VariantArchive keeps Pareto-front stepping stones: high quality, low cost, high safety, high personalization fit, or interesting failure modes. DGM/HGM teach that a variant that loses today may be a parent of a winner tomorrow.

### 5. Invariants and safety

Invariants are executable, versioned, and separated from mutable behavior:

- **Identity invariant:** no variant may bypass identity, permission, capability attenuation, clone limits, or parent-thread communication rules (#06).
- **Audit invariant:** every proposed, evaluated, deployed, and rolled-back change emits signed append-only events.
- **Evaluation invariant:** proposers cannot edit benchmark results, hidden tests, judge prompts, verifier diversity settings, or tamper-evident log hashing.
- **Sandbox invariant:** generated code and agents run with least privilege, no unrestricted network, scoped filesystem, deterministic resource limits, and explicit egress.
- **Rollback invariant:** every promoted variant has a known previous champion and reversible manifest.
- **Human override invariant:** kill switch works even if agents disagree; critical L5/L6 and kernel changes require human approval.
- **Privacy invariant:** per-human personalization variants cannot leak preferences or memories across humans/organizations.

Anti-reward-hacking controls: rotating hidden evals; multiple verifier implementations and models; process-based checks (did it follow allowed route?) before outcome metrics; canary metrics that include complaints and overrides; adversarial variants trained to exploit the evaluator; and periodic "eval evals" where #09 tests whether the test suite detects known bad behaviors. Redwood-style AI-control assumptions apply: challengers may be intentionally subversive [29].

### 6. Personalization

Each human gets a **Preference Variant Profile**: communication style, risk appetite, preferred tools, cost/latency tradeoffs, memory boundaries, approval thresholds, and examples of good/bad outcomes. The system learns this through explicit feedback, inferred corrections, and long-running thread outcomes. Personalization is implemented as constrained deltas over shared variants:

- Global champion: safe default for everyone.
- Segment challenger: e.g., finance teams, developers, executives.
- Human challenger: Alice-specific prompt/context/skill/routing choices.
- Thread challenger: Alice's "legal contract review" preference differs from her "brainstorm game ideas" preference.

Promotion is multi-objective: a variant can promote for one human/thread while remaining rejected globally. The kernel enforces privacy and non-regression: personalization may change style and workflow choices but may not weaken permissions, audit, data retention, or safety gates.

### 7. Scaling path

**Laptop prototype:** SQLite/event log, local Docker sandbox, DSPy/GEPA for L0, simple replay traces, OpenFeature in-process provider, manual Git manifests.

**Team server:** Postgres/ClickHouse event projections, object storage for traces, Kubernetes namespaces for sandboxes, Prime Verifiers suites, GitOps to staging, shadow traffic from recorded traces, OpenTelemetry-style trace export (#09).

**Cluster:** multi-tenant state fabric (#01), durable workflow runtime (#05), GPU training pools for ART/Agent Lightning/verl, isolated evaluation clusters, feature flag service, Argo/Flux promotion, tamper-evident ledger, and policy-as-code integrated with #06.

### 8. Testing and proof plan: the 70% rule

For every ImprovementProvider implementation, write red tests first:

- Unit tests: manifest validation, patch application, invariant checks, rollout thresholds.
- Integration tests: proposer -> archive -> evaluator -> rollout using fake traces and fake flags.
- Mocked E2E: replay historical snapshots and prove a known bug is fixed without non-regression.
- E2E: sandboxed real agent run with shadow and canary simulation.
- Agentic verifiers: independent agents inspect whether the variant solved the why, not just the metric.
- Security/control tests: malicious variant tries to alter eval logs, read hidden tests, broaden permissions, disable rollback, or leak another human's preferences.
- Time/cost tests: every evaluation and rollout declares expected wall time/cost; slow loops become improvement objectives themselves.

No variant reaches canary unless the proof bundle is complete and queryable.

## What is still unsolved (late 2026)

- **Long-horizon evaluation.** DGM/HGM optimize coding benchmarks; Year96 needs to evaluate "better ownership" over weeks or years without waiting years for every promotion. Proxy metrics are needed but become reward-hack targets.
- **Architecture-level proof.** L6 changes alter assumptions of the evaluator itself. Immutable kernel plus human approval is necessary but not sufficient; Year96 needs simulation and formal interface contracts to compare architectures.
- **Verifier trust.** Agentic verifiers can be wrong, collude through shared model failures, or overfit to style. Diversity helps but does not solve epistemic uncertainty.
- **Hidden eval contamination.** A self-improving OS stores everything as state; hidden tests must be physically and logically separated from proposers while still producing useful feedback.
- **Personalization versus fairness/privacy.** Tailoring to a human can encode fragile, private, or harmful preferences. Need policy boundaries and explainable preference ledgers.
- **Open-ended archive bloat.** Keeping every stepping stone forever is searchable-state expensive. Need retention, summarization, and diversity-preserving pruning that does not discard future breakthroughs.
- **Reward hacking in RL agents.** RL frameworks are mature enough to use, but agent objectives are still easy to game. Year96 must prefer verifiable environment rewards and combine RL with invariant checks, not replace gates with scalar rewards.
- **Unverified 2026 ecosystem claims.** Several named leads could not be confirmed in time. A follow-up by #12 should verify SICA, STOP, Gödel Agent, Agent0, MaAS, MASS, rLLM, SkyRL, and Atropos before they influence build choices.

## Sources

1. `C:\projects\year96_!\year96\docs\research\00-TEAM_BRIEF.md`.
2. `C:\projects\year96_!\year96\docs\YEAR96_INTRO.md`.
3. `C:\projects\year96_!\year96\docs\YEAR96_SPEC.md`.
4. `C:\projects\year96_!\year96\docs\YEAR96_Vision.md`.
5. https://arxiv.org/abs/2505.22954
6. https://github.com/jennyzzt/dgm
7. https://github.com/metauto-ai/HGM
8. https://arxiv.org/abs/2408.08435
9. https://github.com/FoundationAgents/AFlow
10. https://arxiv.org/abs/2410.06153
11. https://arxiv.org/abs/2508.07407
12. https://arxiv.org/abs/2507.19457
13. https://github.com/stanfordnlp/dspy
14. https://github.com/gepa-ai/gepa
15. https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/
16. https://github.com/algorithmicsuperintelligence/openevolve
17. https://sakana.ai/ai-cuda-engineer/
18. https://github.com/SakanaAI/AI-Scientist
19. https://github.com/OpenPipe/ART and https://openpipe.ai/blog/art-trainer-a-new-rl-trainer-for-agents
20. https://www.primeintellect.ai/blog/environments
21. https://github.com/PrimeIntellect-ai/verifiers
22. https://github.com/microsoft/agent-lightning
23. https://github.com/volcengine/verl
24. https://github.com/OpenRLHF/OpenRLHF
25. https://github.com/open-feature/spec
26. https://github.com/argoproj/argo-cd
27. https://github.com/fluxcd/flux2
28. https://www.anthropic.com/news/anthropics-responsible-scaling-policy
29. https://www.redwoodresearch.org/research
30. https://github.com/geekan/MetaGPT
31. GitHub REST API repository metadata opened 2026-09-28 for: `jennyzzt/dgm`, `metauto-ai/HGM`, `stanfordnlp/dspy`, `gepa-ai/gepa`, `OpenPipe/ART`, `PrimeIntellect-ai/verifiers`, `volcengine/verl`, `OpenRLHF/OpenRLHF`, `microsoft/agent-lightning`, `open-feature/spec`, `argoproj/argo-cd`, `fluxcd/flux2`, and `FoundationAgents/AFlow`.
