# 04 — Communication Hub

Scope: this report covers Year96's Communication Hub: the mandatory path for human↔agent, agent↔agent, Int Comm inside the organization, and Ext Comm with outside identities. It owns Communicators, thread chats, invitations, clone participation, listener/hanger subscription mechanics, Builder flags, escalations, and enforcement of the hierarchical talk rule. It does **not** own durable execution (#05), identity/permission policy internals (#06), state/search fabric (#01), scope-effect ranking (#02), or thread memory storage (#03); it defines the communication interfaces those parts must plug into.

## TL;DR for the Year96 architect

- Build the Hub as an **append-only communication control plane**, not as another multi-agent orchestrator. Messages are CloudEvents-compatible envelopes on a durable bus; thread state and policy decisions are stored separately.
- Adopt **A2A 1.0** for cross-agent interoperability and agent discovery where external or heterogeneous agents are involved. It is now an Apache-2.0 Linux Foundation project, originally donated by Google, with Agent Cards, JSON-RPC/gRPC/HTTP bindings, streaming, async tasks, and opaque-agent boundaries [3][4][5].
- Adopt **MCP 2025-06-18** for tools/context, not for agent-to-agent communication. Its latest spec adds client features such as roots, sampling, and elicitation; governance is under LF Projects/AAIF with Apache-2.0 outbound code/specs [1][2].
- Trial **AGNTCY Directory + SLIM** for large-scale discovery, identity routing, observability, and secure low-latency agent transport. It is LF-governed, Cisco-originated, backed by Cisco/Dell/Google/Oracle/Red Hat, and explicitly interoperates with A2A and MCP [6][7][8][9].
- Use **NATS JetStream** as the first internal bus provider: permissive Apache-2.0, CNCF, simple operations, subjects map naturally to org/thread scopes, and JetStream gives persistence, replay, explicit ack, pull consumers, and at-least-once semantics [33][34]. Kafka remains a scale-out provider for very high-volume analytics/event-log use [35].
- Treat **Communicators as non-interfering facilitators** with a hard allow-list: read, classify/meta-label, invite, subscribe/unsubscribe, nudge, pause, summarize, open child thread, escalate/flag, request policy decision. They cannot edit content, choose task conclusions, approve external actions, call domain tools, or decide outputs.
- Enforce non-interference technically: communicators get a capability token with no `content.write`, no `tool.execute`, no `decision.set`, no `artifact.publish`; all their actions produce `comm.meta.*` events, never replacement messages. A verifier watches for any content delta attributable to a communicator.
- Implement the talk rule as a pre-send check owned jointly with #06: an identity may address its same level or descendants; it may address upward only if the thread has the identity's parent as an active participant, or if the action is a `flag/escalation` routed through a Communicator.
- Clone participants are first-class **attenuated identities**: `cloneOf`, purpose, expiry, budget, scope, and transcript separation. Clones can brainstorm but cannot inherit all parent permissions by default (#06).
- Listeners and hangers are subscription roles, not passive participants. Listeners receive thread events/digests without speaking; hangers define a wake condition over state/scope-effect signals from #01/#02.
- For human UX, use **AG-UI** for streaming human-agent app events and **A2UI** for declarative, non-executable generated UI. Both are permissive (AG-UI MIT; A2UI Apache-2.0) and complementary: AG-UI is interaction protocol, A2UI is safe UI payload [10][11].
- Ext Comm must be gateway-mediated: JMAP/IMAP/SMTP, Slack/Teams/Discord/Telegram/WhatsApp/Signal, web chat, and voice (LiveKit/Pipecat). Every outbound event runs identity mapping, DLP, policy, consent/approval, audit, and redaction before leaving the org (#06/#09) [29][30][31][32].
- The closest research analogue to a Communicator is not a manager agent but a **Magentic-One-style ledger/orchestrator stripped of authority**: it keeps a goal/scope/progress ledger and detects stalls, loops, and drift without selecting the answer [19]. Anthropic's 2025 research system confirms orchestrator-worker value and the cost/risk of multi-agent coordination [20].
- Avoid Redis Streams as a core substrate despite technical usefulness: Redis 8 is tri-licensed RSALv2/SSPLv1/AGPLv3, which is Excluded-license for Year96 core [36]. Matrix is good as a federated/E2EE external integration; Synapse is Apache-2.0 but maintainer situation moved toward Element, and chat semantics are not a high-throughput bus [37].

## Landscape

### Agent protocols and governance

**Model Context Protocol (MCP), latest opened spec 2025-06-18.** MCP standardizes how hosts/clients/servers expose resources, prompts, and tools over JSON-RPC 2.0, with capability negotiation, progress, cancellation, logging, and trust/safety guidance [1]. The spec is exactly the right abstraction for Year96 identities to access tools and context, but it is not sufficient as the Hub's full communication plane because it assumes host-client-server tool relationships rather than many long-lived thread participants. The lead's governance memory was directionally right but incomplete: MCP has formal LF Projects governance, Apache-2.0 outbound code/specs, CC-BY-4.0 docs, and a lead/core/maintainer model [2]; AAIF was announced Dec 2025 with MCP, goose, and AGENTS.md as founding contributions [3].

**Agent2Agent (A2A), latest spec 1.0.0.** A2A is an open standard for communication among opaque, independent agents, with Agent Cards, tasks, messages, parts, artifacts, extensions, and protocol bindings for JSON-RPC, gRPC, HTTP/REST, SSE and async push [4]. Google donated it to the Linux Foundation in June 2025; the LF announcement names AWS, Cisco, Google, Microsoft, Salesforce, SAP, and ServiceNow as supporting participants [5]. It matters because Year96 must talk to agents it does not own without exposing private memory or tools. IBM ACP did not simply disappear: IBM/BeeAI's ACP repo still exists as a separate protocol for rich agent/application/human communication [14], while A2A has become the more neutral agent-to-agent standard. Treat "ACP merged into A2A" as **not verified**; IBM teaches A2A with Google, but the ACP repository remains active enough to watch.

**AGNTCY + OASF Directory + SLIM.** AGNTCY is LF-governed infrastructure for discovery, identity, messaging, and observability among agents from different frameworks. The July 2025 LF release says it was initially open-sourced by Cisco in March 2025, has 65+ supporting companies, and interoperates with A2A and MCP [6]. Its Directory service stores OASF metadata in distributed records, routes by skills/domains/modules/locators, and is under active IETF draft development [8]. SLIM is Secure Low-Latency Interactive Messaging: data plane routing by hierarchical names, a session layer for reliable delivery, MLS E2E encryption and group membership, and a control plane for configuration/monitoring [7][9]. Year96 should not bet the first laptop version on AGNTCY, but the Hub data model should be compatible with its names, directory records, and secure transport assumptions.

**AG-UI and A2UI.** AG-UI, from the CopilotKit ecosystem, is an MIT event protocol for connecting agents to user-facing apps: ~16 event types, transports such as SSE/WebSockets/webhooks, streaming state sync, generative UI messages, frontend tools, and human-in-the-loop collaboration [10]. A2UI, from Google/community with CopilotKit contributions, is Apache-2.0 and specifies declarative UI payloads (`application/a2ui+json`) that clients render natively without executing arbitrary code; v0.9.1 is current and v1.0 candidate [11]. Use AG-UI for the session/event stream and A2UI as one allowed structured `MessagePart` type.

**Agent Client Protocol (ACP, Zed-origin).** The Agent Client Protocol standardizes communication between code editors/IDEs and coding agents, using JSON-RPC over stdio locally and HTTP/WebSocket remotely, reusing MCP JSON representations where possible and adding coding UX concepts like diffs [12][13]. It matters as an Ext Comm gateway for developer tools, not as the org-wide Hub protocol.

**NLWeb.** NLWeb, now under `nlweb-ai/NLWeb`, is MIT and proposes natural-language endpoints for websites, returning Schema.org JSON and acting as an MCP server, with A2A planned [15]. It matters for Ext Comm with websites/vendors/customers: a Year96 external identity can be a website with an NLWeb endpoint rather than a human chat account.

**ANP, agents.json, Agent Name Service.** I found community material but no sufficiently authoritative, mature, permissive spec opened in this time box. Keep these as Watch. Year96 should support A2A Agent Cards today and map future agents.json/name-service proposals into the same `ExternalIdentityResolver` interface.

### Multi-agent conversation frameworks

**Microsoft Agent Framework and AutoGen/Magentic-One.** Microsoft Agent Framework (docs updated July/Aug 2026) combines Agents, Harness Agent, workflows, integrations, MCP clients, middleware, context providers, sessions, and model provider support across .NET/Python/Go [16]. AutoGen 0.4 replaced the older 0.2 APIs with an async event-driven Core API and AgentChat; the docs explicitly state 0.2 is maintained but users should upgrade, and pyautogen package ownership changed [17]. AG2 is now the volunteer-maintained AutoGen-derived ecosystem with AG2 v1.0 and AG2 Classic split [18]. Magentic-One is important architecturally: the original implementation is deprecated, but its orchestrator is now `MagenticOneGroupChat` in AgentChat, preserving the pattern of task/progress ledgers and specialist workers [19]. For Year96, borrow ledgers and stall detection, not the authority model.

**LangGraph, OpenAI Agents SDK, Google ADK, CrewAI, CAMEL.** LangGraph's multi-agent supervisor/swarm patterns are proven production patterns, though my direct fetch of the multi-agent page redirected. OpenAI Agents SDK handoffs are explicit transfers represented as tools with optional validated inputs and dynamic enablement [21]. Google ADK is an open, deploy-anywhere agent framework with session/memory/tool-output/artifact context management, observability on Google Cloud, and production agent deployment paths [22]. CrewAI is MIT, role/crew/flow oriented, with enterprise AMP for control-plane features [23]. CAMEL targets research-scale multi-agent simulations up to 1M agents, with stateful memory and dynamic communication [24]. These matter as **Hub clients**, not as the Hub itself: the Hub must allow their teams to participate in threads while making communication, policy, and audit consistent.

**Anthropic's 2025 multi-agent research system.** Anthropic reports an orchestrator-worker research architecture where a lead agent plans and spawns parallel subagents, outperforming single-agent research by 90.2% on internal evals but using ~15× chat tokens [20]. This validates Year96's clones/subagents/hangers model for broad research, and it warns that unbounded multi-agent conversations are expensive and failure-prone without budgets, stop rules, summaries, and progress ledgers.

### Moderation/facilitation research

**Multi-agent debate and ChatEval.** Du et al.'s multiagent debate paper (May 2023) and ChatEval (Aug 2023) show that structured debate/moderator/judge roles can improve factuality/evaluation [25][26]. But those roles make or score decisions; Year96 Communicators must not. The useful part is protocol design: role-separated turns, speaker selection, explicit stopping criteria, and transcripts for later evaluation.

**CoT monitorability and monitoring agents.** OpenAI's 2025 monitorability work and the July 2025 arXiv “Chain of Thought Monitorability” paper argue that natural-language reasoning traces are a fragile oversight opportunity [27][28]. Year96 should not require private chain-of-thought exposure for all agents. Communicators should monitor observable artifacts: message metadata, tool calls, progress deltas, time since last useful state change, repeated intents, contradiction markers, unresolved questions, and declared goals. If #06/#09 allow private monitor channels, they must be tightly scoped and audited.

**Human deliberation.** The DeepMind “Habermas Machine” (Nature 2024) could not be fetched directly due site restrictions in this time box, but it remains relevant as a model for facilitation that summarizes disagreement and seeks group statements. Year96 should treat it as inspiration for late-joiner summaries and consensus maps, not as authority to resolve thread decisions.

### Clones and brainstorming

Clones are a structured form of self-consistency / mixture-of-agents / persona diversity. They improve breadth but risk diversity collapse, echo chambers, and false consensus. The Hub should create clones with explicit `cloneOf`, `personaDelta`, `budget`, `expiry`, and `independencePolicy`; hide or randomize other clones' intermediate outputs when seeking independent ideation; and compare outputs before discussion. A Communicator can invite clones and enforce budgets, but cannot choose the winning idea.

### Messaging substrate

**NATS + JetStream.** NATS is Apache-2.0/CNCF and positions itself as a secure, performant communications system for services/devices, on-prem/cloud/edge/Raspberry Pi, with 40+ clients [33]. JetStream adds streams, file storage, explicit ack consumers, pull subscriptions, replay, and durable processing [34]. Its subject hierarchy maps well to `org.<orgId>.thread.<threadId>.*` and `identity.<id>.inbox`.

**Kafka.** Apache Kafka is Apache-2.0 and remains the event-streaming heavyweight for high-volume logs, analytics, and mission-critical pipelines [35]. It is a good `MessageBusProvider` implementation when Year96 runs at cluster scale, but overkill for laptop and small org deployments.

**Redis Streams.** Redis is technically useful for queues and streams, but Redis 8 moved to RSALv2/SSPLv1/AGPLv3 tri-licensing for new code [36]. Under the user's licensing rule, Redis is Excluded-license (core). It can remain an external integration or optional deployment dependency only if legal review permits.

**Matrix / Zulip / XMPP.** Matrix is an open communication standard with federation, encryption, and VoIP; Synapse is Apache-2.0 but Matrix.org says it cannot resource Synapse maintenance and development continues at Element [37]. Conduwuit redirected to Tuwunel as the official successor in my GitHub fetch; verify Tuwunel separately before adoption. Zulip is Apache-2.0 and excellent for human topic-based threads, with topics designed to keep many asynchronous conversations organized [38][39]. These are Ext/Human UX integrations, not the core bus.

### Ext Comm gateways and human UX

**Email/JMAP/IMAP.** JMAP is the modern JSON/HTTP alternative to IMAP/SMTP-style integration, useful for mailbox sync, identities, and approvals [32]. Use JMAP where available; IMAP/SMTP remain fallbacks. Every email contact becomes an `ExternalIdentity` with contract, channel, domain, trust level, and permitted scopes.

**Slack/Teams/Discord/Telegram/WhatsApp/Signal.** Hermes Agent is MIT and explicitly supports Telegram, Discord, Slack, WhatsApp, Signal, and CLI from a single gateway process, with cross-platform continuity and scheduled automations [31]. That verifies the lead's hermes-agent gateway claim. I could not verify “OpenClaw” as a strong primary source in time; mark it Watch/unverified.

**Voice/video.** LiveKit Agents is an OSS realtime framework for voice/video/text/physical AI agents; agents become participants in LiveKit rooms, with STT-LLM-TTS pipelines, turn detection, interruptions, traces, and managed/cloud or custom deployment [29]. Pipecat is BSD-2-Clause by license fetch and should be trialed for provider-agnostic realtime voice pipelines [30].

**Human approvals.** HumanLayer's GitHub repo says the public issues repo is deprecated and points to humanlayer.com [40]. Treat as an external product/integration, not core OSS. Year96's own `ApprovalProvider` should be built at the Hub/policy layer, with HumanLayer-like adapters optional.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| MCP | Standard/spec + OSS SDKs | Tool/context/resource/prompt protocol for identities | Apache-2.0 for code/spec; docs CC-BY-4.0 [2] | LF/AAIF; 2025-06-18 spec opened [1] | Adopt |
| A2A | Standard + OSS SDKs | Cross-agent discovery, tasks, messages, Agent Cards | Apache-2.0 [5] | Linux Foundation; latest spec 1.0.0 [4] | Adopt |
| AGNTCY Directory | OSS infrastructure | Distributed agent discovery and metadata | Apache-2.0 likely; repo/license not fully fetched, docs opened | LF; Cisco/Dell/Google/Oracle/Red Hat backing [6][8] | Trial |
| SLIM | OSS transport | Secure low-latency agent messaging with MLS E2EE | Apache-2.0 likely; license not fully fetched, docs opened | AGNTCY; Rust, SDKs, CLI, active docs [7][9] | Trial |
| AG-UI | OSS protocol | Human-agent event stream, state sync, frontend tools | MIT verified by GitHub README badge [10] | CopilotKit ecosystem; active docs | Adopt |
| A2UI | OSS protocol/spec | Safe declarative generated UI payloads | Apache-2.0 verified [11] | v0.9.1 current, v1.0 candidate [11] | Trial |
| Agent Client Protocol | OSS protocol | IDE/coding-agent gateway | Apache-2.0 contribution terms [12][13] | Zed-origin; stable protocol v1 [12] | Trial |
| NLWeb | OSS protocol/tools | Website natural-language Ext Comm endpoint | MIT verified [15] | Microsoft-origin/nlweb-ai; MCP now, A2A soon [15] | Watch |
| NATS JetStream | OSS messaging | First internal bus provider | Apache-2.0 verified [33][34] | CNCF, many clients, edge-friendly [33] | Adopt |
| Apache Kafka | OSS messaging | Cluster-scale event streaming provider | Apache-2.0 verified [35] | Apache; high maturity | Trial |
| Redis Streams | OSS/source-available DB | Simple streams/queues if externally managed | RSALv2/SSPLv1/AGPLv3 tri-license [36] | Popular but license incompatible | Excluded-license |
| Matrix/Synapse | Standard + OSS server | Federated/E2EE external chat bridge | Apache-2.0 Synapse verified [37] | Matrix standard mature; Synapse maintenance moved [37] | Trial as gateway |
| Zulip | OSS product | Human topic/thread UX model | Apache-2.0 verified via LICENSE fetch; README opened [38][39] | Large OSS community; topic model strong | Trial as UX integration |
| Microsoft Agent Framework | OSS/product docs | Provider-rich agent/harness/workflow client framework | MIT (not re-verified in license file; docs only) | Microsoft docs updated 2026 [16] | Watch/Trial |
| AutoGen / Magentic-One | OSS framework/paper | Progress ledgers/orchestrator pattern | MIT (not re-verified here) | v0.4 rewrite; Magentic-One ported/deprecated old impl [17][19] | Trial ideas |
| AG2 | OSS framework | Maintained AutoGen classic/group chat lineage | Apache-2.0? (unverified in time box) | Active volunteer project [18] | Watch |
| CrewAI | OSS framework/product | Role/crew/flow patterns; agent client | MIT verified by README badge [23] | Active, commercial AMP | Watch |
| CAMEL | OSS research framework | Large-scale clone/agent simulation ideas | Apache-2.0? (unverified in time box) | Research community; up to 1M agents claimed [24] | Watch |
| LiveKit Agents | OSS/product | Realtime voice/video gateway | Apache-2.0 via license fetch | Active docs; cloud/custom deploy [29] | Trial |
| Pipecat | OSS framework | Voice/multimodal pipeline gateway | BSD-2-Clause via license fetch [30] | Daily-backed OSS | Trial |
| Hermes Agent | OSS product | Multi-chat gateway, automations, self-improving harness integration | MIT verified by README badge [31] | Nous Research; required stack mentions it | Trial |
| HumanLayer | Product/deprecated OSS repo | Human approvals design inspiration | Repo deprecated; license not useful [40] | Product continues; OSS repo deprecated | Watch |

## How I would build this part of Year96

### Architecture headline

The Communication Hub is five planes around an append-only event stream:

1. **Ingress/Egress Gateway Plane**: channel adapters for AG-UI, A2A, MCP events, ACP/IDE, email/JMAP/IMAP/SMTP, Slack/Teams/Discord/Telegram/WhatsApp/Signal, Matrix/Zulip, LiveKit/Pipecat voice, web chat, and NLWeb.
2. **Message Bus Plane**: NATS JetStream first; Kafka provider later. It carries immutable envelopes and backpressure signals.
3. **Thread Service Plane**: owns thread chat membership, roles, invitations, listeners, hangers, milestones references, and links to #03 thread memory. It does not store full semantic memory itself.
4. **Router + Policy Plane**: resolves identities, applies talk-permission, DLP, external comm policy, clone attenuation, rate limits, and required approvals from #06.
5. **Communicator Pool**: stateless/ephemeral facilitator agents assigned to threads, each with a goal/scope/progress ledger and a strictly limited meta-action API.

### Data model

```ts
type IdentityLevel = 'owner' | 'ownership' | 'duty' | 'execution' | 'external';
type ParticipantRole =
  | 'owner' | 'participant' | 'listener' | 'hanger'
  | 'clone' | 'communicator' | 'human' | 'external';

type HubEventType =
  | 'message.created' | 'message.redacted' | 'thread.created'
  | 'thread.member.invited' | 'thread.member.joined' | 'thread.member.left'
  | 'subscription.listener.added' | 'subscription.hanger.added'
  | 'comm.summary.created' | 'comm.nudge.created' | 'comm.pause.requested'
  | 'comm.escalation.raised' | 'builder.flag.raised'
  | 'policy.decision.recorded' | 'gateway.delivery.requested'
  | 'gateway.delivery.succeeded' | 'gateway.delivery.failed';

interface HubEnvelope<T = unknown> {
  specversion: '1.0';
  id: string;
  type: `year96.comm.${HubEventType}`;
  source: string;
  subject: string;
  time: string;
  datacontenttype: 'application/json';
  dataschema?: string;
  data: T;
  year96: {
    orgId: string;
    threadId: string;
    chatId: string;
    milestoneId?: string;
    causationId?: string;
    correlationId: string;
    sender: IdentityRef;
    addressedTo: IdentityRef[];
    visibility: 'int' | 'ext' | 'mixed';
    policyDecisionId?: string;
    scopeVectorRef?: string;
    stateRefs: string[];
    memoryRefs: string[];
    redactionClass: 'none' | 'pii' | 'secret' | 'contract' | 'privileged';
  };
}

interface IdentityRef {
  id: string;
  orgId?: string;
  level: IdentityLevel;
  parentId?: string;
  external?: { channel: string; address: string; contractId?: string };
}

interface Participant {
  identity: IdentityRef;
  roles: ParticipantRole[];
  cloneOf?: string;
  clonePolicy?: { purpose: string; expiresAt: string; budget: Budget; independence: 'blind' | 'visible' };
  joinedAt: string;
  canSpeak: boolean;
  canReceive: boolean;
}

interface ThreadChat {
  id: string;
  threadId: string;
  goal: string;
  scopeStatement: string;
  participants: Participant[];
  communicatorId: string;
  status: 'active' | 'paused' | 'dormant' | 'awaiting-approval';
}
```

### Provider interfaces

```ts
export interface MessageBusProvider {
  publish<T>(envelope: HubEnvelope<T>, opts?: { idempotencyKey?: string }): Promise<PublishAck>;
  subscribe(filter: BusFilter, handler: (e: HubEnvelope) => Promise<void>): Promise<Subscription>;
  replay(filter: BusFilter, from: Cursor, to?: Cursor): AsyncIterable<HubEnvelope>;
  deadLetter(envelope: HubEnvelope, reason: string): Promise<void>;
}

export interface ThreadRouter {
  route(input: InboundMessage): Promise<RoutePlan>;
  resolveRecipients(threadId: string, addressing: Addressing): Promise<IdentityRef[]>;
  ensureCommunicator(threadId: string): Promise<IdentityRef>;
}

export interface CommunicatorPolicy {
  canSend(sender: IdentityRef, recipients: IdentityRef[], thread: ThreadChat): Promise<PolicyDecision>;
  canInvite(actor: IdentityRef, invitee: IdentityRef, thread: ThreadChat): Promise<PolicyDecision>;
  canEgress(message: HubEnvelope, gateway: GatewayRef): Promise<PolicyDecision>;
  canCommunicatorAct(action: CommunicatorAction, thread: ThreadChat): Promise<PolicyDecision>;
}

export interface GatewayProvider {
  readonly channel: string;
  ingest(raw: unknown): Promise<InboundMessage[]>;
  deliver(message: HubEnvelope): Promise<DeliveryReceipt>;
  mapExternalIdentity(raw: unknown): Promise<IdentityRef>;
  capabilities(): GatewayCapabilities;
}

export interface PresenceProvider {
  getPresence(identityId: string): Promise<Presence>;
  watch(identityIds: string[]): AsyncIterable<PresenceEvent>;
}

export interface NotificationRanker {
  rank(identity: IdentityRef, candidates: NotificationCandidate[]): Promise<RankedNotification[]>;
}
```

### Talk-permission algorithm

```ts
function mayTalk(sender: IdentityRef, recipient: IdentityRef, thread: ThreadChat): boolean {
  if (sender.orgId !== recipient.orgId) return false; // Ext handled by egress policy
  if (sameLevel(sender, recipient)) return true;
  if (isAncestor(sender, recipient)) return true;     // sender talks downward
  const parentInThread = sender.parentId && thread.participants
    .some(p => p.identity.id === sender.parentId && p.canReceive);
  if (parentInThread && isAncestor(recipient, sender)) return true;
  return false;
}
```

Exceptions are not hidden in code. A Builder flag upward is not a normal message; it is `builder.flag.raised`, routed to the nearest allowed Communicator/parent policy endpoint. External communication always goes through `canEgress` and contact/contract mapping.

### Communicator spec

A Communicator has one durable ledger per thread:

- `goalLedger`: original goal, current accepted scope, non-goals, success signals, unresolved questions.
- `participantLedger`: participants, listeners, hangers, clones, external identities, why each is present.
- `progressLedger`: last useful state change, open loops, stalled branches, flags, milestone refs.
- `policyLedger`: talk-rule denials, DLP blocks, approvals, external sends, clone budgets.

Allowed actions:

```ts
type CommunicatorAction =
  | { kind: 'invite'; identity: IdentityRef; reason: string }
  | { kind: 'addListener'; identity: IdentityRef; filter: SubscriptionFilter }
  | { kind: 'addHanger'; identity: IdentityRef; wakeCondition: WakeCondition }
  | { kind: 'nudge'; target: IdentityRef; reason: string }
  | { kind: 'pauseThread'; reason: string }
  | { kind: 'summarize'; audience: 'late-joiner' | 'digest' | 'handoff' }
  | { kind: 'openChildThread'; goal: string; scope: string }
  | { kind: 'escalate'; to: IdentityRef; reason: string; evidenceRefs: string[] }
  | { kind: 'requestPolicyDecision'; question: string; evidenceRefs: string[] };
```

State machine: `assigned → read_context → establish_goal_scope → monitor → {summarize | invite | nudge | pause | escalate} → monitor`, with side states `awaiting_policy`, `awaiting_human`, `dormant`, `handoff_to_new_communicator`. Communicator replacement is safe because all state is in ledgers and events, not in agent memory.

Non-interference is enforced by: (1) capability tokens, (2) event schema separation (`comm.*` cannot mutate `message.created`), (3) no domain tools in the communicator tool registry, (4) runtime policy denying content decisions, (5) audit/verifier that compares artifacts before/after communicator actions, (6) adversarial tests where communicators are prompted to decide the answer, secretly edit outputs, or bias a clone.

### Typical flow

1. Human sends “start Facebook ads for my business” through AG-UI/web chat.
2. Gateway creates `message.created`; router creates/locates a thread in #03 and assigns a Communicator.
3. Policy validates the human can create the thread and invite the Ownership identity (#06).
4. Communicator summarizes goal/scope, invites relevant Ownership/Duty identities, and adds #02 hanger subscriptions for ad-platform/account state changes.
5. Duty identities talk to same-level/below execution Builders. Builders raise flags through `builder.flag.raised` if blocked.
6. A vendor must be contacted. Gateway maps vendor email/Slack as `ExternalIdentity`; DLP/policy/approval runs; outbound message gets an audit trail.
7. Listener identities receive digests ranked by #02 notification ranker. Hangers wake only when #01 state changes and #02 says scope effect crosses a threshold.
8. Late joiner gets a communicator summary with source message refs, not a rewritten transcript.

### Scaling path

Laptop: SQLite/Postgres for thread metadata, NATS single node with JetStream, one communicator worker, local web/CLI gateway. Team server: clustered NATS, Postgres, object store for attachments, multiple communicator workers, gateway workers per channel. Enterprise cluster: Kafka provider for analytics/event retention, NATS/SLIM for low-latency commands, AGNTCY Directory for cross-org discovery, per-tenant encryption keys, regional gateway shards, replayable audit store. Global: federated Hubs connected by A2A/AGNTCY, with explicit cross-org contracts and legal/policy edges.

### Testing and proofs (70% rule)

- Unit: envelope schema, routing, talk-rule matrix, clone attenuation, listener/hanger filters, communicator action allow-list.
- Property tests: random org trees prove no upward talk unless parent participant; random external identities prove no egress without policy decision.
- Integration: NATS provider replay/idempotency/backpressure; gateway adapters with fake Slack/email/voice; A2A Agent Card discovery and message exchange.
- Mocked E2E: human starts thread, clones brainstorm independently, listener digest sent, hanger wakes, Builder flag escalates, external email blocked by DLP.
- Agentic verifiers: adversarial Communicator prompts; verify it never writes conclusions, edits content, or selects winners.
- Observability: every message has correlation/causation IDs, OpenTelemetry spans, before/after state captures, policy decision refs, and redaction class.
- Replay: deterministic simulation from event log must reconstruct participants, permissions, summaries, escalations, and notifications.

## What is still unsolved (late 2026)

- **Non-interference is sociotechnical.** Capability gates can prevent direct edits, but subtle framing/bias in summaries and invitations can influence decisions. Year96 needs summary-bias evals, balanced evidence presentation, and maybe “summary diff” reviews by #09.
- **Agent identity standards are fragmented.** A2A Agent Cards, AGNTCY OASF records, NLWeb endpoints, MCP server metadata, and future agents.json/name services overlap. Year96 needs a canonical identity resolver with lossless adapters.
- **E2E encryption vs. moderation.** SLIM/Matrix-style E2EE conflicts with DLP, policy, and communicator oversight. The likely solution is endpoint-side policy enforcement with auditable encrypted metadata, but this is not solved generally.
- **Thread explosion UX.** Thousands of never-closing threads need ranking, digests, topic clustering, and interruption budgets. The Hub can emit signals, but #02 must solve relevance and #03 must solve bounded memory.
- **Clone diversity collapse.** Clones often converge because they share model priors and prompt context. Need independence controls, model/provider diversity, blinded ideation, and evaluators that reward novelty without rewarding nonsense.
- **External platform terms.** WhatsApp/Signal/Slack/Teams automation terms can change; some channels disallow unattended bots or scraping. Gateway providers need compliance metadata, not just APIs.
- **Exactly-once semantics.** Human communication cannot tolerate duplicate outbound emails/DMs. The bus can be at-least-once; gateways need idempotency keys, outbox tables, and delivery reconciliation.
- **Communicator staffing.** Always-on Communicators for every thread may be expensive. Year96 needs dormant-mode ledgers plus event-triggered activation, and maybe cheaper rule-based monitors before LLM facilitators.

## Sources

[1] https://modelcontextprotocol.io/specification/2025-06-18
[2] https://modelcontextprotocol.io/community/governance
[3] https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation
[4] https://a2a-protocol.org/latest/specification/
[5] https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents
[6] https://www.linuxfoundation.org/press/linux-foundation-welcomes-the-agntcy-project-to-standardize-open-multi-agent-system-infrastructure-and-break-down-ai-agent-silos
[7] https://github.com/agntcy/slim
[8] https://dir.agntcy.org/latest/dir/dir-overview/
[9] https://slim.agntcy.org/latest/slim/slim-howto/
[10] https://github.com/ag-ui-protocol/ag-ui
[11] https://a2ui.org/
[12] https://agentclientprotocol.com/overview/introduction
[13] https://github.com/agentclientprotocol/agent-client-protocol
[14] https://github.com/i-am-bee/acp
[15] https://github.com/nlweb-ai/NLWeb
[16] https://learn.microsoft.com/en-us/agent-framework/overview/
[17] https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/migration-guide.html
[18] https://github.com/ag2ai/ag2
[19] https://github.com/microsoft/autogen/tree/main/python/packages/autogen-magentic-one
[20] https://www.anthropic.com/engineering/multi-agent-research-system
[21] https://openai.github.io/openai-agents-python/handoffs/
[22] https://adk.dev/
[23] https://github.com/crewAIInc/crewAI
[24] https://github.com/camel-ai/camel
[25] https://arxiv.org/abs/2305.14325
[26] https://arxiv.org/abs/2308.07201
[27] https://openai.com/index/evaluating-chain-of-thought-monitorability/
[28] https://arxiv.org/abs/2507.11473
[29] https://docs.livekit.io/agents/
[30] https://github.com/pipecat-ai/pipecat
[31] https://github.com/NousResearch/hermes-agent
[32] https://jmap.io/spec.html
[33] https://github.com/nats-io/nats-server
[34] https://docs.nats.io/nats-concepts/jetstream
[35] https://github.com/apache/kafka
[36] https://github.com/redis/redis
[37] https://github.com/matrix-org/synapse
[38] https://github.com/zulip/zulip
[39] https://zulip.com/help/introduction-to-topics
[40] https://github.com/humanlayer/humanlayer
