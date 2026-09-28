# 09 — Verification, Proof-of-Done, Observability, and Time Measurement

Scope: this report designs the Year96 subsystem that spends the promised 70% on proof rather than activity. It covers Proof-of-Done (PoD) bundles, RED-first checks, verifier identities, test-level orchestration, deterministic and visual environments, OpenTelemetry-based evidence capture, time ledgers, provenance attestations, and non-code verification for tasks such as “Facebook ads launched” or “Unreal Engine build is playable.” It assumes #01 provides searchable state/snapshots, #05 provides durable runtime/timeouts/schedulers, #06 provides identity/permission gates, #07 provides sensors and human mental-model predicates, #08 consumes proofs for promotion/self-improvement, and #11 owns the pi.dev/pstack harness integration.

## TL;DR for the Year96 architect

- Build PoD as a blocking operating-system primitive, not a QA sidecar: every thread milestone, Builder action, and self-improvement promotion must emit a signed `ProofBundle` before any identity can claim “done.”
- The core invariant is **expected end state before execution**: a task starts only after an independent planner/verifier writes machine-checkable predicates, pre-state snapshot references, environment matrix, evidence plan, timeout budget, and RED-first failing checks.
- Adopt OpenTelemetry as the universal trace substrate: one trace per thread, span per agent step/tool/test/environment, GenAI/MCP semantic conventions where stable, and a Year96 evidence schema for screenshots, console dumps, logs, artifacts, sensor readings, and verifier verdicts.
- Adopt permissive core components: Inspect AI (MIT), promptfoo (MIT), DeepEval/Ragas/MLflow (Apache-2.0), Playwright (Apache-2.0; recheck Microsoft repo access before vendoring), in-toto (Apache-2.0), SLSA/Sigstore-style attestations, Pact/Stryker/fast-check/fuzzing for conventional proof depth.
- Keep non-permissive or source-available tools outside core: Arize Phoenix was checked as Elastic License 2.0, Grafana Loki is AGPL, Vector and Hypothesis are MPL; use as integrations only unless legal policy changes.
- Verifier independence is mandatory: builders can never mark their own work done; verifier identities must be separate, permission-limited, diverse in model/provider/prompt/tooling, calibrated against gold/human audits, and auditable by #06.
- LLM-as-judge alone is insufficient. Use verifier ensembles, rubric checklists, property tests, executable predicates, deterministic replay, mutation testing, and human spot-audits; LLM judges mostly explain and triage, while executable checks decide wherever possible.
- For code, the PoD gate should require level-appropriate unit, integration, mocked integration, E2E, mocked E2E, property/mutation/fuzz/contract tests, static/formal checks for high-risk modules, environment matrix runs, starting-state matrix runs, and timing regression checks.
- For non-code, represent “done” as predicates over external state: API reads, browser screenshots, third-party receipts, ledger entries, physical/sensor observations from #07, delayed outcome checks, and reopen timers if the proof depends on future state.
- Deterministic simulation is the template for agent OS reliability: every durable workflow and multi-agent protocol needs seedable simulations with injected time/network/tool/model failures, then replayable minimal failing traces.
- The DurationLedger is a first-class product signal: every command, model call, verifier run, environment boot, and human wait is timed, normalized by frequency and critical path, then fed to #08 to eliminate bottlenecks.
- Proof bundles are “proof-carrying task completion”: store evidence content-addressed, sign attestations, optionally log hashes to a transparency log, and make completion impossible without the bundle hash.

## Landscape

### Agent eval frameworks and benchmarks

**Inspect AI, UK AI Security Institute / Meridian Labs (2024–2026).** Inspect is a Python framework for frontier AI evaluations. Its docs describe Tasks composed of datasets, solvers, and scorers, with support for coding, agentic tasks, reasoning, multimodal understanding, tools, agents, over 20 model providers, and a catalogue of 200+ benchmark implementations [1]. GitHub LICENSE was checked directly: MIT. Why it matters: Inspect is the best candidate for Year96’s benchmark/eval authoring provider because it is public-sector influenced, provider-neutral, and naturally models solver/scorer separation.

**OpenAI Evals (2023–2026).** OpenAI’s repository describes a framework for evaluating LLMs or LLM systems, custom evals, and private evals; the page now points users to OpenAI Dashboard evals [2]. The root LICENSE was not accessible via GitHub MCP at the expected path, so license status is unverified here. Why it matters: useful as an external adapter and legacy registry model, but not a core dependency until license and activity are rechecked.

**promptfoo (active 2025–2026).** promptfoo’s docs call it an open-source CLI/library for evaluating and red-teaming LLM apps with declarative test cases, provider-agnostic APIs, CI/CD, local runs, red-team scans, caching, and matrix comparisons [3]. GitHub LICENSE checked: MIT. Why it matters: its YAML/CLI style is ideal for RED-first prompt/task regression files committed next to code and pstack tasks.

**DeepEval / Confident AI (active 2026).** DeepEval docs list end-to-end, trajectory, component-level, MCP, agentic metrics (task completion, tool correctness, plan adherence, step efficiency), CI/CD unit testing, and local backend storage [4]. GitHub LICENSE checked: Apache-2.0. Why it matters: useful for agent trajectory metrics and a pytest-like developer workflow.

**Ragas (active 2025).** Ragas positions itself as moving AI apps from “vibe checks” to systematic evaluation loops, with experiments, metrics, datasets, and integration with LangChain/LlamaIndex [5]. GitHub LICENSE checked: Apache-2.0, correcting the lead memory that it was MIT. Why it matters: RAG-specific faithfulness/context metrics are essential for #01/#03 state retrieval and thread-memory quality.

**MLflow 3 GenAI (2025–2026).** MLflow docs describe an open-source AI engineering platform for agents and LLMs with tracing, evaluation, prompt management, monitoring, AI gateway, 100+ integrations, OpenTelemetry compatibility, and 26K+ GitHub stars / 30M+ monthly downloads [6]. GitHub LICENSE checked: Apache-2.0. Why it matters: strong candidate for experiment tracking and eval/trace correlation if Year96 wants one permissive “boring enterprise” backbone.

**W&B Weave, Braintrust, LangSmith (2025–2026 products).** Weave docs describe agent/LLM observability and evaluation with OTel-compatible SDKs, traces for sessions/turns/LLM/tool calls, and LLM judges [7]. Braintrust and LangSmith are mature hosted eval platforms, but their hosted/commercial terms make them integrations rather than core. Why it matters: Year96 should expose adapters, not depend on them.

**Regression-suite benchmarks (2024–2026).** SWE-bench Verified evaluates generated patches by applying them and running FAIL_TO_PASS and PASS_TO_PASS tests, and labels test quality to reduce false negatives [8]. Terminal-Bench 4.0 is a terminal-agent benchmark “to measure and evolve with the frontier of agent work” [9]. tau2-bench evaluates conversational agents in a dual-control environment [10]. GAIA targets general AI assistants [11]. OSWorld supplies real OS tasks with initial-state setup and execution-based evaluation scripts; its original paper/site reported humans 72.36% vs best model 12.24% [12]. WebArena provides realistic web environments for autonomous agents [13]. METR’s time-horizon work measures the human-time length of tasks models can complete and keeps methodology/results current [14]. Why it matters: Year96 should not treat these as product gates, but as calibration suites for verifier identities, harness regressions, and #08 promotion.

### Verifiers and judge reliability

**Agent-as-a-Judge / LLM-as-Judge (2025–2026).** Recent papers survey the move from single LLM judges to tool-using, multi-step Agent-as-a-Judge evaluation [15][16]. EMNLP 2025 work frames LLM-as-judge opportunities and challenges [17]. A 2026 ACL survey covers Process Reward Models as process-level rather than only outcome-level supervision [18]. Why it matters: Year96 verifiers should inspect traces and intermediate states, not just final artifacts. However, LLM judges are biased, prompt-sensitive, and often overconfident; they require calibration, diversity, adversarial examples, and executable ground truth.

**Verifier ensembles and rubrics.** The state of the art is converging on multiple independent judges, prompt variants, rubric dimensions, tool-use verification, and calibrated aggregation. For Year96 this becomes a policy: no single LLM verdict can close a thread. A completion requires an ensemble quorum plus executable predicates; disagreement triggers escalation or a delayed/human audit.

### Formal and semi-formal methods

**Lean 4, Dafny, Verus, TLA+/P.** These remain high-value but selective tools. Use Lean for mathematical/protocol proofs; Dafny/Verus for verified critical code; TLA+ or P for distributed protocols, identity gates, workflow state machines, and exactly-once/durable scheduling invariants. LLM provers (AlphaProof, DeepSeek-Prover, Aristotle-like systems) are useful assistants, not trusted roots. Why it matters: Year96 should reserve formal methods for small, critical kernels: permission attenuation (#06), Durable Workflow semantics (#05), ProofBundle state transitions, and verifier quorum rules.

**Design-by-contract and runtime verification.** The practical default is executable contracts: preconditions, postconditions, invariants, temporal monitors, and policy-as-code. For non-code tasks, “contract” means predicates over external systems and sensors.

### Conventional QA techniques Year96 must make automatic

**Property-based testing.** fast-check (MIT; GitHub owner is `dubzzz/fast-check`, not `fast-check/fast-check`), proptest, and Hypothesis (MPL-2.0, therefore Excluded-license for core under this brief) generate many cases and shrink failures. Year96 should let LLMs propose properties, but never trust unreviewed properties; mutation testing should test test quality.

**Mutation testing.** StrykerJS is MIT (checked) and PIT is Apache-2.0 for JVM. Mutation score is the best practical “are tests meaningful?” gate. Year96 should not require 100% globally, but should require changed-risk-adjusted thresholds and explain killed/surviving mutants in the ProofBundle.

**Fuzzing.** AFL++ and OSS-Fuzz are established fuzzing ecosystems. Google has published AI-assisted fuzzing work, but Year96’s core design should be language-agnostic: generate harnesses, corpus seeds, invariants, and triage crashes into replayable bundles.

**Contract testing.** Pact JS is MIT (checked) and fits provider/consumer boundaries among Year96 interfaces. Every Provider interface below should have consumer-driven contract tests.

**Chaos and deterministic simulation.** Chaos Mesh and LitmusChaos are CNCF-adjacent Kubernetes chaos frameworks, but ordinary chaos is hard to reproduce. Deterministic simulation is stronger for core Year96 protocols. Antithesis describes itself as a deterministic simulation environment that exposes unlikely distributed bugs and gives perfect repro [19]. TigerBeetle’s 2026 protocol-aware DST post emphasizes safety/liveness invariants, strict serializability, and testing individual replicas, beyond black-box testing [20]. Year96 should make DST the default for multi-agent runtime semantics.

**Record/replay and session capture.** rr, Replay.io, rrweb, asciinema, and VHS support replayable human/terminal/browser evidence. Browser and terminal session recordings become first-class artifacts in ProofBundles, not debugging afterthoughts.

### E2E, visual, mobile, and game deliverables

**Playwright and Playwright Test Agents (2025).** Playwright docs now ship planner/generator/healer agents: planner explores and writes Markdown plans, generator turns plans into tests, healer executes and repairs failing tests; VS Code 1.105 (Oct 2025) is referenced for the agentic experience [21]. Why it matters: Year96 should adapt this pattern directly: independent plan, generated test, healer only suggests fixes, verifier signs result.

**Stagehand and browser-use.** Stagehand describes Playwright-style APIs plus self-healing actions, agent-optimized page context, complex DOM support, determinism, reliability, and observability [22]. browser-use provides cloud/open-source browser agent integrations with operational details such as concurrency and rate limits [23]. Use them as BrowserAction/TestRunner providers, not proof authorities.

**Visual regression.** Storybook/Chromatic, Lost Pixel, Argos, pixelmatch, odiff, and Playwright screenshots are the practical stack. Licenses vary; core should use permissive pixelmatch/odiff/Playwright and treat hosted visual SaaS as external.

**Mobile and games.** Maestro is mobile E2E. Unreal Automation Framework and Gauntlet are the right sources of truth for “Unreal game build playable”: boot packaged build, run smoke map, input simulation, FPS/crash logs, screenshot/video capture, and a deterministic seed where possible.

### Observability, telemetry, and provenance

**OpenTelemetry and GenAI semantic conventions.** The dedicated GenAI semantic-conventions repo covers spans, metrics, and events for GenAI clients, MCP, and provider-specific conventions [24]. MLflow, Langfuse, and Weave all advertise OTel-compatible tracing [6][7][25]. Why it matters: Year96 should not invent tracing. It should extend OTel with a stable Year96 evidence model and keep GenAI fields versioned because they are still moving.

**Langfuse.** Langfuse docs describe an open-source, self-hostable AI engineering platform with traces for LLM/non-LLM calls, sessions, user tracking, agent graphs, dashboards, alerts, SDKs, 100+ integrations, and OTel basis [25]. GitHub LICENSE checked: core MIT with EE directories. Trial as a self-hosted UI, but do not let its schema become the source of truth.

**Arize Phoenix.** Phoenix is useful for LLM observability/eval, but GitHub LICENSE checked as Elastic License 2.0, so it is Excluded-license for core under the brief. Use only externally.

**ClickHouse/HyperDX, SigNoz, Jaeger, Prometheus, Grafana/Loki, Vector.** Jaeger/Prometheus/OTel Collector are permissive core-friendly. Loki is AGPL and Vector is MPL, so mark as external or replace with permissive backends. ClickHouse is Apache-2.0 server-side but check each distribution.

**Provenance.** in-toto says it records what steps were performed, by whom, and in what order [26]; GitHub license checked Apache-2.0. SLSA v1.1 defines supply-chain levels and provenance/verification guidance [27]. Sigstore Cosign supports keyless signing and `cosign attest` for arbitrary JSON predicates with verification policies [28]. C2PA defines provenance standards for media content [29]. Why it matters: ProofBundleStore should emit in-toto/SLSA-like attestations for software artifacts and C2PA-like manifests for media/non-code outputs.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| OpenTelemetry + Collector | OSS/standard | Trace/log/metric backbone; thread/step/tool spans | Apache-2.0, checked for collector | CNCF; active; stars not rechecked due API rate limit | Adopt |
| OTel GenAI semantic conventions | Standard/OSS | GenAI/MCP span attributes for model/tool/agent calls | Apache-2.0 implied by OTel repo (not LICENSE-checked here) | Dedicated active repo; schema still evolving | Adopt with version pin |
| Inspect AI | OSS | Frontier eval authoring: dataset/solver/scorer/tasks | MIT, GitHub LICENSE checked | UK AISI/Meridian; active docs; stars not rechecked | Adopt |
| promptfoo | OSS | Declarative LLM evals/red-team, CI, local matrix tests | MIT, GitHub LICENSE checked | Active product/docs; stars not rechecked | Adopt |
| DeepEval | OSS | Agent trajectory metrics, MCP/tool correctness, CI tests | Apache-2.0, GitHub LICENSE checked | Active 2026 docs; stars not rechecked | Trial |
| Ragas | OSS | RAG metrics for #01/#03 retrieval quality | Apache-2.0, GitHub LICENSE checked | Active docs dated 12/2025; stars not rechecked | Adopt for RAG |
| MLflow 3 GenAI | OSS/product | Eval/tracing/prompt registry, enterprise tracking | Apache-2.0, GitHub LICENSE checked | 26K+ stars/30M downloads claimed by docs | Trial as eval registry |
| W&B Weave | Product/SDK | Hosted/self SDK tracing/evals, OTel agent traces | License not checked; hosted terms | Mature W&B-backed | External integration |
| Langfuse | OSS/open-core | Self-hostable LLM observability, sessions, agent graphs | MIT core + EE dirs, GitHub LICENSE checked | Active docs; ClickHouse-backed | Trial UI; not source of truth |
| Arize Phoenix | OSS/open-core | LLM observability/evals | ELv2, GitHub LICENSE checked | Mature Arize-backed | Excluded-license (core) |
| Playwright | OSS | Browser E2E, screenshots, traces, test agents | Apache-2.0 (not MCP-checked due Microsoft SAML; verify before adoption) | Mature Microsoft-backed; official 2025 agents | Adopt after license recheck |
| Stagehand | OSS/product | Agent browser actions with Playwright-like API and self-healing | MIT, GitHub LICENSE checked | Active Browserbase-backed | Trial |
| browser-use | OSS/product | Browser agent/cloud execution | MIT, GitHub LICENSE checked | Active docs; cloud API V4 | Trial/external |
| Pact JS | OSS | Consumer-driven contract tests for providers | MIT, GitHub LICENSE checked | Mature | Adopt |
| StrykerJS / PIT | OSS | Mutation testing/test adequacy | MIT for StrykerJS, checked; PIT Apache-2.0 (unverified here) | Mature | Adopt selectively |
| fast-check | OSS | TypeScript property-based testing | MIT (repo owner corrected; not MCP-checked in this run) | Mature | Adopt after recheck |
| Hypothesis | OSS | Python property-based testing | MPL-2.0, GitHub LICENSE checked | Mature | Excluded-license (core); external only |
| AFL++ / OSS-Fuzz | OSS/service | Fuzzing harnesses/corpus/crash repro | Apache-2.0 for OSS-Fuzz (unverified here); AFL++ mixed (unverified) | Mature | Trial/adopt per license |
| Antithesis | Product | Deterministic simulation/replay for distributed systems | Proprietary/product | Mature enterprise | External integration |
| TigerBeetle VOPR ideas | OSS/design | Protocol-aware deterministic simulation pattern | Apache-2.0 likely; not rechecked here | Active DB project | Adopt pattern, not dependency |
| in-toto | OSS/standard | Signed step attestations, layout/link model | Apache-2.0, GitHub LICENSE checked | CNCF graduated ecosystem | Adopt |
| SLSA | Standard | Supply-chain provenance levels and verification | CC/Apache ecosystem (unverified repo license here) | Industry standard | Adopt standard |
| Sigstore/cosign | OSS | Keyless signing, attestation, transparency logs | Apache-2.0 likely; not rechecked here | CNCF mature | Adopt after recheck |
| C2PA | Standard | Media/content provenance | Standard terms, not OSS lib | JDF/industry | Trial for media artifacts |
| Grafana Loki | OSS | Log backend | AGPL-3.0 (known; not checked here) | Mature | Excluded-license (core) |
| Vector | OSS | Telemetry pipeline | MPL-2.0 (known; not checked here) | Mature | Excluded-license (core) |

## How I would build this part of Year96

### 1) Architecture headline: Proof-of-Done as an OS gate

Year96 should make completion a state transition controlled by a Proof Gate. A Builder may emit `work_finished`, but only the Verification Ownership can emit `done_attested`. The Communication Hub (#04) and Identity Gates (#06) reject “done” messages, milestone closure, deployment promotion, payment triggers, or self-improvement promotion (#08) unless the referenced task has a valid `ProofBundle` hash whose policy evaluates to pass.

The PoD subsystem has four layers:

1. **Specification layer**: captures why, risk, expected end state, predicates, environment matrix, verifier independence, timing budget, evidence plan, and delayed checks before execution.
2. **Execution evidence layer**: wraps every command/tool/model/browser/session in timeout + OTel span + artifact capture + DurationLedger entry.
3. **Verification layer**: runs level-specific tests and independent verifiers across environments/starting states, requiring RED-first evidence where applicable.
4. **Attestation layer**: signs bundle, stores content-addressed artifacts, emits in-toto/SLSA/C2PA-style claims, and logs hash to a transparency/audit store.

### 2) Provider interfaces

```ts
type TaskRef = { orgId: string; threadId: string; taskId: string; parentTaskId?: string };
type ArtifactRef = { uri: string; sha256: string; mediaType: string; createdAt: string; retention: 'short'|'standard'|'legalHold' };
type PredicateKind = 'stateQuery'|'apiRead'|'browserAssertion'|'unitTest'|'metricThreshold'|'sensorReading'|'humanAudit'|'temporal';
type Verdict = 'pass'|'fail'|'inconclusive'|'waived';

interface ProofBundleStore {
  createDraft(ref: TaskRef, spec: ExpectedEndStateSpec): Promise<ProofBundleId>;
  appendEvidence(id: ProofBundleId, evidence: EvidenceRecord): Promise<void>;
  finalize(id: ProofBundleId, attestation: SignedAttestation): Promise<ProofBundle>;
  get(id: ProofBundleId): Promise<ProofBundle>;
  query(q: ProofQuery): AsyncIterable<ProofBundleSummary>;
}

interface VerifierProvider {
  identity(): VerifierIdentity;
  canVerify(spec: ExpectedEndStateSpec): Promise<boolean>;
  verify(input: VerificationInput): Promise<VerifierVerdict>;
  calibrationProfile(): Promise<CalibrationProfile>;
}

interface TestRunnerProvider {
  level: 'unit'|'integration'|'mockIntegration'|'e2e'|'mockE2e'|'property'|'mutation'|'fuzz'|'contract'|'formal'|'visual'|'mobile'|'game'|'agentic';
  plan(spec: ExpectedEndStateSpec): Promise<TestPlan>;
  run(plan: TestPlan, env: EnvironmentRef, startState: SnapshotRef, budget: TimeBudget): Promise<TestRunResult>;
}

interface EnvironmentMatrixProvider {
  selectMatrix(spec: ExpectedEndStateSpec, risk: RiskProfile): Promise<EnvironmentRef[]>;
  provision(env: EnvironmentRef, start: SnapshotRef): Promise<RunningEnvironment>;
  destroy(env: RunningEnvironment): Promise<void>;
}

interface StateSnapshotProvider {
  capture(scope: ScopeDescriptor): Promise<SnapshotRef>;
  diff(before: SnapshotRef, after: SnapshotRef): Promise<StateDiff>;
  query(snapshot: SnapshotRef, predicate: StatePredicate): Promise<PredicateResult>;
}

interface ArtifactCaptureProvider {
  before(step: StepRef, capturePlan: CapturePlan): Promise<ArtifactRef[]>;
  during(step: StepRef, event: CaptureEvent): Promise<ArtifactRef[]>;
  after(step: StepRef, capturePlan: CapturePlan): Promise<ArtifactRef[]>;
}

interface AttestationProvider {
  sign(bundle: ProofBundle, signer: IdentityRef): Promise<SignedAttestation>;
  verify(attestation: SignedAttestation): Promise<AttestationVerification>;
  publishTransparencyHash?(attestation: SignedAttestation): Promise<TransparencyRef>;
}

interface TelemetryProvider {
  startThreadTrace(ref: TaskRef, attrs: Record<string, unknown>): Promise<TraceRef>;
  span<T>(trace: TraceRef, name: string, attrs: Record<string, unknown>, fn: () => Promise<T>): Promise<T>;
  exportEvidence(trace: TraceRef): Promise<ArtifactRef[]>;
}

interface DurationLedger {
  start(op: OperationRef, budget: TimeBudget): Promise<TimerRef>;
  stop(timer: TimerRef, outcome: Verdict): Promise<DurationRecord>;
  regressions(scope: ScopeDescriptor): Promise<DurationRegression[]>;
}
```

Every provider is swappable. Local laptop mode writes bundles/artifacts to a filesystem + SQLite/Postgres + local OTel collector. Cluster mode writes artifacts to object storage, events to Kafka/Pulsar, traces to OTel Collector, metrics to Prometheus-compatible storage, proofs to an append-only database, and attestations to Sigstore/Rekor-like transparency logs.

### 3) Data model: ProofBundle schema

```ts
type ProofBundle = {
  schemaVersion: 'year96.pod.v1';
  bundleId: string;
  task: TaskRef;
  why: { userIntent: string; ownershipReason: string; scopeEffectRefs: string[] };
  createdBeforeExecutionAt: string;
  preState: { snapshot: SnapshotRef; scope: ScopeDescriptor; summary: string };
  expectedEndState: ExpectedEndStateSpec;
  redFirst: { required: boolean; failingEvidence?: EvidenceRecord; waiver?: WaiverRecord };
  plan: { steps: PlannedStep[]; risks: RiskProfile; timeoutBudget: TimeBudget; environmentMatrix: EnvironmentRef[]; startingStates: SnapshotRef[] };
  executionTrace: { traceId: string; spansArtifact: ArtifactRef; logs: ArtifactRef[]; console: ArtifactRef[]; screenshots: ArtifactRef[]; recordings: ArtifactRef[] };
  results: Record<TestRunnerProvider['level'], TestRunResult[]>;
  verifierVerdicts: VerifierVerdict[];
  timing: { total: DurationRecord; byStep: DurationRecord[]; regressions: DurationRegression[] };
  environmentManifest: { images: ImageRef[]; lockfiles: ArtifactRef[]; seeds: string[]; toolVersions: Record<string,string>; modelVersions: Record<string,string> };
  postState: { snapshot: SnapshotRef; diff: StateDiff; predicateResults: PredicateResult[] };
  delayedChecks: DelayedVerification[];
  exceptions: WaiverRecord[];
  attestation: SignedAttestation;
};
```

The `ExpectedEndStateSpec` is the core. It must be committed before execution and is immutable except by a new version that invalidates prior RED-first evidence. It contains typed predicates such as: database row exists; API says campaign active; page has visible order confirmation; Unreal build boots map and remains playable for N seconds; metric error rate below threshold; no unauthorized state diff outside declared scope; delayed conversion or retention metrics checked at future time.

### 4) Typical flow

1. **Thread requests work.** #04 creates/updates a Thread and calls Verification Ownership for a proof spec template.
2. **Expected end state is authored.** A planner identity plus verifier identity translate the human “why” into checkable predicates and risk. For “Facebook ads launched,” predicates include campaign ID exists, status active/not draft, budget matches, targeting matches, creative IDs approved, billing not blocked, screenshots/CSV/API receipt captured, and delayed check at +24h that impressions > 0 or platform explains review delay.
3. **Pre-state is captured.** #01 snapshots relevant repo, browser/app state, external API reads, permissions, environment manifest, and any #07 sensor baselines. The scope-effect engine (#02) supplies likely affected state to observe.
4. **RED-first is recorded.** For a bug fix, at least one failing unit/E2E/property/monitor check is captured before the fix. For a new feature or non-code task, a precondition failure demonstrates “not done yet” (campaign absent, game build fails to boot, visual element missing).
5. **Builder executes under wrappers.** #05 enforces timeout and 15-minute clock checks. Every command/tool/model call is a child OTel span, with console/log/screenshot/session capture. Builders can add evidence but cannot set verdicts.
6. **Test matrix runs.** TestRunnerProviders execute against selected environments and starting states: local mocks, integration services, browser/device matrix, deterministic simulation seeds, and production-like canary where safe.
7. **Independent verifiers judge.** At least two independent verifier identities check the bundle. For high risk, require diversity across model families/providers and one non-LLM executable verifier. Verifier outputs include rubric item scores, evidence references, confidence, calibration profile, and dissent.
8. **Proof gate evaluates policy.** Policy checks no self-verification, required levels pass, RED-first exists/waived, artifact hashes valid, no forbidden state diff, duration regressions below threshold or flagged, and attestation signature valid.
9. **Completion or reopen.** If all pass, bundle is finalized, signed, and thread milestone moves to `done_attested`. If delayed predicates exist, the thread remains in `done_pending_temporal_verification`; failed delayed checks reopen the thread automatically.

### 5) Non-code deliverables

Non-code tasks need the same rigor but different sensors:

- **External SaaS tasks:** use official APIs where possible, then browser screenshots as secondary evidence. Example: Facebook ads launched = API campaign/adset/ad records + billing/review status + screenshot + immutable timestamp + delayed delivery check.
- **Content/media tasks:** C2PA-style manifests, source asset hashes, rendered output hash, visual/audio quality checks, policy checks, human spot audit, publication URL proof.
- **Physical/IoT tasks:** #07 sensors provide signed readings: camera frame, GPS, scale, temperature, machine telemetry. Predicates include calibration metadata and confidence.
- **Games/creative software:** Unreal Gauntlet/Automation runs, crash logs, FPS histogram, controller input script, screenshot/video recording, packaged artifact hash, target hardware matrix.
- **Long-outcome tasks:** sales, ad performance, hiring, support resolution, and SEO may not be knowable immediately. The bundle records immediate done predicates plus delayed outcome predicates that schedule future verifiers and can reopen the thread.

### 6) Organizing the 70/30

Create a **Verification Ownership** equal in status to product/runtime ownership. Its Duties:

- **Proof schema and gate duty:** owns ProofBundle schema, policies, migrations, and compatibility.
- **Test-framework builders:** build shared runners, fixtures, property generators, mocks, simulators, visual baselines, game/mobile harnesses.
- **Verifier pool:** maintains independent verifier identities, model diversity, calibration sets, red-team prompts, and human audit sampling.
- **Environment matrix duty:** curates containers, devcontainers/Nix/Bazel/Dagger images, seeded databases, browser/device/game hardware pools.
- **Starting-state generator:** works with #01 to generate representative snapshots: empty/new user, dirty/migrated user, large org, permission edge case, flaky network, long-dormant thread.
- **Observability duty:** owns OTel schemas, capture policy, retention/redaction, dashboards/query APIs for #01.
- **Timing duty:** owns DurationLedger, bottleneck alerts, and “rare but slow vs frequent and slow” economics.
- **Red team duty:** creates adversarial tasks where builders can fake done, verifiers can be fooled, or external systems drift.

70/30 means product teams spend less time writing bespoke tests because Verification Ownership supplies reusable harnesses. It does not mean every low-risk typo fix runs a week-long cluster suite; risk policy chooses minimal sufficient proof while preserving the invariant that all applicable levels are considered and explicitly passed/waived.

### 7) Scaling path

- **Laptop:** local OTel collector, SQLite proof index, filesystem artifact store, Playwright traces, promptfoo/Inspect local evals, Docker/devcontainer snapshots.
- **Team server:** Postgres proof store, object storage artifacts, CI runners, environment pools, Langfuse/MLflow UI, signed in-toto attestations.
- **Cluster:** Kubernetes jobs, ephemeral namespaces, Bazel/Dagger hermetic builds, deterministic simulation farm, fuzzing fleet, model-gateway quotas, centralized OTel collector, Prometheus/Jaeger-compatible stores, policy engine, transparency log.
- **Regulated/enterprise:** hardware-backed signing, legal hold retention, human audit queues, data residency partitions, SLSA provenance, tamper-evident logs, per-customer proof export.

### 8) How this subsystem proves itself

The PoD subsystem must eat its own dogfood. Its own ProofBundles require: schema property tests; contract tests for every provider; mutation testing of gate policies; deterministic simulations of verifier disagreement, missing artifacts, forged signatures, timeouts, and delayed-check reopen; E2E tests that a Builder cannot self-close; visual/browser tests for proof UI; load tests for artifact retention; formal specs for state transitions; and red-team attempts to mark done with forged or incomplete evidence.

## What is still unsolved (late 2026)

- **Semantic “100%” is not fully formalizable.** Many human desires are underspecified. Year96 can force expected predicates up front, but deciding whether predicates capture the real “why” remains an Ownership/#07 mental-model problem.
- **LLM verifier calibration remains fragile.** Agent-as-a-Judge improves over single judges, but prompt sensitivity, model collusion, hidden shared training data, and overconfidence persist. Human audits and executable checks remain necessary.
- **External systems are adversarially incomplete.** SaaS APIs may hide review state, rate-limit, change UI, or forbid scraping. Proof must combine API, screenshot, receipts, and delayed outcome checks, but certainty is bounded.
- **Cost explosion.** Running every level across every environment/starting state is impossible. Year96 needs risk-based test selection, test-impact analysis, caching, and proof reuse without creating loopholes.
- **Privacy vs evidence.** Screenshots, console logs, prompts, and traces can contain secrets or personal data. The hard problem is retaining enough evidence to reproduce while redacting/minimizing under policy.
- **Flakiness vs real nondeterminism.** Agents, browsers, SaaS, and models are non-deterministic. Deterministic wrappers help, but model/provider changes and external review queues require probabilistic gates and delayed checks.
- **Physical-world proof is expensive.** Sensor calibration, spoofing resistance, chain-of-custody, and human-in-the-loop audits must be solved with #07 and #06.
- **Formal methods do not scale automatically.** LLM provers help author specs/proofs, but trusted formal verification remains limited to small kernels. The innovation is choosing the right kernels.
- **Verifier independence can be illusory.** Different agents may share prompts, models, data, or organizational incentives. Year96 needs identity-level separation, audit, model diversity, and random human spot checks.
- **Proof UX can become bureaucracy.** If PoD is slow or noisy, builders will route around it. The system must make the correct proof path the fastest path by generating specs, tests, captures, and attestations automatically.

## Sources

1. https://inspect.aisi.org.uk/
2. https://github.com/openai/evals
3. https://www.promptfoo.dev/docs/intro/
4. https://deepeval.com/
5. https://docs.ragas.io/en/stable/
6. https://mlflow.org/docs/latest/genai/
7. https://docs.wandb.ai/weave
8. https://openai.com/index/introducing-swe-bench-verified/
9. https://terminal-bench.com/
10. https://arxiv.org/abs/2506.07982
11. https://arxiv.org/abs/2311.12983
12. https://os-world.github.io/
13. https://webarena.dev/
14. https://www.metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/
15. https://arxiv.org/abs/2508.02994
16. https://arxiv.org/abs/2601.05111
17. https://aclanthology.org/2025.emnlp-main.138/
18. https://aclanthology.org/2026.acl-long.163/
19. https://antithesis.com/docs/
20. https://tigerbeetle.com/blog/2026-08-20-protocol-aware-dst/
21. https://playwright.dev/docs/test-agents
22. https://www.stagehand.dev/
23. https://docs.browser-use.com/
24. https://github.com/open-telemetry/semantic-conventions-genai
25. https://langfuse.com/docs
26. https://in-toto.io/
27. https://slsa.dev/spec/v1.1/
28. https://docs.sigstore.dev/cosign/signing/signing_with_containers/
29. https://c2pa.org/specifications/specifications/2.2/index.html
