# 11 — Harness & methodology stack

Scope: this report evaluates the mandated Year96 harness/methodology stack — pi.dev, NousResearch hermes-agent, cursor/plugins `pstack`, obra/superpowers, mattpocock/skills, plus Agent Skills and AGENTS.md — and turns it into a concrete Year96 Harness design for Communicator, Owner, Duty, Builder, Verifier, and Sensor identities. The key question is not which agent is “best”; it is how to make every identity auditable, permissioned, time-aware, state-capturing, and unable to claim completion without proof.

## TL;DR for the Year96 architect

- **Use Pi as the core identity harness, but extend before forking.** The active source I verified is `earendil-works/pi`; the older `badlogic/pi-mono` naming still appears in some ecosystem pages, but pi.dev points to the current project. Pi is MIT, TypeScript, active on 2026-09-28, and its README lists `pi-coding-agent`, `pi-agent-core`, `pi-ai`, `pi-durable`, `pi-telemetry`, `pi-tui`, session backends, protocol, and server packages [1][2][4].
- **Pi’s primitive-first design is exactly what Year96 needs.** It has TypeScript SDK, JSON/RPC modes, tree sessions, skills, prompt templates, custom providers, extensions, tool hooks, context transforms, and package distribution [1][9][10][11][12].
- **Pi is not enough for security.** Its README explicitly says it lacks a built-in permission system for filesystem, process, network, or credentials; tools/extensions run with the process permissions [2]. Year96 must wrap Pi with #06 identity gates, #05 sandboxes/durable runtime, and #09 proof/observability.
- **Hermes should be the monitored automation runner, not the canonical runtime.** Hermes brings cron, messaging gateways, memory, skills learning, MCP, subagents, terminal backends, no-agent scheduled scripts, and trajectory capture [13][14][15][16][17][18]. Use it for Sensor/Monitor jobs, scheduled thoughts, watchdogs, and gateway notifications.
- **pstack is the primary methodology.** `poteto-mode` routes work through rigorous playbooks, proof, design-space exploration, subagents, `arena`, `swarm`, `interrogate`, and principles such as prove-it-works, build-the-lever, sequence-verifiable-units, and never-block-on-the-human [19][20].
- **Superpowers is the secondary methodology.** Its automatic SDLC — brainstorming, worktrees, writing plans, subagent-driven/executing plans, TDD, code review, branch finish — gives Year96 a clean delivery discipline [21].
- **mattpocock/skills is tertiary but essential for mental-model capture.** Its strongest Year96 value is grilling, shared language, domain modeling, ADR/context docs, wayfinding, triage, TDD, diagnosis, and code review [23].
- **Standardize methodology as Agent Skills.** `SKILL.md` is now the portable format across Pi, Hermes, Cursor, Claude Code, Codex/OpenAI, GitHub Copilot, VS Code, Gemini CLI, and others [9][17][25][26][27][28][29][30][31]. Canonical Year96 skills should live in `.agents/skills`.
- **Use AGENTS.md for static project guidance, not workflows.** AGENTS.md is now an AAIF/Linux Foundation founding project contribution; it is a simple universal project-specific guidance file for coding agents [32][33][34]. Long procedures belong in skills.
- **Recommended package plan:** `@year96/harness-core`, `@year96/harness-pi`, `@year96/pi-extensions`, `@year96/methodology`, `@year96/roles-*`, `@year96/hermes-bridge`, `@year96/proof-kit`, `@year96/otel`, and `@year96/conformance`.
- **License status is mostly safe.** Pi MIT, Hermes MIT, pstack subdir MIT, Superpowers MIT, Matt skills MIT, AGENTS.md MIT, Agent Skills code Apache-2.0/docs CC-BY-4.0 [8][22][24][25][35][37]. The parent `cursor/plugins` API reports no repo-level license, so vendor pstack by exact subdir/license SHA.

## Landscape

### Pi / pi.dev agent harness — Aug 2025–Sep 2026

Pi is a minimal, self-extensible agent harness and coding agent. The verified repository is `earendil-works/pi`; the GitHub API reported TypeScript, MIT, about 110k stars, and a push on 2026-09-28 [4]. The README lists the coding agent CLI, core agent runtime, unified multi-provider LLM API, durable runtime, telemetry, TUI, session backends, protocol, and server packages [2][3]. The website says Pi deliberately ships primitives rather than baked-in features: subagents, plan mode, permission gates, protected paths, SSH, sandboxing, MCP integration, custom editors, status bars, and overlays are extensions/examples rather than immutable product behavior [1][7].

Pi matters because Year96 needs a harness it can change. The SDK embeds sessions inside a Node/Bun host [10]; RPC runs Pi as JSONL child process for other languages or isolation [11]; sessions are JSONL trees with branch/fork semantics [12]; skills implement progressive disclosure [31]; extensions can observe/modify lifecycle, register tools/commands/providers, mutate/block tool calls, transform context, persist entries, and react at `agent_before_settle` [9]. That set is unusually close to the Year96 spec: interface-based, swappable, stateful transcript tree, and self-modifiable workflow.

The caveat is security and authority. Pi’s own README says it does not sandbox filesystem/process/network/credential access [2]. Therefore Pi should run inside Year96’s identity/runtime envelope, not serve as that envelope.

### NousResearch/hermes-agent — Jul 2025–Sep 2026

Hermes Agent is a Python self-improving agent from Nous Research. The verified repo is MIT, Python, API-reported about 249k stars, and pushed on 2026-09-28 [6][35]. The README describes a self-improving agent with memory, autonomous skill creation, skill self-improvement, session search, Honcho user modeling, cron, messaging gateways, isolated subagents, RPC scripts, terminal backends (local, Docker, SSH, Singularity, Modal, Daytona, Vercel), and trajectory generation [13].

The architecture docs show a broad agent application: `run_agent.py`, `agent/context_engine.py`, `memory_manager.py`, SQLite state, tool registry, terminal backends, MCP facade, delegate tool, gateway session/delivery/pairing/hooks, `cron/`, plugins, skills, optional skills, and tests [14]. Cron can schedule one-shot/recurring jobs, attach skills, deliver results to platforms/files/origin chats, run no-agent scripts, trigger on webhooks, preflight provider/skill/platform/MCP config, pin model/reasoning, and run inside a workdir [15]. Security docs list authorization, dangerous-command approval, write safety, container isolation, MCP credential filtering, context-file scanning, cross-session isolation, and input sanitization [36].

Hermes matters because Year96 explicitly needs repetitive automations and monitored ongoing processes. It should run Sensor/Monitor identities: scheduled thoughts, watchdogs, daily reviews, alerts, and “hang on thread until relevant state changes.” It should not own canonical state, identity policy, thread truth, or proof verdicts; those belong to teammates #01, #03, #05, #06, and #09.

### cursor/plugins `pstack` — Jan 2026–Sep 2026

`pstack` is a Cursor plugin by Lauren “poteto” Tan. The parent `cursor/plugins` repo API reports TypeScript, about 8.8k stars, a 2026-09-28 push, and no repo-level license [5]. The `pstack/LICENSE` file is MIT [37]. Its README says pstack exists because AI writes too much slop code; its goal is to go deep first, write less, write better, and enable “fearless parallelism” [19].

The main entry point is `/poteto-mode`, configured by `/setup-pstack`. It has 23 playbooks: investigation, bug fix, perf, hillclimb, runtime forensics, trace forensics, feature, refactoring, prototype, visual parity, authoring a skill, eval, babysit, shipping, autonomous run, orchestrate, autopilot-full, autopilot-stack, session pickup, pause safely, multi-phase plan, worktree cleanup, and opening a PR [19]. It includes skills such as `how`, `why`, `recall`, `blast-radius`, `architect`, `arena`, `swarm`, `interrogate`, `automate-me`, `reflect`, `teach`, `tdd`, verification-skill creation/maintenance, `show-me-your-work`, `unslop`, and technical writing [19][20]. Its principle skills include foundational thinking, build-the-lever, model-the-domain, boundary discipline, type-system discipline, idempotence, prove-it-works, fix-root-causes, sequence-verifiable-units, test-behavior-not-implementation, guard-context-window, never-block-on-human, and encode-lessons-in-structure [19].

This is the closest mandated stack element to Year96’s capable-agent ideal: autonomous, rigorous, parallel, evidence-hungry, and designed to convert lessons into structure.

### obra/superpowers — Oct 2025–Sep 2026

Superpowers is a complete software development methodology for coding agents. API metadata reports MIT, about 292k stars, and a push on 2026-09-27 [38]; LICENSE is MIT [22]. It supports many harnesses, including Pi and Hermes, and explains native install paths for both [21]. Its basic workflow is `brainstorming`, `using-git-worktrees`, `writing-plans`, `subagent-driven-development` or `executing-plans`, `test-driven-development`, `requesting-code-review`, and `finishing-a-development-branch` [21]. It also includes systematic debugging, verification before completion, diagnosing-superpowers, receiving code review, dispatching parallel agents, and writing skills [21].

Superpowers matters because it gives Year96 a clear fallback SDLC: design before build, red/green TDD, small tasks, review gates, and branch completion. It is less role-specific than pstack but easier to enforce for ordinary software delivery.

### mattpocock/skills — Feb 2026–Sep 2026

Matt Pocock’s `skills` repo is MIT, API-reported about 271k stars, and pushed on 2026-09-24 [39][24]. Its README frames the skills as “Skills For Real Engineers”: small, adaptable, composable, and model-independent [23]. It targets four agent failure modes: misalignment, verbosity/jargon, missing feedback loops, and software entropy [23]. Engineering skills include `ask-matt`, `grill-with-docs`, `triage`, `improve-codebase-architecture`, `setup-matt-pocock-skills`, `to-spec`, `to-tickets`, `implement`, `wayfinder`, plus model-invoked `prototype`, `diagnosing-bugs`, `research`, `tdd`, `domain-modeling`, `codebase-design`, `code-review`, `resolving-merge-conflicts`, and `wizard` [23][40]. Productivity skills include `grill-me`, `handoff`, `teach`, `to-questionnaire`, `wait-what`, `grilling`, and `writing-for-agents` [23][41].

This matters most for Communicator, Owner, and Duty identities. Matt’s “grilling” and shared-language patterns are directly relevant to Year96’s need to capture why, domain language, preferences, ADRs, and mental models before execution.

### Agent Skills (`SKILL.md`) — 2025–2026

Agent Skills is the portable skill standard. The `agentskills/agentskills` README says a skill is a folder with `SKILL.md`, optional `scripts/`, `references/`, and `assets/`, loaded by progressive disclosure: discovery metadata, activation instructions, then resources on demand [25]. The spec requires `name` and `description`, with optional `license`, `compatibility`, `metadata`, and experimental `allowed-tools` [26]. The repo is Apache-2.0 for code and CC-BY-4.0 for docs [25].

The standard is now supported or documented by GitHub Copilot [27], Claude Code [28], OpenAI/Codex [29], Cursor [30], Pi [31], and Hermes [17]. Therefore Year96 should write canonical methodology as Agent Skills and only generate harness-specific packaging where required.

### AGENTS.md and AAIF — Aug/Dec 2025–2026

AGENTS.md is “a README for agents”: a predictable project-specific guidance file [32]. The AAIF/Linux Foundation announcement on 2025-12-09 says AGENTS.md was released by OpenAI in August 2025 and donated as a founding AAIF project alongside MCP and goose; it claimed adoption by more than 60,000 open source projects and frameworks including Amp, Codex, Cursor, Devin, Factory, Gemini CLI, GitHub Copilot, Jules, and VS Code [33]. The `agentsmd/agents.md` repo has an MIT license in its listing [34].

Use AGENTS.md for static repo guidance: commands, conventions, safety notes, project layout, and contribution rules. Do not put long workflows there; those belong in skills to preserve progressive disclosure and versioned methodology.

### Claude Code, Cursor, OpenAI/Codex, and GitHub Copilot skill/plugin ecosystems

Claude Code plugins package skills, agents, hooks, MCP servers, and other components as one installable unit [28]. Cursor supports Agent Skills plus frontmatter extensions such as `paths`, `disable-model-invocation`, `icon`, and `color`; it also has built-in skills like `/automate`, `/autopilot`, `/create-skill`, `/loop`, `/review`, and `/review-security` [30]. OpenAI/Codex skills build on Agent Skills and can be distributed through plugins that include skills, connectors, and MCP server configuration [29]. GitHub Copilot supports project and personal skills in `.github/skills`, `.claude/skills`, `.agents/skills`, `~/.copilot/skills`, and `~/.agents/skills` [27].

The conclusion is clear: Year96’s methodology should be harness-portable. Pi is the base runtime, but skills and AGENTS.md should remain usable by Hermes, Claude, Cursor, Codex, and Copilot.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| Pi (`earendil-works/pi`) | OSS harness | TypeScript agent core, CLI, SDK/RPC/JSON, tree sessions, extensions, skills, providers, TUI, packages | MIT, LICENSE verified [8] | API: ~110k stars, pushed 2026-09-28, Earendil Works/Mario Zechner [4] | Adopt |
| NousResearch/hermes-agent | OSS agent/automation runner | Cron, messaging gateways, memory, MCP, subagents, terminal backends, skills learning, trajectories | MIT, LICENSE verified [35] | API: ~249k stars, pushed 2026-09-28, Nous Research [6] | Trial |
| cursor/plugins `pstack` | OSS plugin/methodology | Primary rigorous playbooks, principles, verification, swarm/arena/interrogate | MIT in `pstack/LICENSE` verified [37]; parent repo license null via API [5] | API parent: ~8.8k stars, pushed 2026-09-28, Cursor | Adopt with license-copy guard |
| obra/superpowers | OSS methodology/plugin | Secondary SDLC: brainstorming, planning, TDD, worktrees, reviews, verification, multi-harness packaging | MIT verified [22] | API: ~292k stars, pushed 2026-09-27, Prime Radiant/Jesse Vincent [38] | Adopt |
| mattpocock/skills | OSS skills | Tertiary mental models: grilling, specs/tickets, domain language, ADRs, diagnosis, code review | MIT verified [24] | API: ~271k stars, pushed 2026-09-24, Matt Pocock/AI Hero [39] | Adopt |
| Agent Skills / SKILL.md | Standard + OSS reference | Portable skill format and progressive disclosure across harnesses | Apache-2.0 code; docs CC-BY-4.0 verified [25] | Active spec; broad client adoption [25][27][28][29][30] | Adopt |
| AGENTS.md | Standard + OSS site | Static project guidance plane for agents | MIT in repo listing [34] | AAIF founding project; >60k OSS projects claimed [33] | Adopt |
| Claude Code plugins | Product/plugin ecosystem | Packaging model for skills+agents+hooks+MCP; portability target | Product terms; plugin contents vary | Mature docs and marketplace model [28] | Trial as distribution target |
| Cursor skills/plugins | Product/plugin ecosystem | pstack origin; automations, custom modes, paths-scoped skills | Product terms; plugin contents vary; pstack MIT | Active Cursor official repo/docs [5][30] | Trial as distribution target |
| OpenAI/Codex skills/plugins | Product/plugin ecosystem | Codex/ChatGPT skill support, plugin distribution, connectors/MCP | Product terms; skills vary | Active docs; Agent Skills-compatible [29] | Trial as distribution target |

## How I would build this part of Year96

### 1. Fork vs extend Pi

Recommendation: **extend Pi first; fork only if conformance tests prove a hard gate cannot be enforced.** Pi already exposes lifecycle and tool-call boundaries through extensions [9], and the SDK lets Year96 supply custom tools, resources, models, sessions, and managers [10]. A fork would create API-churn and merge debt against a fast-moving project. Instead, build a Year96 distribution that loads Pi with a locked extension pack, provider interfaces, canonical skills, role profiles, and a conformance suite. Fork narrowly if Pi cannot enforce command wrapping, policy denials, proof gates, or state capture before/after tool use.

Proposed layout:

```text
packages/
  harness-core/          # provider interfaces, role/profile schema, policy contracts
  harness-pi/            # Pi SDK host, ResourceLoader, session manager adapters
  pi-extensions/         # timeouts, state capture, proof gate, talk gate, audit, OTel
  methodology/           # canonical .agents/skills + precedence manifest
  roles-communicator/
  roles-owner/
  roles-duty/
  roles-builder/
  roles-verifier/
  roles-sensor/
  hermes-bridge/         # cron/gateway/job adapter
  proof-kit/             # evidence schemas, proof runners, verifier APIs
  conformance/           # tests against Pi/Hermes adapters
```

Core provider interfaces:

```ts
export type IdentityKind = "communicator" | "owner" | "duty" | "builder" | "verifier" | "sensor";

export interface IdentityProfile {
  id: string;
  kind: IdentityKind;
  threadId: string;
  parentIdentityId?: string;
  modelPolicy: ModelPolicy;
  tools: ToolGrant[];
  skills: SkillGrant[];
  permissions: CapabilityGrant[];
  talkPolicy: TalkPolicyRef;
  timeBudget: TimeBudget;
  proofPolicy: ProofPolicy;
  observability: ObservabilityPolicy;
}

export interface HarnessProvider {
  startSession(input: StartSession): Promise<AgentSessionRef>;
  send(input: AgentMessageInput): Promise<AgentTurnResult>;
  steer(input: SteeringInput): Promise<void>;
  fork(input: ForkSessionInput): Promise<AgentSessionRef>;
  loadSkills(skills: SkillBundle[]): Promise<void>;
  dispose(sessionId: string): Promise<void>;
}

export interface PolicyProvider {
  authorizeToolCall(call: ToolCallIntent, ctx: IdentityContext): Promise<PolicyDecision>;
  authorizeTalk(msg: TalkIntent, ctx: IdentityContext): Promise<PolicyDecision>;
  redact(input: unknown, ctx: IdentityContext): Promise<RedactionResult>;
}

export interface StateCaptureProvider {
  captureBefore(op: OperationIntent): Promise<StateSnapshotRef>;
  captureDuring(op: OperationRef): AsyncIterable<StateObservation>;
  captureAfter(op: OperationRef): Promise<StateSnapshotRef>;
  diff(before: StateSnapshotRef, after: StateSnapshotRef): Promise<StateDiff>;
}

export interface ProofProvider {
  declareExpectedEndState(goal: GoalSpec): Promise<ExpectedState>;
  collectEvidence(run: RunRef): Promise<EvidenceBundle>;
  verify(bundle: EvidenceBundle, policy: ProofPolicy): Promise<ProofVerdict>;
}
```

### 2. Role profiles

**Communicator.** Pi profile with thread/message tools only: create/update thread, invite identities, summarize, scope-check, and route to #04 communication channels. No execution tools. Skills: pstack `why`, `recall`, `teach`; Matt `grill-me`, `wait-what`, `to-questionnaire`; Superpowers `brainstorming`. It must never decide technical outputs or modify Builder artifacts. Every external talk action calls #06 policy.

**Owner.** Strategic identity that owns “why” and Duty lifecycle. It does not execute. Tools: mental-model memory, strategy CRUD, duty creation, sensor request, policy consultation. Skills: Matt `grill-with-docs`, `domain-modeling`, `wayfinder`; pstack `why`, `attack-the-premise`, `foundational-thinking`; Superpowers `brainstorming`.

**Duty.** Converts Owner mental model into tasks and ongoing processes. Tools: task decomposition, Builder request, Verifier request, sensor spec, state query. Skills: pstack `architect`, `blast-radius`, `figure-it-out`, `sequence-verifiable-units`; Superpowers `writing-plans`; Matt `to-spec`, `to-tickets`, `triage`.

**Builder.** Execution identity. Tools: repo/file/shell/browser/MCP according to capability grants, but every tool call is wrapped. Skills: pstack `poteto-mode` primary; Superpowers TDD/executing-plans/subagent-driven-development; Matt `implement`, `tdd`, `diagnosing-bugs`. Builder cannot declare final done; it submits evidence.

**Verifier.** Independent proof identity. Tools: read-heavy diff, test, browser, observability, state diff, verifier runners. Skills: pstack `interrogate`, `prove-it-works`, `test-behavior-not-implementation`; Superpowers `verification-before-completion`, `requesting-code-review`; Matt `code-review`. Verifier signs or rejects Proof-of-Done.

**Sensor/Monitor.** Hermes-backed automation identity. Tools: schedule, webhook ingestion, state query, limited sandbox commands, talk-request. Skills: sensor-specific Agent Skills plus pstack `runtime-forensics`/`hillclimb` when investigating. It generates “thoughts” on quiet threads by running scoped periodic prompts against thread state.

### 3. Pi extensions/hooks implementing Year96 rules

Pi extensions should enforce Year96 rules programmatically, not by instruction text. Pi exposes `tool_call`, `tool_result`, `before_agent_start`, `turn_end`, `agent_before_settle`, commands, tools, custom session entries, and context transforms [9]. Use them as follows:

1. **`timeout-wrapper`:** intercept command/bash/tool execution, attach the role’s deadline, emit span, and terminate only owned processes on timeout.
2. **`state-capture`:** before every command/test/process/tool, call `captureBefore`; while running, stream observations; after result, call `captureAfter` and diff. Store refs as custom session entries and in #01 state fabric.
3. **`talk-permission`:** intercept all messaging tools and custom talk tools; ask #06 whether this identity may talk to that target on that thread.
4. **`clock-check`:** session timer injects a 15-minute check-in event and records current time, elapsed time, and bottleneck classification.
5. **`proof-gate`:** at `agent_before_settle`, detect completion claims. If no Proof-of-Done bundle/verifier verdict exists, block finalization or force a continuation.
6. **`audit-emitter`:** emit append-only audit events with identity, thread, policy decision, before/after state, evidence refs, and OTel trace ID.
7. **`otel-bridge`:** map sessions, turns, tool calls, skill loads, policy denials, proof gates, and Hermes job callbacks into OpenTelemetry spans for #09.
8. **`methodology-loader`:** resolves skill precedence and loads canonical pstack/superpowers/Matt skills from `.agents/skills`.

Data model:

```ts
export interface HarnessAuditEvent {
  id: string;
  ts: string;
  threadId: string;
  identityId: string;
  sessionId: string;
  op: "turn" | "tool_call" | "command" | "skill_load" | "proof_gate" | "talk";
  before?: StateSnapshotRef;
  after?: StateSnapshotRef;
  policyDecision?: PolicyDecision;
  otelTraceId: string;
  evidenceRefs: string[];
}

export interface ProofOfDone {
  goalId: string;
  expectedState: ExpectedState;
  unit: EvidenceRef[];
  integration: EvidenceRef[];
  mockedIntegration: EvidenceRef[];
  e2e: EvidenceRef[];
  mockedE2e: EvidenceRef[];
  agenticVerifier: EvidenceRef[];
  environmentMatrix: EvidenceRef[];
  startingStateMatrix: EvidenceRef[];
  verdict: "pass" | "fail" | "waived";
  waiver?: { approver: string; reason: string; expiresAt: string };
}
```

### 4. Methodology composition

Precedence should be deterministic:

1. Year96 constitutional gates: permissions, safety, proof, observability, timeouts, content/security restrictions.
2. Role profile: Owner never executes; Communicator never interferes with decisions; Builder cannot sign done.
3. pstack playbooks/principles: default rigorous route for nontrivial work.
4. Superpowers: SDLC discipline for software delivery once design/spec exists.
5. mattpocock/skills: alignment, shared language, domain modeling, specs/tickets, teaching, handoff.
6. Project AGENTS.md and repo skills: specialize commands/context but cannot weaken higher rules.

Conflict examples: if Matt `grill-with-docs` wants more questions but pstack `never-block-on-the-human` says proceed, Owner/Duty may proceed with explicit assumptions if action is reversible; Communicator must preserve the question for human follow-up. If Superpowers says “write plan” but pstack bug-fix says “reproduce first,” the Year96 bug-fix profile requires reproduction evidence before implementation. If a project skill says tests are optional, Proof-of-Done overrides it.

Mapping to layers:

- Communicator: Matt grilling/wait-what/to-questionnaire + pstack why/recall + Superpowers brainstorming.
- Owner: Matt domain-modeling/wayfinder + pstack foundational-thinking/why.
- Duty: pstack architect/blast-radius/figure-it-out + Superpowers writing-plans + Matt to-spec/to-tickets.
- Builder: pstack poteto-mode + Superpowers TDD/executing-plans + Matt implement/tdd/diagnosing-bugs.
- Verifier: pstack interrogate/prove-it-works + Superpowers verification/code-review + Matt code-review.
- Sensor: Hermes cron skills + pstack runtime-forensics/hillclimb + custom Year96 monitor skills.

### 5. Hermes integration protocol

Hermes responsibilities:

- scheduled thoughts and periodic thread reviews;
- watchdogs for CI, PRs, services, inboxes, calendars, analytics, and state feeds;
- message gateway adapters when #04 selects Hermes as the connector;
- no-agent scheduled scripts where no LLM is needed;
- skill learning from repeated automations after #08/#09 approval;
- trajectory capture for self-improvement experiments.

Hermes must not own canonical identity permissions (#06), thread memory (#03), universal state (#01), durable workflow truth (#05), final proof verdicts (#09), or external communication policy (#04/#06).

Protocol: #05 creates a `SensorJobSpec` with identity, thread, workdir, skills, model pin, reasoning effort, toolsets, permissions, and callback URL. `@year96/hermes-bridge` installs or updates the Hermes cron job. Each run posts `SensorRunEvent` containing transcript refs, logs, tool outputs, delivery target, proposed state changes, and evidence refs. Year96 Verifier checks the run before state is promoted. Hermes job drift is reconciled periodically against #01 desired state.

### 6. Scaling path

Laptop: Pi SDK in-process for interactive identities; Hermes local profile for sensors; file/SQLite-backed state; local OTel collector.

Team server: Pi RPC workers in containers; Hermes gateway/cron per profile; shared Postgres/event log; object store for evidence; centralized OTel collector; sandboxed terminal backends.

Cluster: #05 durable actor/workflow runtime schedules identities; Pi sessions run in ephemeral pods; Hermes sensors run as isolated profiles/workers; #01 state fabric is append-only log + graph/search index; proof evidence is content-addressed; #06 serves ReBAC/capability decisions.

### 7. Testing and proving the harness

The conformance suite is part of the product, not optional:

- Unit: provider interfaces, skill precedence, role profile validation, proof schema, policy decisions.
- Integration: Pi extension hooks with fake tools and fake policy provider; Hermes bridge against test cron profile.
- Mocked integration: fake state fabric, fake OTel collector, fake #06 denials.
- E2E: Builder tries to claim done without proof -> blocked; command without timeout -> rejected; external talk without permission -> denied; Sensor cron misconfigured -> no LLM spend and blocked alert.
- Agentic verifier tests: adversarial agents try to bypass role boundaries, skip tests, mutate state through side channels, or hide failed evidence.
- Environment matrix: local, Docker, SSH/cloud sandbox; Windows/Linux; online/offline.
- Starting-state matrix: clean, compacted, branched, resumed Hermes job, stale skill, revoked credential.
- Golden traces: every test asserts before/after snapshots, audit events, and OTel spans.

## What is still unsolved (late 2026)

1. **Hard isolation remains outside Pi.** Pi shares process permissions [2][9]. Year96 must prove sandboxing through #05/#06.
2. **Skill conflict resolution is not standardized.** Agent Skills standardizes packaging, not precedence among skills, roles, policies, and project instructions.
3. **Proof-of-Done is domain-specific.** Year96 needs proof templates and learned verifier expectations per domain.
4. **Pi tree sessions are not durable workflows.** They are excellent conversation branches, not exactly-once multi-day orchestration. #05 owns durability.
5. **Hermes memory is useful but too local/small for universal state.** Its `MEMORY.md`/`USER.md` pattern [16] complements but cannot replace #01/#03.
6. **Supply-chain/provenance of skills needs tooling.** pstack’s subdir license is verified but parent repo has no API license [5][37]. Vendor exact SHAs and scan bundled assets.
7. **Self-improving skills are risky.** Hermes `/learn`, pstack `reflect`, and Superpowers `writing-skills` must go through #08 promotion and #09 evals before default use.
8. **A2A messaging is still fragmented.** MCP covers tools; SKILL.md covers procedures; AGENTS.md covers static guidance. Identity-to-identity delegation, consent, and proof exchange still need Year96 #04/#06.
9. **Agent observability schemas are immature.** AAIF has an Observability & Traceability working group [34], but concrete LLM/tool/proof/span schemas are not settled.
10. **Human mental-model capture is not solved.** Matt’s skills help, but Owner must learn why/preferences/strategies without inventing or overfitting.

## Sources

1. https://pi.dev/
2. https://raw.githubusercontent.com/earendil-works/pi/main/README.md
3. GitHub MCP directory listing: https://github.com/earendil-works/pi/tree/main/packages
4. https://api.github.com/repos/earendil-works/pi
5. https://api.github.com/repos/cursor/plugins
6. https://api.github.com/repos/NousResearch/hermes-agent
7. GitHub MCP directory listing: https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions
8. GitHub MCP file: https://github.com/earendil-works/pi/blob/main/LICENSE
9. https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/extensions.md
10. https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/sdk.md
11. https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/rpc.md
12. https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/session-format.md
13. https://raw.githubusercontent.com/NousResearch/hermes-agent/main/README.md
14. https://hermes-agent.nousresearch.com/docs/developer-guide/architecture
15. https://hermes-agent.nousresearch.com/docs/user-guide/features/cron
16. https://hermes-agent.nousresearch.com/docs/user-guide/features/memory
17. https://hermes-agent.nousresearch.com/docs/user-guide/features/skills
18. https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
19. https://raw.githubusercontent.com/cursor/plugins/main/pstack/README.md
20. GitHub MCP directory listings for `cursor/plugins/pstack/skills`, `agents`, `automations`, `.cursor-plugin`
21. https://raw.githubusercontent.com/obra/superpowers/main/README.md
22. GitHub MCP file: https://github.com/obra/superpowers/blob/main/LICENSE
23. https://raw.githubusercontent.com/mattpocock/skills/main/README.md
24. GitHub MCP file: https://github.com/mattpocock/skills/blob/main/LICENSE
25. https://raw.githubusercontent.com/agentskills/agentskills/main/README.md
26. https://agentskills.io/specification
27. https://docs.github.com/en/copilot/concepts/agents/about-agent-skills
28. https://code.claude.com/docs/en/plugins and https://code.claude.com/docs/en/skills
29. https://learn.chatgpt.com/docs/build-skills
30. https://cursor.com/docs/skills
31. https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/docs/skills.md
32. https://github.com/agentsmd/agents.md/blob/main/README.md
33. https://aaif.io/news/linux-foundation-announces-formation-of-aaif
34. https://aaif.io/ and https://aaif.io/projects/agents-md
35. GitHub MCP file: https://github.com/NousResearch/hermes-agent/blob/main/LICENSE
36. https://hermes-agent.nousresearch.com/docs/user-guide/security
37. GitHub MCP file: https://github.com/cursor/plugins/blob/main/pstack/LICENSE
38. https://api.github.com/repos/obra/superpowers
39. https://api.github.com/repos/mattpocock/skills
40. GitHub MCP directory listing: https://github.com/mattpocock/skills/tree/main/skills/engineering
41. GitHub MCP directory listing: https://github.com/mattpocock/skills/tree/main/skills/productivity
