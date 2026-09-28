# 05 — Durable runtime, sandboxes, and time-aware execution

Scope: this report owns the Year96 execution substrate: durable identity/thread lifecycles, Builder workflows, scheduling, sandbox and environment leases, time awareness, model access, budget enforcement, and deployment topology. It assumes #01 supplies the state fabric, #02 supplies scope-effect signals, #03 owns thread memory, #04 owns communication, #06 owns identity/permission gates, #09 owns verification, and #11 owns the pi.dev/hermes-agent harness details.

## TL;DR for the Year96 architect

- Model every **Identity** and **Thread** as a virtual actor: always exists by ID, activates on message/timer/scope-effect trigger, serializes local decisions, persists state into #01, and passivates when idle. Orleans, Dapr Actors, and Cloudflare Durable Objects validate the pattern.
- Model every **Builder** as a durable workflow, not as an actor. Steps are activities with start-to-close/schedule-to-close/heartbeat timeouts, retries, compensation, idempotency keys, and sandbox leases.
- **Adopt Temporal** for Builder runtime. Server and TypeScript SDK licenses are MIT (verified). Temporal is mature, has schedules/timers/signals/queries/task queues, and its 2026 Durable AI docs list TypeScript/Python integrations for OpenAI Agents SDK, Vercel AI SDK, Google ADK, LangGraph, Mastra, Pydantic AI, and Strands Agents [1][2][3].
- For actor hosting in a TypeScript-first system, **Adopt Dapr Actors** first for Kubernetes because it is Apache-2.0 and language-neutral; adopt the **Orleans virtual-actor model** conceptually; trial Orleans only if .NET silos are acceptable [9][10].
- **Cloudflare Durable Objects + Agents SDK** are the cleanest hosted embodiment of “logically alive, physically asleep”: globally named stateful actors, SQLite storage, hibernation, WebSocket hibernation, and alarms [6][7]. Trial as an external provider, not portable core.
- **Time awareness is infrastructure, not prompting**: every command/tool/model call carries a deadline; every agent session gets a 15-minute timer; every heartbeat records elapsed time, remaining deadline, sandbox lease expiry, model spend, and duration baseline percentile.
- Use the gRPC deadline model: callers always set realistic deadlines; downstream calls receive remaining timeouts after elapsed time is deducted, reducing clock-skew errors [8].
- Use **LiteLLM** for laptop MVP model gateway: virtual keys, model allowlists, spend tracking, max budgets, TPM/RPM limits, and user/team/key ledgers are documented [18]. Trial **Portkey** for fallback/cache/guardrails/budgets [19]. Adopt **Agent Router / Envoy AI Gateway** for Kubernetes-grade governed model egress [20].
- Use tiered sandboxes: Wasm for deterministic plugins, containers/gVisor for normal code, Kata/Firecracker microVMs for untrusted code, browser providers for web tasks, desktop VMs for computer use, and GPU/Windows/macOS build pools for Unreal/Unity/Xcode.
- **Do not place Restate, Golem, Inngest, Akka JVM, Daytona, or n8n in core** without legal exception. I verified BSL for Restate/Golem, SSPL for Inngest, and the assignment already flags Akka/n8n; such tools may remain external integrations.
- hermes-agent should be a **SchedulerProvider/AutomationProvider client** owned with #11, not the runtime. It registers repetitive monitored automations into the core runtime; the core owns actors, workflows, timers, leases, budgets, and deadlines.
- Biggest hard problem: dense wake storms. Millions of dormant actors are cheap; millions waking at once with isolated environments and model calls can overload queues and Kubernetes control planes. Google’s 2026 Agent Substrate exists specifically because normal K8s is not designed for millions of sub-second agent tool calls [13].

## Landscape

### Durable execution and workflow engines

**Temporal (MIT verified).** Temporal is the durable-execution baseline. Workflows persist event history and recover after crashes; Activities isolate retries and side effects; timers/signals/queries/child workflows/schedules cover long waits and human-in-the-loop. Temporal docs state workflows are designed to be long-running, default Workflow Execution Timeout is infinite, and timers are preferred to global caps; Workflow Task Timeout detects failed workers [1]. For Year96 this means a Builder can live for months while every command still has strict deadlines. Temporal’s Durable AI docs now explicitly target agents, pipelines, internal agent platforms, and model training, with cookbook recipes and TypeScript/Python integrations [2]. Its OpenAI Agents SDK integration became GA for Python in March 2026 and emphasizes resuming after rate limits, network failures, crashes, and bug fixes [3]. Adopt.

**DBOS (MIT verified for dbos-transact-ts).** DBOS is database-centric durable execution. Its OpenAI Agents docs replace `Runner.run` with `DBOSRunner.run`, annotate agent callers with `@DBOS.workflow`, annotate tools/guardrails with `@DBOS.step`, and recommend SQLite for quick start/Postgres for production [5]. Trial for laptop or Postgres-only deployments; Temporal remains stronger for heterogeneous workers and long operational history.

**Restate (BSL-1.1 verified).** Technically appealing durable services/sagas, but current license is Business Source License with restrictions on a public Restate platform service and Apache change after four years. Excluded-license (core).

**Hatchet (MIT verified) and Trigger.dev (Apache-2.0 verified).** Both are good developer-facing job/workflow platforms, especially for TypeScript ergonomics. Trial for application-level jobs; do not make them the universal runtime until multi-year workflow replay, schedules, and large-scale recovery are proven.

**Inngest (SSPL verified).** Inngest docs describe event-driven durable functions, TS/Python/Go, steps, queueing, scaling, concurrency, throttling, rate limits, observability, crons, AI agent resources, and MCP [22]. License excludes it from core.

**Cadence.** Still a valuable predecessor, but Temporal has the current ecosystem and agent integrations.

**LangGraph persistence/checkpointers.** Useful inside an agent harness for graph state, but not a substitute for timers, queueing, sandbox leases, retries, and budget enforcement. Wrap production graph runs in Temporal.

**Golem (BSL verified) and Resonate (Apache-2.0 verified).** Golem is excluded from core despite a strong durable-computing story. Resonate is permissive and emerging; Watch/Trial for durable promises.

### Actor runtimes and always-alive identities

**Orleans virtual actors (Apache-2.0 verified).** Microsoft’s Orleans page says an actor always exists virtually, cannot be explicitly created/destroyed, transcends in-memory instances, activates on message, reclaims idle instances, and re-instantiates after server crash [9]. That is almost a literal implementation of Year96 threads that never close. Adopt the model; runtime is .NET-first.

**Dapr Actors (Apache-2.0 verified).** Dapr implements the virtual actor pattern with language-neutral sidecars. Its docs define actors as one-at-a-time message processors with identity, state, on-demand activation, and suitability for thousands of independent units; Dapr Workflow builds on actors for orchestration [10]. Adopt for K8s MVP.

**Cloudflare Durable Objects and Agents (Agents repo MIT verified; platform proprietary).** Durable Objects are globally named stateful serverless objects with attached strongly consistent storage, SQLite GA, fast startup, idle shutdown, and hibernation [6]. Alarms wake an object in the future, guarantee at-least-once execution, retry with exponential backoff, and can multiplex many logical timers through one physical alarm [7]. Trial as `CloudflareActorHostProvider`.

**Akka/Ray/OTP.** Akka JVM is excluded due BSL concerns; Akka.NET is MIT but .NET-specific. Ray (Apache-2.0 verified) is better as GPU/ML compute than identity runtime. Erlang/Elixir OTP remains the best supervision-tree reference.

### Agent platforms and Kubernetes-native runtimes

AWS Bedrock AgentCore is positioned as a platform to build/connect/secure/debug/scale agents with any framework/model [23]. Google’s Agent Platform documents managed Agent Runtime, Sessions, Memory Bank, evaluation, tracing/logging/monitoring, code execution, and computer use [24]. Microsoft Foundry Agent Service documents runtime, toolboxes/MCP endpoint, model catalog, observability, optimizer, Entra/RBAC, and VNet isolation [25]. All validate the Year96 architecture but are external providers, not portable core.

**Kubernetes/GKE Agent Sandbox.** Google docs describe isolated, stateful, single-replica workloads for AI agent runtimes executing untrusted LLM-generated code, with gVisor/Kata, sub-second provisioning, Sandbox/SandboxClaim/SandboxTemplate/SandboxWarmPool CRDs, Pod snapshots, default-deny networking, and SDK access [14]. The 2026 GA blog reports 16x sandbox growth in under five months, 300 allocations/sec/cluster, 90% in 200ms, pod snapshots for idle agents, and Agent Substrate for ultra-scale agents [13]. Trial to Adopt after license verification.

**kagent.** kagent + Agent Substrate frames the Kubernetes agent problem around isolation, security, scale, policy enforcement, state snapshotting, and isolated multi-tenant routing [15]. Watch/Trial with #11.
### Sandboxes and execution environments

**Firecracker (Apache-2.0), gVisor (Apache-2.0/BSD mix), Kata Containers (Apache-2.0).** Licenses were verified directly. Firecracker gives microVM isolation, gVisor intercepts syscalls in a user-space kernel, and Kata provides VM-isolated containers. Year96 should choose among them by risk: normal trusted code in containers/gVisor, untrusted generated code in Kata/Firecracker, deterministic extensions in Wasm.

**E2B (Apache-2.0 verified for infra; product).** E2B docs define isolated sandboxes for agents with JS/Python SDKs, fast secure Linux VMs, templates, and persistence where pause saves filesystem and memory indefinitely until killed [21]. Excellent external burst provider.

**Cloudflare Sandbox SDK and Vercel Sandbox.** Cloudflare’s 2026 Sandbox SDK provides a TypeScript API for untrusted code in isolated containers with commands, files, background processes, services, Python/Node execution, and Workers integration [16]. Vercel Sandbox is GA for isolated Linux microVMs for agent workflows and one-off commands [17]. Both are external `EnvironmentProvider`s.

**Wasmtime and Extism.** Wasmtime’s Apache-2.0 WITH LLVM-exception and Extism’s BSD-3-Clause licenses were verified. Use them for low-latency capability-scoped plugin execution.

**container-use.** The opened GitHub page describes isolated coding-agent environments per git branch, real-time logs, direct intervention, and MCP compatibility [26]. Trial for laptop after license verification.

**Browser/desktop/game-engine environments.** Browserbase/Steel/Kernel/Playwright, Windows/macOS VMs, and GPU build farms must be separate providers. The Unreal example needs persistent GPU/Windows/Linux build leases, artifact caches, license-aware engine installs, and explicit lease expiry.

### Scheduling, automation, and triggers

Temporal Schedules should own durable starts and cron-like workflow triggers in core. Kubernetes CronJobs handle cluster maintenance. Airflow/Dagster/Prefect/Kestra are external data/workflow integrations. n8n is excluded from core because fair-code licensing conflicts with the brief. hermes-agent should register repetitive monitored processes into `SchedulerProvider`, then receive runtime events; it should not own actor state or Builder semantics.

### Model gateways, inference, budgets, and routing

**LiteLLM (MIT core verified).** LiteLLM virtual keys support model access, owner/team inheritance, MCP access, max budgets, TPM/RPM, and spend tracking per key/user/team; costs are calculated from model price tables [18]. Adopt for laptop and early cluster.

**Portkey Gateway (MIT verified).** Portkey docs list universal API, simple/semantic cache, MCP, fallbacks, conditional routing, retries, circuit breaker, load balancing, canary testing, request timeout, budget limits, rate limits, and custom hosts [19]. Trial/Adopt where its gateway features beat LiteLLM.

**Agent Router / Envoy AI Gateway (Apache-2.0 verified).** Now branded Agent Router, it is built on Envoy and targets LLM/AI traffic routing, failover, credentials isolation, quotas, usage attribution, MCP control, observability, and Kubernetes-native operation [20]. Adopt for production clusters once policies stabilize.

**agentgateway, kgateway, Gateway API inference extension, vLLM, SGLang.** Checked repos are permissive (Apache-2.0 for agentgateway/gateway-api inference/vLLM/SGLang). vLLM/SGLang should sit behind the model gateway for self-hosted inference.

### Time awareness, deadlines, and baselines

gRPC deadlines are the right standard: clients should always set deadlines because otherwise calls may wait forever; servers cancel expired work; downstream calls receive timeouts after elapsed time is deducted, protecting against clock skew [8]. Temporal provides workflow/activity/task/heartbeat timeouts; Durable Objects alarms and Temporal timers supply wakeups. Year96 adds a central duration ledger: operation kind, identity, thread, environment, model, deadline, elapsed time, status, retry count, heartbeat interval, baseline p50/p90/p99, rarity, and bottleneck flag. LLMs should not be trusted to remember dates: runtime injects signed `TimeContext` before every agent turn and at 15-minute intervals, and rejects tool calls without deadlines.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| Temporal | OSS | Durable Builder workflows, timers, retries, heartbeats, schedules, AI integrations | MIT verified | Mature; Temporal Technologies; repo metadata rate-limited but docs active 2026 | Adopt |
| Temporal TypeScript SDK | OSS | TypeScript workflows for pi.dev integration | MIT verified | Official mature SDK | Adopt |
| DBOS | OSS/product | Postgres-backed durable execution; OpenAI Agents wrapper | MIT verified | Active startup | Trial |
| Restate | Source-available | Durable services/sagas | BSL-1.1 verified | Active | Excluded-license |
| Hatchet | OSS | Friendly tasks/workflows | MIT verified | Active startup | Trial |
| Trigger.dev | OSS/product | TS background jobs and long tasks | Apache-2.0 verified | Active product | Trial |
| Inngest | Source-available | Event-driven durable steps, cron, throttling | SSPL-1.0 verified | Active product | Excluded-license |
| Orleans | OSS | Virtual actors exactly matching endless identities | Apache-2.0 verified | Mature Microsoft/.NET | Adopt model / Trial runtime |
| Dapr Actors | OSS/CNCF | Language-neutral virtual actors | Apache-2.0 verified | CNCF mature | Adopt |
| Cloudflare Durable Objects/Agents | Product + OSS SDK | Hosted hibernating actors, SQLite, alarms | Agents SDK MIT verified; platform proprietary | Cloudflare, docs updated 2026 | Trial |
| GKE/K8s Agent Sandbox | OSS/product | Secure stateful sandbox CRDs, warm pools, snapshots | License not directly fetched; Google says open-source | GA on GKE 2026 | Trial→Adopt |
| Firecracker | OSS | microVM isolation | Apache-2.0 verified | AWS-backed mature | Adopt |
| gVisor | OSS | user-space kernel sandbox | Apache-2.0/BSD verified | Google-backed mature | Adopt |
| Kata Containers | OSS | VM-isolated containers | Apache-2.0 verified | Mature | Adopt |
| E2B | Product/OSS | Agent Linux VMs, pause/resume persistence | Apache-2.0 verified for infra | Active product | Trial external |
| Cloudflare Sandbox SDK | Product | TS edge sandboxes over Containers | Proprietary service | Preview/1.0 docs 2026 | Trial external |
| Vercel Sandbox | Product | GA Linux microVM sandboxes | Proprietary service | Vercel GA 2026 | Trial external |
| Wasmtime | OSS | deterministic Wasm runtime | Apache-2.0 WITH LLVM-exception verified | Bytecode Alliance mature | Adopt |
| Extism | OSS | embeddable Wasm plugins | BSD-3-Clause verified | Active | Adopt |
| LiteLLM | OSS/product | Model proxy, virtual keys, spend, budgets | MIT verified; enterprise dir separate | Widely used | Adopt MVP |
| Portkey Gateway | OSS/product | Routing, fallback, cache, budgets/rate limits | MIT verified | Active; PRISMA AIRS docs | Trial/Adopt |
| Agent Router / Envoy AI Gateway | OSS | Kubernetes/Envoy LLM gateway, policy, quotas, MCP | Apache-2.0 verified | Envoy/Agentic AI Foundation | Adopt cluster |
| vLLM | OSS | high-throughput self-host inference | Apache-2.0 verified | Very active | Adopt |
| SGLang | OSS | structured high-performance serving | Apache-2.0 verified | Active | Trial |
| Golem | Source-available | durable computing runtime | BSL-1.1 verified | Active | Excluded-license |
| Resonate | OSS | emerging durable execution/promises | Apache-2.0 verified | Newer | Watch/Trial |
| container-use | OSS MCP tool | local isolated coding-agent envs/logs | unverified in this pass | Early Dagger project | Trial after license check |
| pi.dev | Harness | Required minimal extensible harness | unverified in this pass | Site active; TS-oriented docs | Adopt as mandated |
| hermes-agent | Harness automation | repetitive monitored automations | unverified in this pass | Needs #11 validation | Trial as Scheduler client |

## How I would build this part of Year96

### Runtime architecture

Year96 Runtime has five planes.

1. **Actor plane:** every `IdentityActor` and `ThreadActor` is addressable by stable ID. It receives messages, updates small local projections, appends state pointers, schedules next thoughts/check-ins, wakes listeners/hangers, and starts Builder workflows. It never stores bulky memory; #03 and #01 own durable memory and truth.
2. **Workflow plane:** every Builder run is a Temporal workflow. Steps are activities: model call, tool call, shell command, browser action, build, deploy, verify, human approval wait, compensation. Activities declare timeout, heartbeat, retry policy, budget class, required environment, idempotency key, and compensator.
3. **Environment plane:** leases sandboxes, browsers, desktops, and build machines. Workflows renew leases by heartbeat; idle environments snapshot/passivate or die by policy.
4. **Time plane:** authoritative clock, timers, reminders, deadline propagation, 15-minute session check-ins, duration ledger, baselines, anomaly/bottleneck flags.
5. **Model plane:** model gateway, budget provider, rate limits, routing, fallback, caches, token accounting, and policy gates from #06.

The Owner never executes. Ownership/Duty identities request Builders; Builder workflows execute in leased environments; Communicators (#04) receive progress, bottleneck, approval, and escalation events.

### Provider interfaces

```ts
type Instant = string;
type DurationMs = number;
type Deadline = { at: Instant; budgetMs: DurationMs; source: 'user'|'policy'|'parent'|'runtime' };
type WakeReason = 'message'|'timer'|'scope_effect'|'heartbeat'|'budget'|'manual'|'recovery';

interface ActorHostProvider {
  send<T>(ref: ActorRef, message: T, ctx: RuntimeContext): Promise<void>;
  ask<TReq,TRes>(ref: ActorRef, message: TReq, ctx: RuntimeContext): Promise<TRes>;
  scheduleWake(ref: ActorRef, timer: TimerSpec, ctx: RuntimeContext): Promise<TimerId>;
  passivate(ref: ActorRef, reason: string): Promise<void>;
  getState<T>(ref: ActorRef): Promise<T>;
}

interface WorkflowEngineProvider {
  start<T>(name: string, input: T, opts: WorkflowStartOptions): Promise<WorkflowRef>;
  signal<T>(ref: WorkflowRef, signal: string, payload: T): Promise<void>;
  query<T>(ref: WorkflowRef, query: string): Promise<T>;
  cancel(ref: WorkflowRef, reason: string): Promise<void>;
}

interface SandboxProvider {
  lease(req: SandboxRequest, ctx: RuntimeContext): Promise<SandboxLease>;
  exec(lease: SandboxLease, command: CommandSpec, ctx: RuntimeContext): Promise<CommandResult>;
  heartbeat(lease: SandboxLease, hb: LeaseHeartbeat): Promise<void>;
  snapshot(lease: SandboxLease, reason: string): Promise<SnapshotRef>;
  release(lease: SandboxLease, disposition: 'destroy'|'snapshot'|'keep-warm'): Promise<void>;
}

interface EnvironmentProvider extends SandboxProvider {
  openBrowser(req: BrowserRequest, ctx: RuntimeContext): Promise<BrowserLease>;
  openDesktop(req: DesktopRequest, ctx: RuntimeContext): Promise<DesktopLease>;
  leaseBuildMachine(req: BuildMachineRequest, ctx: RuntimeContext): Promise<BuildMachineLease>;
}

interface ClockProvider { now(): Instant; monotonicMs(): number; }
interface TimerService { setTimer(owner: string, spec: TimerSpec): Promise<TimerId>; cancelTimer(id: TimerId): Promise<void>; }
interface SchedulerProvider { register(spec: ScheduleSpec): Promise<ScheduleId>; pause(id: ScheduleId, reason: string): Promise<void>; fireNow(id: ScheduleId, ctx: RuntimeContext): Promise<void>; }
interface ModelGatewayProvider { complete(req: ModelRequest, ctx: RuntimeContext): Promise<ModelResponse>; embed(req: EmbedRequest, ctx: RuntimeContext): Promise<EmbedResponse>; estimate(req: ModelEstimateRequest): Promise<CostEstimate>; }
interface BudgetProvider { reserve(req: BudgetReservationRequest, ctx: RuntimeContext): Promise<BudgetReservation>; commit(reservation: BudgetReservation, usage: UsageRecord): Promise<void>; release(reservation: BudgetReservation, reason: string): Promise<void>; }
```
### Data model

```ts
type ActorRecord = {
  actorId: string; kind: 'identity'|'thread'|'clone'|'communicator'|'duty'|'ownership';
  ownerOrgId: string; identityId?: string; threadId?: string; stateVersion: number;
  lastActiveAt: Instant; passivatedAt?: Instant; nextWakeAt?: Instant;
  status: 'active'|'idle'|'blocked'|'deleted';
};

type BuilderRun = {
  runId: string; threadId: string; requestedBy: string; workflowRef: string; goal: string;
  status: 'running'|'waiting'|'succeeded'|'failed'|'compensating'|'cancelled';
  deadline: Deadline; startedAt: Instant; endedAt?: Instant;
  budgetScope: BudgetScope; environmentLeaseIds: string[];
};

type RuntimeTimer = { timerId: string; ownerId: string; fireAt: Instant; reason: WakeReason; payload: unknown; cancelledAt?: Instant };
type DurationLedgerEntry = { opKey: string; actorId?: string; threadId?: string; runId?: string; startedAt: Instant; endedAt: Instant; elapsedMs: number; deadlineMs: number; baselineP50?: number; baselineP95?: number; rarity: number; status: string; bottleneck: boolean };
type BudgetLedgerEntry = { scope: BudgetScope; model: string; inputTokens: number; outputTokens: number; cachedTokens?: number; usd: number; runId?: string; actorId?: string; threadId?: string; at: Instant };
```

### Key algorithms

**Wake routing:** #02 scope-effect, #04 messages, and timers enter a partitioned wake queue. The actor host deduplicates by `(actorId, causeId)`, loads the actor projection from #01/#03, processes one message at a time, emits events, sets the next timer, and passivates after `idleGraceMs`.

**Builder execution:** `startBuilder(goal)` creates a workflow with a deadline and budget reservation. Each step calls `withDeadline(parentDeadline, stepPolicy)`, leases an environment, starts an activity, heartbeats, records duration, commits budget, and emits progress to the thread actor. Timeout leads to retry, compensation, or #04 flag depending on policy.

**Deadline propagation:** convert absolute deadlines to remaining timeouts at process boundaries. A command without a deadline is rejected. If remaining time is below the minimum viable time, fail fast and raise a planning flag.

**15-minute time check:** every active agent session receives a recurring timer. The actor receives `TimeCheck(now, elapsedSinceStart, deadlineRemaining, budgetRemaining, stuckOperations)`. The harness injects it into context and may request escalation.

**Duration baseline:** maintain exponentially weighted quantiles per `(operationKind, tool, model, environmentClass, repo/project, rarityBucket)`. Mark bottleneck when elapsed exceeds `max(p95 * factor, fixedSlo)` or a rare operation misses an explicit deadline. Send events to #04 and proof data to #09.

### Typical flow

A human asks to build an Unreal game. #04 records the message and wakes `ThreadActor`. The actor asks #07 Duty to plan; Duty requests a Builder. Runtime starts `BuildUnrealPrototypeWorkflow` with a three-day deadline and budget. The workflow leases a GPU/Windows build machine, snapshots a workspace, calls pi.dev agents through the model gateway, runs commands with deadlines, heartbeats compile progress, commits token/spend records, and compares build durations to Unreal baselines. If shader compilation exceeds p95, TimeService emits a bottleneck. If the lease nears expiry, the workflow renews or snapshots. If unrecoverable, compensation preserves artifacts, destroys cloud resources, and flags #04 with evidence.

### Integration with the other Year96 layers

- #01 stores actor records, workflow event pointers, timer records, duration/budget ledgers, sandbox snapshots, and searchable runtime state.
- #02 emits scope-effect wake triggers and consumes runtime bottleneck/anomaly signals.
- #03 owns memory/meta-memory; runtime injects time/budget/tool observations but does not summarize thread meaning.
- #04 is the notification/escalation path for missed deadlines, bottlenecks, blocked Builders, and human approvals.
- #06 gates every actor send, workflow start, model request, sandbox network rule, and clone budget.
- #09 defines replay tests, chaos tests, deterministic simulations, observability, and proof gates.
- #11 owns pi.dev, pstack, skills, and hermes-agent integration; runtime enforces substrate contracts.

### Laptop MVP and Kubernetes cluster path

**Laptop MVP:** Temporal dev server; SQLite/Postgres state; Dapr placement or an in-process SQLite actor provider; LiteLLM proxy; Docker/container-use sandbox; Playwright browser; Wasmtime/Extism plugins; OpenTelemetry collector; simple TimerService table; pi.dev harness as the agent process.

**Single-cluster MVP:** Temporal Helm + Postgres; Dapr actors; KEDA for workers; LiteLLM or Portkey; NetworkPolicies; gVisor RuntimeClass; warm sandbox pool; object storage for snapshots/artifacts; Prometheus/Grafana/OTel; external secrets; #06 policy engine.

**Production:** multi-namespace Temporal; actor pools partitioned by org; Agent Router/Envoy at model egress; Kubernetes/GKE Agent Sandbox; Firecracker/Kata pools; GPU pools; regional TimeService; strongly consistent budget ledger; disaster recovery replay from #01.

### Testing and proof plan

Runtime testing must follow the 70% rule. Unit tests cover provider contracts, deadline math, budget reservation, quantile baselines, idempotency, and lease expiry. Integration tests run Temporal workflows with fake activities, Dapr actor activation/passivation, LiteLLM virtual keys, and sandbox command timeouts. Mocked E2E simulates years of dormant actors, timer storms, retry storms, model outage, budget exhaustion, and workflow replay after code deployment. Real E2E runs a Builder in a sandbox, kills the worker mid-command, verifies heartbeat timeout/retry, confirms no duplicate side effects, and checks #04 receives a bottleneck flag. Agentic verifiers inspect traces, screenshots, command logs, budget ledger, and final state. Chaos tests kill workers, block model providers, skew clocks within allowed bounds, revoke permissions, and fill sandbox disks.

## What is still unsolved (late 2026)

- **Ultra-dense wake storms:** millions of dormant actors are easy in storage, but mass wakeups can overload queues, model gateways, and Kubernetes. Agent Substrate is evidence this remains frontier work [13].
- **Portable snapshot/restore:** E2B, GKE Pod snapshots, Cloudflare, Vercel, and Docker expose different persistence semantics.
- **Exactly-once external side effects:** retries are safe only with idempotency keys or effect ledgers; many SaaS APIs lack them.
- **Budget semantics for thoughts/clones:** limiting self-generated thoughts without suppressing useful ownership needs policy from #06/#07/#09.
- **Long-term workflow evolution:** multi-year workflows across schema/provider migrations require strict versioning and replay discipline.
- **LLM time cognition:** runtime-injected time context and hard tool gates are necessary because prompts alone are not reliable.
- **Licensing churn:** attractive infra keeps moving to BSL/SSPL/AGPL. Year96 needs automated license gates.
- **GPU/desktop/game-engine environments:** browser/Linux sandboxes are mature; Windows/macOS GUI, Unreal/Unity, GPU drivers, and licensed toolchains remain bespoke and expensive.

## Sources

1. https://docs.temporal.io/encyclopedia/detecting-workflow-failures
2. https://docs.temporal.io/ai
3. https://temporal.io/blog/announcing-openai-agents-sdk-integration
4. https://pi.dev/
5. https://docs.dbos.dev/integrations/openai-agents
6. https://developers.cloudflare.com/durable-objects/
7. https://developers.cloudflare.com/durable-objects/api/alarms/
8. https://grpc.io/docs/guides/deadlines/
9. https://www.microsoft.com/en-us/research/project/orleans-virtual-actors/
10. https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/
11. GitHub license files opened via GitHub MCP: temporalio/temporal, temporalio/sdk-typescript, dbos-inc/dbos-transact-ts, restatedev/restate, hatchet-dev/hatchet, triggerdotdev/trigger.dev, inngest/inngest, dotnet/orleans, dapr/dapr, cloudflare/agents, ray-project/ray, golemcloud/golem, resonatehq/resonate.
12. GitHub license files opened via GitHub MCP: firecracker-microvm/firecracker, google/gvisor, kata-containers/kata-containers, e2b-dev/infra, microsandbox/microsandbox, bytecodealliance/wasmtime, extism/extism.
13. https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate
14. https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox
15. https://kagent.dev/blog/kagent-agent-substrate-sandboxes
16. https://developers.cloudflare.com/sandbox/
17. https://vercel.com/docs/sandbox
18. https://docs.litellm.ai/docs/proxy/virtual_keys
19. https://portkey.ai/docs/product/ai-gateway.md
20. https://theagentrouter.ai/docs/
21. https://docs.e2b.dev/
22. https://www.inngest.com/docs
23. https://aws.amazon.com/bedrock/agentcore/
24. https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale
25. https://learn.microsoft.com/en-us/azure/foundry/agents/overview
26. https://github.com/dagger/container-use
27. GitHub license files opened via GitHub MCP: BerriAI/litellm, Portkey-AI/gateway, envoyproxy/ai-gateway, agentgateway/agentgateway, kubernetes-sigs/gateway-api-inference-extension, lm-sys/RouteLLM, vllm-project/vllm, sgl-project/sglang.
