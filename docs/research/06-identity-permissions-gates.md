# 06 — Identities, Permissions, Gates, Governance

Scope: this report owns Year96 identity and authority: how humans, agents, clones, services, external contacts, and organizations receive credentials; how every action is authorized; how clone/builder permissions are attenuated; how programmatic gates defend state, tools, messages, external communications, and money; and how audit/governance satisfy a 2026–2027 compliance environment. I assume Year96's other subsystems are delivered by #01 state-fabric, #02 scope-effect, #03 thread-memory, #04 communication-hub, #05 runtime, #09 verification, and #11 harness-stack; this design supplies their trust boundary.

## TL;DR for the Year96 architect

- Treat **Identity** as the root object: any entity with >=1 permission is registered, credentialed, risk-tiered, and audit-visible. Humans, agents, clones, services, external contacts, organizations, tools, budgets, gateways, state namespaces, threads, and environments all become typed authorization resources.
- Use **SPIFFE/SPIRE-style workload identity** for runtime-issued, short-lived cryptographic identity. SPIFFE's SVID model is proven for heterogeneous workloads [1]; the 2026 IETF AI-agent auth draft explicitly says agent credentials should be dynamically provisioned, posture-bound, short-lived, and hidden from the LLM [2].
- Use **OAuth 2.1 + RFC 8693 token exchange** for user/agent on-behalf-of flows, and follow the **MCP authorization spec** for HTTP MCP resources: MCP servers act as OAuth resource servers and must publish protected resource metadata when auth is enabled [3].
- Put **OpenFGA** as the default ReBAC graph for Year96's social/org/thread/resource rules. Use **Cedar** or **OPA** for contextual ABAC decisions that need attributes, risk, time windows, taint labels, geography, model class, or regulatory tier. Do not choose only one: graph authorization and policy evaluation solve different halves.
- Capability attenuation for clones/builders should use **Biscuit or Macaroons-like caveated tokens**, not copied OAuth tokens. A clone's capability is derived from parent authority, narrows by resource/action/budget/time/thread, and is non-escalating by construction.
- Encode the Year96 **talk rule** programmatically: an identity may talk to same layer or below; upward/cross-parent talk is denied unless the target's parent participates in that thread. This belongs in ReBAC, not in prompts.
- Encode the Ownership/Duty/Builder rule in ReBAC: Ownership CRUDs its own Duties and may discuss CRUD of Ownerships; Duties may discuss CRUD sibling Duties and request Builders; Builders execute only delegated goals and cannot mint siblings or modify Ownership.
- For prompt injection, use architectural controls, not prompt-only controls. The 2025 CaMeL paper argues for separating control/data flow and enforcing capabilities [13]; Willison's lethal-trifecta model says private data + untrusted content + external communication is the dangerous combination [14]. Year96 gates must prevent that combination except under approval.
- Place gates at **all levels**: identity issuance, harness session, task plan, tool call, state read/write, message send, ext-comm, money, code change, environment egress, and post-action verification. All gates emit audit events before and after decisions.
- For money, model AP2/ACP/Visa/Mastercard mandates as **budget permissions**. AP2 is Apache-2.0 sample/protocol work with signed payment mandates [17]; OpenAI's commerce docs use one-time delegated payment requests with max charge and expiry [18]; Visa TAP requires time-bound, merchant/purpose-specific signatures [19]; Mastercard Agent Pay uses registered/authenticated agents and agentic tokens [20].
- Audit must be both queryable and tamper-evident: hot OLTP/warehouse for investigations, plus hash-chained append log, periodic Merkle roots, and transparency anchoring with Rekor/Tessera-style logs. Rekor v1 is maintenance; Rekor v2 is moving to Trillian/Tessera tile logs [9][10].
- Regulation pushes toward inventory, risk assessment, human oversight, logging, cybersecurity, and documentation. The EU AI Act's high-risk obligations start applying from 2027 for many systems [24]; NIST AI RMF is being revised in 2026 and has a critical-infrastructure profile in progress [25]; ISO/IEC 42001 is the management-system wrapper [26].

## Landscape

### Agent and workload identity

**SPIFFE/SPIRE** is the strongest OSS base for non-human runtime identity. SPIFFE defines short-lived cryptographic identity documents, SVIDs, delivered through a Workload API for mutual authentication across heterogeneous platforms [1]. SPIRE is the reference implementation and is marked production phase in its repository; the license is Apache-2.0 verified from the repository LICENSE. Why it matters: Year96 will run thousands of agents, clones, builders, MCP tools, sandboxes, and services across laptops and clusters; static API keys cannot survive that scale.

**IETF WIMSE / AI-agent authentication drafts** are the standards path. The AI-agent authentication draft says credentials should be provisioned and rotated dynamically, bound to posture signals such as software integrity and runtime context, and that the LLM must not access agent or tool credentials [2]. Why it matters: this matches Year96's rule that the agent harness, not the model text, owns authority.

**MCP Authorization, 2025-06-18** defines OAuth-based authorization for HTTP MCP transports. It says protected MCP servers act as OAuth resource servers, clients act as OAuth clients, authorization servers issue access tokens, and protected resource metadata is mandatory when supported [3]. Why it matters: MCP tools are a major execution surface; Year96 must never call tools over unauthenticated ad-hoc channels.

**Microsoft Entra Agent ID, Amazon Bedrock AgentCore Identity, Auth0 AI Identity, Okta Cross App Access.** Microsoft documents agent-specific authorization and blocks many high-privilege roles for agents [4]. AWS AgentCore Identity is built for AI agents and automated workloads, with workload identity, third-party service credentials, JWT authorizers, consent, and audit trails [5]. Auth0 documents AI agent login, OAuth user-delegated API calls, Token Vault, CIBA approval, and FGA-backed RAG authorization [6]. Okta Cross App Access was not verified from the guessed URL during this time-box, but it remains a likely external integration, not core. Why it matters: enterprise tenants will already have identity providers; Year96 should federate, not replace them.

### Authorization engines and capabilities

**OpenFGA** is a Zanzibar-inspired, high-performance fine-grained authorization server [7]. Apache-2.0 license verified from GitHub. Why it matters: Year96 rules are mostly relationship rules: parent, owner, participant, sibling, listener, budget holder, environment maintainer, thread member.

**SpiceDB** is another mature Zanzibar-style engine focused on scale [8]. Apache-2.0 license verified. Why it matters: if Year96 needs global-scale consistency semantics and mature commercial support around Authzed, SpiceDB is a strong trial candidate. OpenFGA gets the default recommendation because of its simple Auth0 ecosystem fit and readable model language.

**Cedar** is an authorization language from AWS with schema validation and automated reasoning support [11]. Apache-2.0 license verified. Why it matters: risk, context, and proof obligations fit Cedar better than a pure relation graph.

**OPA/Rego** is a CNCF-graduated, general-purpose policy engine [12]. Apache-2.0 license verified. Why it matters: OPA is the ecosystem default for Kubernetes, gateways, CI, and infrastructure policy. Year96 can use it at environment and gateway layers even if product permissions sit in OpenFGA.

**Biscuit** is an attenuable authorization token. Its repository states tokens can be created, attenuated, inspected, and authorized; implementations exist across languages, with the project seeking more cryptographic review [21]. Apache-2.0 license verified. Why it matters: clones/builders need delegated rights that can only shrink.

### Prompt-injection and agent runtime security

**CaMeL, “Defeating Prompt Injections by Design” (2025)** proposes design-level defenses that separate data and control flows and enforce capabilities [13]. Why it matters: Year96 should treat untrusted content as tainted state and prevent it from issuing control instructions.

**The lethal trifecta (Simon Willison, June 2025)** describes the dangerous combination of private-data access, untrusted content exposure, and external communication [14]. Why it matters: this is a simple gate predicate for Year96 risk tiering.

**OWASP GenAI / LLM Top 10** remains the baseline taxonomy for LLM application risks [15], while **MITRE ATLAS** is the adversary-technique knowledge base [16]. Why it matters: Year96 red-team suites and #09 verification should map gates to known threat categories.

**Guardrails AI, NeMo Guardrails, Purple Llama / Llama Guard / Prompt Guard.** Guardrails AI is moving validators to standard PyPI packages and discontinuing hosted remote inference in August 2026 [22]. NeMo Guardrails is Apache-2.0 and actively positioned as a rails framework [23]. Meta Purple Llama includes Prompt Guard, Llama Guard, Code Shield, and CyberSec Eval; the repository says evals/benchmarks are MIT but models use the Llama Community License [27]. Why it matters: use these as pluggable guardrail providers, not as the sole security boundary.

**MCP gateways.** Docker MCP Gateway centralizes MCP server lifecycle, credentials, access control, isolated containers, restricted network/resource usage, logging, and call tracing [28]. Agentgateway supports MCP authorization rules over list/call tool methods and can hide unauthorized tools from list responses [29]. Why it matters: Year96 should put every tool behind a gateway with per-call authz and audit.

### Agentic payments and money

**Google AP2** is an Apache-2.0 repository with samples and protocol types for Agent Payments Protocol, demonstrated in September 2025 [17]. Why it matters: it introduces signed agent payment mandates as portable permissions.

**OpenAI Agentic Commerce / Delegated Payment Spec** requires product feeds, checkout sessions, and delegated payment requests. It states that OpenAI prepares a one-time delegated payment request with maximum chargeable amount and expiry, and Stripe Shared Payment Token is the first compatible implementation [18]. Why it matters: Year96 budgets should issue one-time, merchant-scoped, amount-capped payment capabilities.

**Visa Trusted Agent Protocol** says merchants need to know whether an AI agent is trusted, acting for an authenticated user, and carrying valid purchase instructions [19]. Its capability overview says signatures are merchant-specific, purpose-specific, time-bound, and non-replayable [30]. Why it matters: external commerce identity is converging on signed intent artifacts.

**Mastercard Agent Pay** launched in April 2025, with agentic tokens, trusted agent registration/authentication, Microsoft/IBM/acquirer partnerships, and pre/during/post transaction transparency [20]. Why it matters: Year96's “money” gate is not just a payment API; it is a mandate registry plus spend verifier.

### Governance and compliance control planes

**Microsoft Agent 365** is positioned as a control plane for AI agents with registry, access control, visualization, interoperability, and security; it can quarantine unsanctioned agents and enforce least privilege via Entra [31]. **ServiceNow AI Control Tower** tracks MCP servers as governed assets, requires approval before activation, preserves audit history, and exposes lifecycle/risk/control metadata via APIs [32]. Why it matters: Year96 needs the same primitives internally: registry, purpose, risk class, approvals, kill switch, offboarding, and evidence export.

**EU AI Act, NIST AI RMF, ISO/IEC 42001.** The EU page lists prohibited practices effective February 2025, an AI Omnibus addition effective December 2026, and high-risk obligations from December 2027 including risk mitigation, logging, documentation, human oversight, robustness, cybersecurity, and accuracy [24]. NIST AI RMF 1.0 is voluntary but being revised in 2026 [25]. ISO/IEC 42001 supplies the AI management-system lifecycle [26]. Why it matters: build evidence by default; do not bolt compliance on later.

### Tamper-evident audit

**Sigstore Rekor** provides an immutable, tamper-resistant transparency log for signed metadata; the repo says v1 is in maintenance and v2 will use tile-based logs and Trillian/Tessera [9]. **Tessera** is the successor library for tile-based transparency logs, introduced in 2024 and considered production-ready since beta [10]. **in-toto** provides signed supply-chain layouts and links proving that authorized functionaries performed steps as planned [33]. Why it matters: Year96 audit should link identity/action/reason/evidence and make deletion or rewriting detectable.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| SPIFFE/SPIRE | OSS/standard | Workload IDs, SVIDs, short-lived mTLS/JWT identity | Apache-2.0 verified | Production-phase repo; stars not verified due GitHub API rate limit | Adopt |
| IETF WIMSE / AI-agent auth | Standard draft | Posture-bound dynamic agent credentials and credential isolation from LLM | IETF document | Active draft -03, 2026 | Adopt |
| MCP Authorization | Standard | OAuth 2.1 authorization for MCP HTTP resources | Spec | 2025-06-18 spec | Adopt |
| OpenFGA | OSS | ReBAC/Zanzibar graph for identities, threads, resources | Apache-2.0 verified | Active repo; stars not verified | Adopt |
| SpiceDB | OSS/product | Alternative Zanzibar engine with strong scale posture | Apache-2.0 verified | Active Authzed-backed repo; stars not verified | Trial |
| Cedar | OSS/language | ABAC policies with schema validation and automated reasoning | Apache-2.0 verified | Active AWS-origin repo; stars not verified | Adopt for ABAC |
| OPA/Rego | OSS/CNCF | Infra/gateway/environment policy engine | Apache-2.0 verified | CNCF graduated; stars not verified | Adopt |
| Biscuit | OSS/capability | Attenuable clone/builder capability tokens | Apache-2.0 verified | Multi-language; seeks crypto audit | Trial |
| Docker MCP Gateway | OSS/product | Containerized MCP lifecycle, secrets, access control, tracing | License not verified in time-box | Docker Desktop 4.59+ ecosystem | Trial |
| agentgateway | OSS/product | Per-MCP-method authorization with JWT/CEL rules | License not verified | Active docs; maturity unverified | Watch/Trial |
| Guardrails AI | OSS/product | Input/output validators and structured output guards | License not verified | Hosted inference discontinued Aug 2026 | Trial, avoid hosted dependency |
| NeMo Guardrails | OSS | Rails framework for LLM apps | Apache-2.0 visible in repo badge | NVIDIA-backed | Trial |
| Meta Purple Llama | OSS/models | Prompt Guard, Llama Guard, Code Shield, evals | Mixed: MIT for evals, Llama Community for models | Meta-backed | Trial, not core-model license |
| Google AP2 | OSS/protocol | Signed payment mandates for agent commerce | Apache-2.0 visible in repo badge | 2025 samples; SDK package pending | Trial |
| OpenAI ACP / Delegated Payment | Product/spec | One-time delegated payment requests, max charge, expiry | Spec/product, license not verified | OpenAI/Stripe ecosystem | External integration |
| Visa TAP | Product/spec | Trusted agent signatures: merchant/purpose/time-bound | Product/spec | Visa-backed | External integration |
| Mastercard Agent Pay | Product/spec | Agentic tokens and registered trusted agents | Product/spec | Mastercard-backed | External integration |
| Rekor | OSS | Transparency log for signed metadata | Apache-2.0 verified | v1 maintenance; v2 planned | Trial/Watch v2 |
| Tessera | OSS/library | Tile-based transparency logs | License not verified in time-box | Production-ready since beta per repo | Adopt for new tlog experiments |
| in-toto | OSS | Signed task attestations and supply-chain provenance | License not verified in time-box | Established CNCF-adjacent supply-chain tool | Adopt with #09 |
| Microsoft Agent 365 / Entra Agent ID | Product | Enterprise registry, access controls, agent identity | Product | Microsoft-backed, GA claim in docs/blog | External integration |
| AWS AgentCore Identity | Product | Workload identity and credential providers for agents | Product | AWS-backed | External integration |
| Auth0 AI Identity/FGA | Product/OSS mix | User auth, delegated APIs, Token Vault, CIBA, RAG FGA | Product; OpenFGA Apache-2.0 | Auth0/Okta-backed | External integration |

## How I would build this part of Year96

### Identity model

Every subject and controlled object has a canonical URN, immutable creation event, lifecycle state, owning organization, risk tier, data residency tags, and credential bindings.

```ts
type IdentityKind = 'Human'|'Agent'|'Clone'|'Service'|'ExternalContact'|'Organization'|'Tool'|'Gateway';
type Layer = 'organization'|'ownership'|'duty'|'builder'|'service'|'external';
type Lifecycle = 'proposed'|'active'|'suspended'|'revoked'|'deleted-shadow';

interface Identity {
  id: `idn_${string}`;
  kind: IdentityKind;
  layer: Layer;
  orgId: string;
  parentId?: string;          // clone parent, duty parent, builder requester
  cloneOf?: string;
  attenuation?: CapabilityConstraint[];
  humanAccountId?: string;
  agentRole?: 'Owner'|'Ownership'|'Duty'|'Builder'|'Communicator'|'Verifier'|'ToolBroker';
  lifecycle: Lifecycle;
  riskTier: 'low'|'medium'|'high'|'critical';
  credentials: CredentialBinding[];
  createdForThreadId?: string;
  purpose: string;
}

interface CredentialBinding {
  type: 'spiffe-svid'|'oauth-client'|'oidc-subject'|'vc'|'api-key-legacy'|'external-contact-proof';
  subject: string;
  issuer: string;
  audience?: string[];
  expiresAt?: string;
  postureEvidence?: string[];
}

interface ActingForLink {
  actor: string;              // runtime identity doing the action
  principal: string;          // human/org/agent authority source
  mechanism: 'oauth-token-exchange'|'capability'|'approval'|'service-account'|'external-mandate';
  constraints: CapabilityConstraint[];
  previous?: ActingForLink;
}
```

Issuance flow: (1) registry creates proposed identity with purpose and parent; (2) posture verifier checks code digest, harness version from #11, environment from #05, and approval if needed; (3) SPIRE/WIMSE provider issues short-lived SVID/JWT; (4) authorization graph tuples are written; (5) first audit root is anchored. Revocation is graph-deny first, credential expiry second, and emergency kill-switch third.

### Provider interfaces

```ts
interface IdentityProvider {
  register(input: RegisterIdentity): Promise<Identity>;
  activate(id: string, evidence: Evidence[]): Promise<Identity>;
  suspend(id: string, reason: string): Promise<void>;
  resolve(idOrCredential: string): Promise<Identity>;
}

interface CredentialProvider {
  issue(identityId: string, audience: string[], posture: Evidence[]): Promise<Credential>;
  exchange(subjectToken: Credential, requested: DelegationRequest): Promise<Credential>;
  revoke(credentialId: string, reason: string): Promise<void>;
}

interface AuthorizationProvider {
  check(req: AuthzRequest): Promise<AuthzDecision>;
  expand(resource: string, relation: string): Promise<string[]>;
  writeTuples(changes: TupleChange[], cause: AuditRef): Promise<void>;
}

interface PolicyGateProvider {
  evaluate(action: ActionRequest, ctx: GateContext): Promise<GateDecision>;
}

interface GuardrailProvider {
  classify(input: GuardrailInput): Promise<TaintFinding[]>;
  transform?(input: GuardrailInput): Promise<GuardrailOutput>;
}

interface AuditLogProvider {
  append(event: AuditEvent): Promise<AuditRef>;
  anchor(range: AuditRange): Promise<TransparencyProof>;
  query(q: AuditQuery, viewer: Identity): Promise<AuditEvent[]>;
}

interface ApprovalProvider {
  request(gate: GateDecision, approvers: string[]): Promise<ApprovalTicket>;
  resolve(ticketId: string, decision: 'approve'|'deny', rationale: string): Promise<void>;
}
```

### Concrete ReBAC schema

OpenFGA-style model sketch:

```fga
model
  schema 1.1

type identity
  relations
    define parent: [identity]
    define clone_of: [identity]
    define same_level_peer: [identity]
    define member_of: [organization]
    define can_clone: owner or parent->can_delegate
    define can_delegate: [identity]

type organization
  relations
    define owner: [identity]
    define member: [identity]
    define admin: owner
    define can_create_ownership: owner
    define can_register_external: owner or admin

type ownership
  relations
    define org: [organization]
    define owner: [identity]
    define participant: [identity]
    define child_duty: [duty]
    define can_discuss_crud: owner or org->owner
    define can_create_duty: owner
    define can_read: owner or participant or org->admin
    define can_update: owner

type duty
  relations
    define parent_ownership: [ownership]
    define owner: [identity]
    define sibling: [duty]
    define participant: [identity]
    define requested_builder: [builder]
    define can_read: owner or participant or parent_ownership->owner
    define can_update: owner
    define can_discuss_crud: owner or sibling->owner or parent_ownership->owner
    define can_request_builder: owner or parent_ownership->owner

type builder
  relations
    define requested_by: [duty]
    define executor: [identity]
    define can_execute: executor and requested_by->can_request_builder
    define can_write_artifact: can_execute
    define can_request_more_permission: executor

type thread
  relations
    define org: [organization]
    define participant: [identity]
    define listener: [identity]
    define parent_participant: [identity]
    define subject_ownership: [ownership]
    define subject_duty: [duty]
    define can_read: participant or listener or org->admin
    define can_post: participant
    define has_parent_of_target: parent_participant

type message_target
  relations
    define identity: [identity]
    define same_or_lower: [identity]
    define parent: [identity]
    define can_talk: same_or_lower or parent from thread_has_parent

type tool
  relations
    define owner: [identity]
    define gateway: [gateway]
    define invoker: [identity]
    define can_invoke: invoker and gateway->approved

type state_namespace
  relations
    define owner: [identity]
    define reader: [identity]
    define writer: [identity]
    define can_read: owner or reader
    define can_write: owner or writer

type budget
  relations
    define owner: [identity]
    define spender: [identity]
    define approver: [identity]
    define can_reserve: spender
    define can_spend: spender and owner
    define can_approve_over_limit: approver

type gateway
  relations
    define approved: [identity]
    define admin: [identity]
    define can_publish_tool: admin
```

OpenFGA cannot express numeric budget ceilings, time windows, taint labels, or “thread_has_parent” joins alone. Those are evaluated by Cedar/OPA with the graph decision as input. The action gate asks: graph says actor can invoke tool? policy says request is within amount/time/taint/risk? approval says required human approved? only then issue a one-use capability.

### Capability attenuation for clones and builders

Clone creation requires `identity.can_clone` and a purpose. The parent never shares its long-lived credential. Instead, CredentialProvider performs token exchange and CapabilityProvider mints a Biscuit-style token with caveats:

```ts
interface CapabilityConstraint {
  resource: string;
  actions: string[];
  threadId?: string;
  maxBudgetMinor?: number;
  notBefore?: string;
  expiresAt: string;
  taintPolicy?: 'no-untrusted-to-ext'|'summaries-only'|'no-private-data';
  maxDelegationDepth: number;
  approvalTicket?: string;
}
```

A clone may further attenuate for sub-clones only if `maxDelegationDepth > 0`, and the child constraints must be a subset. Builders get the narrowest possible capability: goal, tool list, state namespaces, budget reservation, egress allowlist, time limit, and verification hooks.

### Gate pipeline

1. **Pre-action gate**: authenticate credential; resolve acting-for chain; check ReBAC; evaluate Cedar/OPA policy; classify inputs for taint/prompt injection; check budget/time/geography/data-residency; compute risk tier; request human approval if needed.
2. **In-action gate**: run tool in #05 sandbox; inject credentials only into gateway sidecar, never model context; restrict filesystem, network, environment, clipboard, browser profile, and MCP tool list; monitor egress and token streams; pause on policy drift.
3. **Post-action gate**: verify claimed effects with #09; diff state in #01; attach screenshots/logs/receipts; update thread meta-memory in #03 with who/why; settle budget; anchor audit.

“Gates at all levels” maps as follows: identity harness gate (credential issuance), task gate (plan approval), environment gate (sandbox/egress), code gate (tests/provenance/in-toto), messaging gate (talk rule and taint), state gate (read/write auth), external comm gate (recipient trust and exfiltration), money gate (mandate/budget), promotion gate (self-improvement release by #08), and governance gate (inventory/lifecycle by control plane).

Risk-tiered approval: low = automatic with audit; medium = asynchronous notification and reversible action; high = CIBA/human approval before execution; critical = two-person approval plus verifier simulation and spending/egress hold.

### Audit design

AuditEvent includes actor, actingFor chain, action, resource, request hash, policy version, graph tuple version, guardrail findings, approval ticket, before/after state refs, verification refs, thread id, milestone id, rationale, and privacy labels. Store full events in encrypted queryable storage with row/column policy; store hashes and signatures in an append-only log; periodically write Merkle roots to Tessera/Rekor-like transparency log. Privacy: redact payloads by default, keep hashes for integrity, and support GDPR/AI Act evidence export without exposing unrelated personal data.

### Scaling and testing

Laptop: SQLite/Postgres OpenFGA, local OPA, local SPIRE, Docker MCP Gateway, file-backed audit. Team cluster: HA OpenFGA/SpiceDB, SPIRE federation, OPA bundles, gateway fleet, Kafka/Pulsar audit stream, warehouse, Tessera anchoring. Enterprise: federated IdPs, regional state partitions, hardware-backed attestation, approval integrations, SIEM export.

Proof plan: unit-test every relation; model-check policy examples; property-test attenuation subset rules; replay known prompt-injection corpora; simulate lethal-trifecta cases; run E2E flows for human desire -> communicator -> ownership -> duty -> builder -> tool -> audit; red-team external comm and payments; verify audit inclusion proofs; require #09 verifier sign-off before policy promotion.

## What is still unsolved (late 2026)

- No universal, mature standard yet unifies agent discovery, identity, delegation, tool authorization, and payment mandates. WIMSE/OAuth/MCP/AP2/TAP/ACP overlap but do not fully compose.
- Prompt-injection defense remains fundamentally architectural. Guardrail classifiers help, but cannot prove arbitrary natural-language content is harmless.
- ReBAC engines do not naturally express dynamic risk, budgets, taint, and temporal constraints; Year96 must compose graph + ABAC + capabilities without creating inconsistent decisions.
- Clone identity creates hard accountability questions: a clone is not exactly the parent, but audit and liability still roll up through the acting-for chain.
- Tamper-evident audit conflicts with privacy deletion and data minimization. The likely answer is redactable encrypted payloads plus immutable hashes, but operational UX is hard.
- Agentic commerce standards are proprietary-adjacent and fast-moving. Use them through provider interfaces and keep Year96's internal budget mandate format independent.
- Human approval can become the bottleneck. Year96 needs risk-calibrated approvals, reversible actions, escrow, simulations, and learned policies without normalizing unsafe autonomy.

## Sources

1. https://spiffe.io/docs/latest/spiffe-about/overview/
2. https://www.ietf.org/archive/id/draft-klrc-aiagent-auth-03.html
3. https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
4. https://learn.microsoft.com/en-us/entra/agent-id/authorization-agent-id
5. https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html
6. https://auth0.com/ai/docs
7. https://github.com/openfga/openfga
8. https://github.com/authzed/spicedb
9. https://github.com/sigstore/rekor
10. https://github.com/transparency-dev/trillian-tessera
11. https://github.com/cedar-policy/cedar
12. https://github.com/open-policy-agent/opa
13. https://arxiv.org/abs/2503.18813
14. https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
15. https://genai.owasp.org/llm-top-10/
16. https://atlas.mitre.org/
17. https://github.com/google-agentic-commerce/AP2
18. https://developers.openai.com/commerce/guides/key-concepts
19. https://developer.visa.com/use-cases/trusted-agent-protocol
20. https://investor.mastercard.com/investor-news/investor-news-details/2025/Mastercard-Unveils-Agent-Pay-Pioneering-Agentic-Payments-Technology-to-Power-Commerce-in-the-Age-of-AI/default.aspx
21. https://github.com/biscuit-auth/biscuit
22. https://github.com/guardrails-ai/guardrails
23. https://github.com/NVIDIA/NeMo-Guardrails
24. https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
25. https://www.nist.gov/itl/ai-risk-management-framework
26. https://www.iso.org/standard/81230.html
27. https://github.com/meta-llama/PurpleLlama
28. https://docs.docker.com/ai/mcp-catalog-and-toolkit/mcp-gateway/
29. https://agentgateway.dev/docs/standalone/latest/documentation/configuration/security/mcp-authz/
30. https://developer.visa.com/capabilities/trusted-agent-protocol/overview
31. https://www.microsoft.com/en-us/copilot/blog/2025/11/18/microsoft-agent-365-the-control-plane-for-ai-agents/
32. https://www.servicenow.com/community/ai-control-tower-articles/ai-control-tower-what-s-new-in-the-june-2026-release/ta-p/3561445
33. https://github.com/in-toto/in-toto
