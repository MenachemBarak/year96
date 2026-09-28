# 12 — GitHub Trending & the OSS Agentic-OS Lineage

Scope: this report surveys late-2025/2026 open-source projects that implement pieces of Year96's 2027 "Agentic OS": always-on personal agents, coding harnesses, multi-agent orchestration, agent-native task/spec/memory systems, agentic-OS research, low-code agent builders, and registries. I verified repository metadata through GitHub/`gh repo view` on 2026-09-28 where possible; Microsoft org metadata was partially blocked by SAML, so Microsoft stars are marked unverified. Verdicts apply to **Year96 core** under the team rule that core dependencies should be permissive (MIT/Apache/BSD/ISC/public domain).

## TL;DR for the Year96 architect

- The OSS center of gravity moved from "chatbot frameworks" to **agent control planes**: OpenHands Agent Canvas, Vibe Kanban, opencode, Codex CLI, Gemini CLI, Goose, Cline/Kilo and Container Use all assume many concurrent agents, branches, sandboxes, logs, reviews, and human feedback loops [2][10][11][33].
- **OpenClaw** is the verified outlier: 390k stars, MIT license file, latest push today, and a product shape closest to "always-on assistant in chats/devices" [1]. Borrow the gateway/plugin shape, but do not make it the Year96 kernel.
- **Spec-driven development exploded**: GitHub `spec-kit` has 139k stars, MIT, and explicitly gives agents structured processes, templates, and documented outcomes [3]. Year96 should adopt/fork the methodology into every Duty→Execution handoff.
- **Durable orchestration remains the most reusable core primitive.** LangGraph (MIT, 42k stars) explicitly targets long-running stateful agents with durable execution, memory, replay and HITL [5]. Microsoft Agent Framework has the same production vocabulary but metadata was partially inaccessible; trial it as an external provider, not a hard dependency [7].
- The "AI company" lineage (MetaGPT, ChatDev, CrewAI, AutoGen/AG2, BMAD) is useful as **role/process inspiration**, not as Year96's org model. They lack evergreen threads, strict non-interfering Communicators, identity permissions, and scope-effect learning.
- The agentic-OS research line (AIOS, MemOS, UFO, Agent-S, OS-Copilot, Cradle) validates Year96's layer vocabulary: scheduling, memory, context switch, tool management, GUI/world control and memory OS [12][13][14]. Most are research/prototype-grade.
- **Memory is now a first-class OS subsystem.** MemOS (Apache-2.0, 11.6k stars) and Graphiti (Apache-2.0, 31k) are strong trials for Thread/Duty memory: editable graph memories, temporal provenance, feedback/correction, async ingestion [13][17].
- **MCP became the packaging/discovery substrate** for tools and sandboxes. The official MCP Registry is preview, stores standardized `server.json` metadata, and is backed by Anthropic/GitHub/PulseMCP/Microsoft [4]. Year96 needs a private registry mirror with pinning and attestation.
- Licensing disqualifies several beloved tools from Year96 core: Dify modified Apache, n8n fair-code/Sustainable Use, Suna Elastic-2.0, Khoj AGPL, Flowise/Activepieces commercial hybrids, AgenticSeek GPL, Crush FSL. Keep them as external integrations only.
- The gap nobody fills: a **single state/scope/thread/ownership/duty OS** where thoughts are state, scope effect is learned per identity/thread, Communicators oversee without altering decisions, and agents own "why" over years.
- Recommended core stack: **OpenHands or Vibe Kanban as agent control-plane inspiration; LangGraph/Microsoft Agent Framework as orchestration providers; Container Use for sandbox provider; MCP Registry mirror for tools; MemOS/Graphiti as memory providers; spec-kit/BMAD/12-Factor Agents as methodology providers; OpenClaw/Eliza as chat gateway examples.**

## Landscape

### 1) Always-on personal agents and chat gateways

**OpenClaw** (renamed from the Clawdbot/Moltbot line per web reports; direct repo verified) is an MIT personal assistant that runs on devices and chats, with npm packaging, CI, and chat/community integrations [1]. At 390,715 stars and 82,158 forks, it is the fastest public signal that users want an always-on assistant rather than a one-shot bot. Year96 should borrow its "assistant in every channel" posture, plugin packaging, and local-device deployment, but put Year96's Communication Hub and Identity gates above it.

**ElizaOS/eliza** (MIT, 19.5k stars) is a multi-agent/social-agent runtime for characters, Discord/Telegram/Twitter-like surfaces and plugins. It matters because chat gateways are how non-technical humans will expose threads, but Year96 must avoid personality/character coupling in core.

**Agent Zero** (MIT license file despite GitHub license detector "other", 19.3k stars) is a highly practical general agent framework with terminal/tool use. Trial its ergonomics but isolate it: agent frameworks tend to mix state, prompts, tools and side effects.

**Open Interpreter** (Apache-2.0, 68.5k stars) remains the canonical "LLM can operate my computer" assistant. Its value is UX and local command execution, not durable governance.

**Suna/Kortix** (Elastic-2.0, 20.2k stars) is a full browser/cloud agent product. It is an excellent UI reference for autonomous web work, but the Elastic license makes it Excluded-license for core.

**OpenManus** (MIT, 58.4k stars) and **CAMEL/OWL** (CAMEL Apache-2.0; OWL lacks detected SPDX/file in my check) continue the "open generalist/manus-like agent" line. Use as benchmark tasks and UI inspiration, not kernel dependencies.

**Khoj** (AGPL-3.0, 37.5k stars) is an always-on personal AI/RAG assistant; useful as an external personal-search integration, excluded from core.

### 2) Coding harnesses and agent control planes

**OpenHands Agent Canvas** describes itself as a "self-hosted developer control center for coding agents and automations" that can run OpenHands, Claude Code, Codex, Gemini, or ACP-compatible agents locally, remotely or in cloud backends; it decomposes GitHub issues, posts reports to Slack, and connects to Docker/VM/infrastructure backends [2]. With 89k stars and MIT license, it is the best Adopt/Trial candidate for Year96's Execution workbench UX.

**opencode** (repo redirected to `anomalyco/opencode`, MIT, 210k stars) is the biggest coding CLI signal in the dataset. **Codex CLI** (Apache-2.0, 127k stars), **Gemini CLI** (Apache-2.0, 107k), **Goose** (Apache-2.0, 54.7k), **Cline** (Apache-2.0, 69.5k), **Kilo Code** (MIT, 27.4k), **Qwen Code** (Apache-2.0, 28.2k), **Continue** (Apache-2.0, 36.1k), **Aider** (Apache-2.0, 49.2k), **mini-swe-agent** (MIT, 8k), and **Crush** (FSL with MIT future license, 28.3k) show the same convergence: agents need terminal access, model gateways, file editing, diff review, resumable sessions and tool servers. Year96 should not bless one coding agent; it should provide an `ExecutionAgentProvider` interface and run several behind policy gates.

**Vibe Kanban** (Apache-2.0, 28.2k stars) is especially Year96-relevant because it turns agent work into kanban issues, branch/workspace allocation, terminal/dev-server ownership, inline diff review and preview browser [3]. That looks like a concrete UI for Duty→Builder work queues.

**Container Use** (Apache-2.0, 4k stars) is a small but high-signal MCP server/CLI from Dagger: each agent gets a fresh container plus git branch, command logs, direct terminal intervention, and universal MCP compatibility [16]. This should be an Execution sandbox provider early.

### 3) Multi-agent orchestration, AI-company/org projects and methods

**LangGraph** is a low-level framework for long-running, stateful agents with durable execution, human-in-the-loop interruption, comprehensive memory and traceability [5]. It maps directly to Year96 durable runtime (#05), Thread state (#03) and Verification traces (#09). Adopt its concepts; wrap it behind provider interfaces.

**CrewAI** (MIT, 59.1k stars) remains the best-known role/team orchestration framework [6]. Its role/crew/task vocabulary is useful for Duty decomposition but too shallow for Year96 Ownership: roles finish jobs; Year96 identities own open-ended duties.

**Microsoft Agent Framework** is an open multi-language .NET/Python framework for production agents and multi-agent workflows [7]. Web research says it succeeds AutoGen and AutoGen is maintenance-mode, but direct repo metadata was blocked by SAML. Trial it for enterprise graph/checkpoint/HITL patterns; do not tie Year96 to Azure assumptions.

**Google ADK** (Apache-2.0, 21.7k stars) gives production agent SDK conventions, tools, sessions, evals and deployment [8]. Keep as a provider/backend adapter, especially for Google ecosystem customers.

**AG2** (Apache-2.0, 5k stars) is the community continuation of AutoGen. **AutoGen** itself is historically important but should be Watch/Migrate.

**MetaGPT** (MIT, 70.7k stars) and **ChatDev** (Apache-2.0, 34.4k) are classic "AI software company" simulations. They matter because they made role-based product manager/architect/engineer workflows concrete, but they are not current enough to be core.

**Agno** (Apache-2.0, 42.4k), **Pydantic AI** (MIT, 20.2k), **smolagents** (Apache-2.0, 29.6k), and **Mastra** (detected other/mixed license, 28.4k) represent the typed/tool-calling framework lane. Use them for provider implementation patterns, not central architecture.

**BMAD-METHOD** (MIT license file, 53.6k) and **Taskmaster** (MIT with Commons Clause/custom, 28.1k) show the workflow-method lane: clarify, plan, build, verify, split tasks, keep context. BMAD is Trial/Adopt as methodology content; Taskmaster's license is core-excluded.

**12-Factor Agents** (26.4k, content license detected "other") is not code infrastructure but a reliable-agent design manifesto [19]. Import its principles into Year96 standards with proper attribution; no runtime dependency required.

### 4) Agentic-OS research lineage

**AIOS** explicitly calls itself an AI Agent Operating System: it embeds LLMs into the OS and addresses scheduling, context switch, memory management, storage management, tool management and SDK management [12]. Its license was detected "other" and file content was not SPDX-clear in my quick check, so it is Watch/Research rather than Adopt.

**MemOS** is a Memory Operating System for LLMs/agents with unified add/retrieve/edit/delete APIs, graph structure, multimodal/tool/persona memories, composable memory cubes, async MemScheduler, and feedback/correction [13]. This is the closest OSS match to Year96 Thread/Duty memory. Trial.

**Microsoft UFO/UFO²**, **Agent-S**, **OS-Copilot**, and **Cradle** form the GUI/desktop OS-agent line. Agent-S is active (Apache-2.0, 12.4k) and explicitly operates real GUI by mouse/keyboard/screen with S1/S2/S3 papers [14]. OS-Copilot and Cradle are older but useful benchmarks for desktop/browser embodied actions.

**Graphiti** (Apache-2.0, 31.3k) is not branded as an OS, but its temporal knowledge graph with provenance and changing facts is more directly useful for Year96 STATE/THREAD than many agent frameworks [20].

### 5) Low-code platforms, registries and marketplaces

**Dify** (157k stars, modified Apache), **n8n** (206k, fair-code/Sustainable Use), **Langflow** (MIT, 155k), **Flowise** (custom/mixed and archived in my metadata), **Activepieces** (custom/mixed), **Sim Studio** (Apache-2.0, 29.7k) and **Rowboat** (Apache-2.0, 18k) show that visual workflows are not optional. Year96 should integrate with them externally, but only Langflow/Sim/Rowboat are core-license candidates.

The **official MCP Registry** is preview and provides a central metadata repository for public MCP servers: DNS namespaces, REST discovery, standardized install/config metadata, and `server.json` records, while code remains in npm/PyPI/Docker/etc. [4]. Year96 should run a private mirror/curated registry as the Execution tool marketplace.

Smithery and Glama are important MCP ecosystem directories, but I could not verify stable GitHub repos in the time box; treat as external marketplace sources to crawl, not core.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| OpenClaw | OSS | Always-on assistant across devices/chats; plugin/channel model | MIT (LICENSE verified) | 390,715 stars; commit 2026-09-28 | Trial |
| OpenHands | OSS | Agent Canvas control plane, issue automation, multi-backend coding agents | MIT | 89,389; commit 2026-09-28 | Adopt |
| opencode | OSS | Extremely popular terminal coding agent UX | MIT | 210,585; commit 2026-09-28 | Trial |
| GitHub spec-kit | OSS/method | Spec-driven development templates and convergence | MIT | 139,247; commit 2026-09-28 | Adopt |
| LangGraph | OSS | Durable long-running stateful workflows, memory, HITL | MIT | 42,416; commit 2026-09-28 | Adopt |
| Container Use | OSS/MCP | Per-agent container+branch sandbox, logs, intervention | Apache-2.0 | 4,047; commit 2026-09-21 | Adopt |
| Vibe Kanban | OSS/product | Kanban+workspace+branch UI for parallel coding agents | Apache-2.0 | 28,215; commit 2026-09-19 | Trial |
| MemOS | OSS/research | Memory OS, editable graph memories, async scheduler, feedback | Apache-2.0 | 11,620; commit 2026-09-23 | Trial |
| Graphiti | OSS | Temporal knowledge graph for evolving agent context | Apache-2.0 | 31,265; commit 2026-09-27 | Adopt |
| Microsoft Agent Framework | OSS | Production multi-agent workflows, .NET/Python, enterprise patterns | MIT (README/web; metadata blocked) | stars unverified; active README 2026 | Trial |
| Google ADK | OSS | Agent SDK, sessions/tools/evals/deployment | Apache-2.0 | 21,670; commit 2026-09-26 | Trial |
| CrewAI | OSS | Role-based crews, flows, task orchestration | MIT | 59,136; commit 2026-09-28 | Trial |
| Agno | OSS | Typed agent framework and tools | Apache-2.0 | 42,370; commit 2026-09-28 | Trial |
| Pydantic AI | OSS | Type-safe Python agent interfaces | MIT | 20,236; commit 2026-09-28 | Trial |
| smolagents | OSS | Lightweight code-agent framework | Apache-2.0 | 29,556; commit 2026-09-23 | Trial |
| BMAD-METHOD | OSS/method | Clarify/plan/build/verify agentic SDLC | MIT (LICENSE verified) | 53,590; commit 2026-09-28 | Adopt |
| 12-Factor Agents | standard/method | Reliability principles for LLM apps | Other/content (not core code) | 26,435; commit 2025-09-21 | Trial |
| MCP Registry | standard/OSS | Tool/server discovery metadata, `server.json`, namespaces | MIT/Apache-2.0 transition verified | 7,292; commit 2026-09-23; Anthropic/GitHub/MS ecosystem | Adopt |
| Langflow | OSS/product | Visual agent/RAG workflow builder | MIT | 155,335; commit 2026-09-28 | Trial |
| Sim Studio | OSS/product | Visual AI workflow builder | Apache-2.0 | 29,746; commit 2026-09-28 | Trial |
| Rowboat | OSS/product | Multi-agent builder/workflows | Apache-2.0 | 17,984; commit 2026-09-28 | Watch |
| Open Interpreter | OSS | Local computer-use assistant ergonomics | Apache-2.0 | 68,464; commit 2026-09-27 | Trial |
| DeerFlow | OSS | Super-agent harness with subagents, memory, sandboxes, skills | MIT | 83,156; commit 2026-09-28 | Trial |
| Codex CLI | OSS | Coding agent CLI/provider | Apache-2.0 | 126,955; commit 2026-09-28 | Trial |
| Gemini CLI | OSS | Coding/agent CLI/provider | Apache-2.0 | 107,172; commit 2026-09-28 | Trial |
| Goose | OSS | Local agent CLI with tools/MCP | Apache-2.0 | 54,735; commit 2026-09-28 | Trial |
| Cline | OSS | IDE coding agent pattern | Apache-2.0 | 69,484; commit 2026-09-28 | Trial |
| Kilo Code | OSS | Cline/Roo-derived coding agent | MIT | 27,429; commit 2026-09-28 | Watch |
| Continue | OSS | IDE assistant and model/context routing | Apache-2.0 | 36,054; commit 2026-09-28 | Trial |
| Aider | OSS | Pair-programming CLI and benchmarked editing loop | Apache-2.0 | 49,236; commit 2026-05-22 | Trial |
| mini-swe-agent | OSS | Minimal SWE-bench-style agent harness | MIT | 8,045; commit 2026-09-21 | Adopt (tests) |
| Qwen Code | OSS | Model-vendor coding CLI | Apache-2.0 | 28,195; commit 2026-09-28 | Watch |
| MetaGPT | OSS | AI-company role simulation | MIT | 70,670; commit 2026-01-21 | Watch |
| ChatDev | OSS | AI software company simulation | Apache-2.0 | 34,414; commit 2026-07-24 | Watch |
| AG2 | OSS | AutoGen continuation | Apache-2.0 | 4,965; commit 2026-09-28 | Watch |
| AIOS | OSS/paper | OS kernel concepts: scheduling, context/tool/memory mgmt | Other/unclear | 6,426; commit 2026-07-20 | Watch |
| Agent-S | OSS/paper | GUI/desktop computer-use agent | Apache-2.0 | 12,404; commit 2026-09-05 | Trial |
| OS-Copilot | OSS/paper | Older OS-agent benchmark/prototype | MIT | 1,793; commit 2024-09-09 | Watch |
| Cradle | OSS/paper | Embodied/computer-control research | MIT | 2,593; commit 2024-11-07 | Watch |
| Eliza | OSS | Social/chat agent runtime and plugin ecosystem | MIT | 19,512; commit 2026-09-28 | Trial |
| Agent Zero | OSS | General agent framework ergonomics | MIT (LICENSE verified) | 19,330; commit 2026-09-23 | Trial |
| OpenManus | OSS | Generalist autonomous-agent benchmark/inspiration | MIT | 58,433; commit 2026-08-22 | Watch |
| CAMEL | OSS | Multi-agent society/research framework | Apache-2.0 | 17,792; commit 2026-09-20 | Watch |
| OWL | OSS | CAMEL-associated agent app | NOASSERTION (no license found) | 20,150; commit 2026-09-20 | Avoid |
| Dify | OSS/product | Production low-code agent/RAG platform | Modified Apache/custom | 157,414; commit 2026-09-28 | Excluded-license |
| n8n | OSS/product | Workflow automation with AI nodes | Sustainable Use/fair-code custom | 206,205; commit 2026-09-28 | Excluded-license |
| Suna | OSS/product | Browser/cloud autonomous agent | Elastic-2.0 | 20,241; commit 2026-09-28 | Excluded-license |
| Khoj | OSS/product | Personal AI/search assistant | AGPL-3.0 | 37,531; commit 2026-08-02 | Excluded-license |
| Flowise | OSS/product | Visual LangChain workflows | Custom/mixed; archived in metadata | 55,488; commit 2026-08-13 | Excluded-license |
| Activepieces | OSS/product | Workflow automation | Custom/mixed | 24,771; commit 2026-09-28 | Excluded-license |
| Crush | OSS | Terminal coding agent | FSL-1.1, MIT future | 28,342; commit 2026-09-28 | Excluded-license |
| Taskmaster | OSS/method/tool | AI task system | MIT with Commons Clause/custom | 28,102; commit 2026-04-28 | Excluded-license |
| HumanLayer repo | OSS/product | Human approval concept; repo now deprecated | Apache-2.0 file | 11,617; commit 2026-06-19; README says deprecated [18] | Avoid |
| AgenticSeek | OSS | Local autonomous assistant | GPL-3.0 | 27,362; commit 2026-09-25 | Excluded-license |

## How I would build this part of Year96

### Dominant 2026 patterns and architectural implication

1. **Long-running loops became productized.** LangGraph, OpenHands, DeerFlow, MAF and OpenClaw all treat the agent as a resumable process, not a chat turn. Year96 should make every Duty and Builder a durable actor with checkpointed state and explicit wake/sleep rules (#05).
2. **Worktree/container parallelism is the default.** Vibe Kanban and Container Use assume one branch/container/workspace per agent. Year96 Execution should never run untrusted builders in the host workspace; every Builder gets an ephemeral capability-scoped environment (#06/#09).
3. **Agent-native issue/task systems emerged.** Vibe Kanban, Taskmaster, BMAD and spec-kit are converging on tickets that include why/spec/plan/evidence. Year96 Threads should own this natively, not sync from Jira/GitHub as an afterthought (#03/#07).
4. **Spec-first beats prompt-first.** GitHub spec-kit and BMAD encode repeatable planning artifacts. Year96 should require `IntentSpec`, `ScopeSpec`, `VerificationSpec` and `RollbackSpec` before Execution.
5. **MCP everywhere.** Tool discovery and packaging moved to MCP servers and registries. Year96 should define a private MCP registry mirror plus signed allowlists (#04/#06).
6. **Chat gateways are distribution, not architecture.** OpenClaw/Eliza show the surface, but Year96's Communication Hub must remain channel-agnostic and enforce the rule that Communicators oversee but do not decide (#04).
7. **Memory is editable, graph-like and temporal.** MemOS and Graphiti show the right direction: provenance, corrections, memory cubes, temporal facts. Year96 must add meta-memory and scope-effect weights (#01/#02/#03).
8. **Human-in-the-loop is now interruptible state editing.** LangGraph/MAF/OpenHands let humans inspect/modify. Year96 should formalize this as permissioned thread events, never out-of-band prompt nudges (#06).
9. **Visual control planes matter.** Langflow/Sim/Rowboat/OpenHands/Vibe prove that humans need boards, graphs, previews and logs. Year96 needs both CLI and visual OS consoles.
10. **Licensing and supply chain are architecture.** Many trending projects are custom/fair-code/relicensed. Provider wrappers must make replacement cheap.

### Provider interfaces

```ts
export type Verdict = "Adopt" | "Trial" | "Watch" | "Avoid" | "Excluded-license";

export interface AgentRuntimeProvider {
  kind: "langgraph" | "maf" | "openhands" | "custom";
  startThread(spec: ThreadSpec): Promise<RunHandle>;
  checkpoint(runId: string): Promise<CheckpointRef>;
  resume(checkpoint: CheckpointRef, event?: ThreadEvent): Promise<RunHandle>;
  interrupt(runId: string, reason: InterruptReason): Promise<MutableRunState>;
}

export interface ExecutionAgentProvider {
  name: "opencode" | "codex" | "gemini-cli" | "goose" | "cline" | "aider" | "openhands" | string;
  capabilities(): Promise<CapabilityManifest>;
  execute(goal: BuilderGoal, env: SandboxRef, policy: PolicyBundle): AsyncIterable<ExecutionEvent>;
}

export interface SandboxProvider {
  create(input: { threadId: string; dutyId: string; repo?: RepoRef; permissions: Capability[] }): Promise<SandboxRef>;
  observe(ref: SandboxRef): Promise<TraceBundle>;
  snapshot(ref: SandboxRef): Promise<StateSnapshot>;
  destroy(ref: SandboxRef): Promise<void>;
}

export interface MemoryProvider {
  write(event: ThreadEvent, provenance: Provenance): Promise<MemoryId>;
  retrieve(query: ScopeQuery): Promise<RankedMemory[]>;
  correct(id: MemoryId, correction: NaturalLanguagePatch): Promise<MemoryId>;
  explain(id: MemoryId): Promise<MemoryProvenance>;
}

export interface ToolRegistryProvider {
  search(query: ToolQuery): Promise<ToolPackage[]>;
  resolvePinned(pkg: ToolPackage, policy: SupplyChainPolicy): Promise<PinnedTool>;
  attest(tool: PinnedTool): Promise<AttestationReport>;
}

export interface MethodologyProvider {
  createSpec(desire: HumanDesire, context: ThreadContext): Promise<IntentSpec>;
  plan(spec: IntentSpec): Promise<PlanSpec>;
  verify(plan: PlanSpec, evidence: EvidenceBundle): Promise<VerificationResult>;
}
```

### Data model

```ts
export interface OSSComponent {
  id: string;
  repo?: string;
  layer: Year96Layer[];
  license: { spdx: string; verifiedAt: string; coreAllowed: boolean; notes?: string };
  maturity: { stars?: number; forks?: number; lastCommit?: string; archived?: boolean; backer?: string };
  providers: ProviderBinding[];
  verdict: Verdict;
  risks: string[];
}

export interface ThreadSpec {
  id: string;
  createdAt: string;
  lastActiveAt: string;
  ownerIdentityId: string;
  memoryRefs: MemoryId[];
  metaMemory: { why: string; successModel: string; nonGoals: string[] };
  milestones: Milestone[];
  listeners: IdentityId[];
  hangConditions: ScopePredicate[];
}

export interface BuilderGoal {
  dutyId: string;
  why: string;
  targetState: StatePredicate;
  timeBudgetMs: number;
  tools: PinnedTool[];
  verification: VerificationSpec;
  rollback: RollbackSpec;
}
```

### Flow

1. Human opens a desire in any OpenClaw/Slack/CLI/web channel. Communication Hub (#04) creates or reactivates a Thread.
2. Scope Effect (#02) retrieves Graphiti/MemOS memories and predicts affected Ownership/Duty identities.
3. Ownership (#07) converts desire into an `IntentSpec`; BMAD/spec-kit providers produce `PlanSpec` and `VerificationSpec`.
4. Duty creates Builder goals. Runtime (#05) selects an orchestration provider (LangGraph/MAF/custom) and sandbox provider (Container Use/Dagger/VM).
5. Execution selects coding/browser/research agents via `ExecutionAgentProvider`, pins tools through private MCP Registry, and emits events to STATE (#01).
6. Verification (#09) runs unit/integration/E2E/agentic verifiers in the sandbox, compares state snapshots, and records proofs.
7. If successful, Ownership updates why/strategy/memory; if not, Builder raises a flag through Comm Hub without silent failure.

### Mapping to Year96 layers and gaps

| Year96 layer | Strong OSS inputs | Gap Year96 must fill |
|---|---|---|
| STATE | Graphiti, MemOS, LangGraph checkpoints, OpenHands traces | Universal world-as-state, thoughts-as-state, cross-modal search over everything |
| SCOPE EFFECT | Graphiti temporal edges, MemOS retrieval feedback | Per-identity/thread scope prediction and "generate thoughts" wakeups |
| THREADS | Vibe Kanban issues, OpenHands conversations, LangGraph state | Never-closing threads with meta-memory, listeners/hangers/clones |
| COMM HUB | OpenClaw/Eliza gateways, MCP/A2A patterns | Non-interfering Communicator semantics and auditable oversight |
| OWNERSHIP | BMAD, spec-kit, 12-Factor Agents | Capturing/improving human mental model and why over years |
| DUTY | Taskmaster, BMAD, Vibe Kanban | Permissioned sibling Duty discussion and ongoing process memory |
| EXECUTION | OpenHands, opencode, Codex/Gemini/Goose/Cline, Container Use | Provider-neutral Builders with capability contracts and hard gates |
| SELF-IMPROVEMENT | MemOS feedback, spec-kit retros, verifiers | Safe promotion/evolution of Year96 architecture itself |
| VERIFICATION | mini-swe-agent, OpenHands logs, Container Use logs, spec-kit evidence | 70% proof regime across starting states/environments and agentic judges |

### Scaling path

Laptop: SQLite/Postgres state, local MemOS/Graphiti, Container Use on Docker, one OpenHands/Vibe Kanban console. Team server: Postgres+Kafka/NATS event log, private MCP registry, sandbox pool, multiple execution providers. Cluster: durable workflow engine, object-store trace archive, policy engine, signed tool mirror, per-tenant memory cubes and vector/graph shards. The provider interfaces keep every layer swappable.

### Testing and proofs

For each adopted component: license test (SPDX allowlist), provider contract tests, golden-thread replay tests, sandbox escape tests, deterministic fixture tasks, E2E "desire→thread→duty→builder→proof" tests, and adversarial verifiers. Every tool call logs before-state, expected end-state, live trace, timeout, final state, and provenance. Fast-moving repos are pinned by commit SHA/container digest with SBOM, SLSA/GitHub provenance when available, npm/PyPI lockfiles, and periodic vulnerability scans.

### Supply-chain/security notes

- Never install directly from trending README curl scripts in core. Mirror packages, pin hashes, and verify signatures/attestations.
- Treat MCP servers as remote code/tool bridges: require sandboxing, least privilege, egress policy, secrets broker, audit logs and per-thread capability leases.
- Maintain license gates: custom/fair-code/copyleft packages may run as optional external integrations, never as Year96 kernel/core libraries.
- Re-run license detection because repos relicense frequently (Flowise archived/mixed; Crush FSL→MIT future; Dify/n8n modified terms).
- Keep provider replacements ready for every vendor-backed CLI; Codex/Gemini/Qwen/Goose are valuable but strategically non-exclusive.

## What is still unsolved (late 2026)

- **Universal state remains unsolved.** Graphiti/MemOS solve agent memory slices, not "clock, rock, thought, network, repo, GUI, business KPI" as one searchable state fabric.
- **Scope effect is mostly absent.** Frameworks retrieve context; they do not learn which changes matter to which identity/thread over years, nor do they model dormant threads waking from weak signals.
- **Ownership is shallow.** Crew/role systems assign tasks; they do not own the why, improve strategy, negotiate permissions, or maintain human mental models.
- **Communication governance is missing.** Chat gateways mix oversight, decisions and execution. Year96's Communicator non-interference rule is novel and needs formal tests.
- **Agent-native identity/permissions are immature.** MCP and sandboxes help, but clone attenuation, ReBAC, consent, policy proofs and auditable capability leases are not turnkey.
- **Verification is underbuilt.** Most repos optimize for demos. Year96's 70% verification rule requires investing more in replay, simulation, oracles and agentic judges than any one project provides.
- **License volatility is high.** Some of the hottest projects use "other" licenses, Commons Clauses, fair-code, FSL, Elastic or modified Apache terms. Provider isolation is mandatory.
- **Self-improvement lacks safe promotion.** Tools can remember and iterate, but few can propose, test, canary, rollback and promote changes to their own architecture.

## Sources

1. https://github.com/openclaw/openclaw
2. https://github.com/OpenHands/OpenHands
3. https://github.com/github/spec-kit
4. https://modelcontextprotocol.io/registry/about and https://github.com/modelcontextprotocol/registry
5. https://github.com/langchain-ai/langgraph
6. https://github.com/crewAIInc/crewAI
7. https://github.com/microsoft/agent-framework
8. https://github.com/google/adk-python
9. https://github.com/bytedance/deer-flow
10. https://github.com/BloopAI/vibe-kanban
11. https://github.com/dagger/container-use
12. https://github.com/agiresearch/AIOS
13. https://github.com/MemTensor/MemOS
14. https://github.com/simular-ai/Agent-S
15. https://github.com/bmad-code-org/BMAD-METHOD
16. https://github.com/eyaltoledano/claude-task-master
17. https://github.com/getzep/graphiti
18. https://github.com/humanlayer/humanlayer
19. https://github.com/humanlayer/12-factor-agents
20. https://github.com/langgenius/dify
21. https://github.com/n8n-io/n8n
22. https://github.com/langflow-ai/langflow
23. https://github.com/FlowiseAI/Flowise
24. https://github.com/activepieces/activepieces
25. https://github.com/simstudioai/sim
26. https://github.com/rowboatlabs/rowboat
27. https://github.com/openinterpreter/openinterpreter
28. https://github.com/elizaOS/eliza
29. https://github.com/agent0ai/agent-zero
30. https://github.com/FoundationAgents/OpenManus
31. https://github.com/camel-ai/camel and https://github.com/camel-ai/owl
32. https://github.com/khoj-ai/khoj
33. https://github.com/anomalyco/opencode
34. https://github.com/openai/codex
35. https://github.com/google-gemini/gemini-cli
36. https://github.com/aaif-goose/goose
37. https://github.com/cline/cline
38. https://github.com/Kilo-Org/kilocode
39. https://github.com/Aider-AI/aider
40. https://github.com/SWE-agent/mini-swe-agent
41. https://github.com/charmbracelet/crush
42. https://github.com/QwenLM/qwen-code
43. https://github.com/continuedev/continue
44. https://github.com/FoundationAgents/MetaGPT
45. https://github.com/OpenBMB/ChatDev
46. https://github.com/ag2ai/ag2
47. https://github.com/agno-agi/agno
48. https://github.com/pydantic/pydantic-ai
49. https://github.com/huggingface/smolagents
50. https://github.com/PrimeIntellect-ai/verifiers
51. https://github.com/Fosowl/agenticSeek
