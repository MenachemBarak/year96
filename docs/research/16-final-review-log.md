# 16 — Final review log (multi-model rounds)

This log records the final review of [YEAR96_TECHNICAL_ARCHITECTURE.md](../YEAR96_TECHNICAL_ARCHITECTURE.md). The goal comes from the Vision's definition of done.
**Once built, the system must run by itself, work, heal and evolve by itself. The human only gives input from any chat app, sees every thread, creates their own thread with a
Communicator, and watches the whole architecture live.**

**Protocol (at the user's direction):** *every aspect is reviewed by every model*, because each model catches different things. The rounds repeat until no reviewer reports a blocker.

| Aspect | What it checks |
|---|---|
| **A1 Flows and buildability** | Traces every flow in the catalog (F1–F29 in v0.4, F1–F32 from v0.5, F1–F34 from v0.7) end to end: identities, permissions, commands and events, state machines, gates, proof, failure paths. Checks that the contracts are specific enough to build and that the phases really reach the DoD |
| **A2 Autonomy, safety and resilience** | 30 unattended days: self-healing (including the healer's own dependencies), self-evolution without drift or oscillation, safety invariants under autonomy, scale |
| **A3 Human experience, fidelity and DoD** | Chat apps, threads, the Communicator and the Observatory; fidelity to the concepts in INTRO; whether the DoD is measurable and sufficient |

Models: `gpt-6-sol` · `gpt-6-astra` · `claude-opus-5.5`.

## Round 1 (v0.3)

Round 1 used one aspect per model. The user corrected this protocol after the round.

| Reviewer | Model | Aspect | Verdict |
|---|---|---|---|
| Sol | gpt-6-sol | A1 | **NO**: 8 blockers |
| Astra | gpt-6-astra | A2 | **NO**: 7 blockers |
| Opus | claude-opus-5.5 | A3 | **NO**: 5 blockers |

A separate reviewer's 10 scale findings were folded in as well: cross-shard revocation, the migration fence, lease fencing, the control-plane single point of failure, cross-cell threads,
global ids and time, hot threads, the GPU estimate, what `sim` can prove, and the waiver.

### Consolidated themes and how v0.4 resolves them

| # | Theme (raised by) | Resolution in v0.4 | Where |
|---|---|---|---|
| T1 | Genesis needs authority, a thread and budgets before any exist; nothing proves the prover (Sol, Astra, Opus) | A signed bootstrap manifest runs as a one-time transaction that creates the org, genesis thread #0, the root budget pool, the system identities and the seed ReBAC tuples. A model-free bootstrap verifier provides an external trust anchor. The first human uses a passkey. Bootstrap authority is permanently disabled afterward | §7.0 |
| T2 | Nobody can create the Ownership a new desire needs (Sol, Opus) | A Portfolio Ownership proposes it, the human confirms (or a standing mandate applies), and the kernel creates it with the human as acting-for principal. New flow F23 | §6.6, §7.0 |
| T3 | The human's level and read rights are undecided; "every thread" was capped by the attention budget (Sol, Opus) | The human is the level-0 principal and reads every org thread. The capped Inbox is separate from the uncapped All Threads tree. "Active" is defined. The Q1 default is justified and all readings are tested | §6.1, §6.12, §5, §12 |
| T4 | The human's own chat channels were modelled as Ext Comm (Opus) | Principal-channel adapters (Int Comm) are separate from outsider gateways. Each channel has an assurance level, with passkey step-up for high and critical actions and mandates | §4, §6.5, §6.12 |
| T5 | The Communicator couldn't converse, yet it "accepted" scopes (Opus) | Allowed speech acts: acknowledge, clarify, propose a ledger entry for confirmation, quote the state with citations, routing receipts, digests. Substantive answers come from invited identities | §6.5, §6.12, §7.0 |
| T6 | The human couldn't halt, redirect or grant access (Opus) | `halt / pause / cancel / amend` run at top priority and cascade, with compensation. Access is granted through consent, OAuth and step-up, producing a scoped, revocable secret. New flows F24 and F25 | §6.12 |
| T7 | Unattended waits and resource starvation (Astra, Opus, Sol) | Every wait has an owner, a deadline and a deny-and-park default, plus standing mandates, a one-human two-person substitute and a human-silence conservative mode. Capacity for proofs, healing and sensors is reserved inside the ceiling, and breakers have a reset policy | §6.1 |
| T8 | Authorization can expire between commit and dispatch (Astra, Sol) | A single-use dispatch token revalidates the payload hash, the epochs, the deadline and the reservation immediately before sending | §6.1 |
| T9 | The budget ceiling across shards has no protocol (Astra) | An org budget pool aggregate hands out conserved escrow slices on the spenders' shards | §6.1 |
| T10 | RPO ≤ 5 min contradicts "no data loss"; restores can resurrect deleted data (Astra, Sol, Opus) | Synchronous standbys in another zone give RPO = 0 within D3's failure model. Region loss is declared outside it, with a reconciliation sweep. Tombstones are restored first, restores run under an execution fence, and a canary cell exists from Phase 3 | §6.9, §6.13, §11 |
| T11 | Self-healing depends on what it heals (Astra, Sol, Opus) | A model-free substrate supervisor runs signed infrastructure runbooks, with independent probes, its own audit and out-of-cell peers. Paging goes out-of-band. The system has a no-LLM degraded mode | §6.13 |
| T12 | Evolution can drift slowly, or oscillate with healing (Astra, Sol, Opus) | A stabilization window, pre-registered thresholds and holdouts, rollback plus quarantine, and a single rollout controller that is the only writer of desired state | §6.11, §11 D4 |
| T13 | No canonical command/event registry or state machines (Sol) | A contract registry generated from LinkML, plus state machines as data for Task, Effect, Approval, Variant, Incident and Thread | §5 |
| T14 | The payment gate needed a proof that only exists after paying (Sol) | A pre-spend authorization proof and a post-spend settlement proof | §6.10 |
| T15 | `unknown` effects could stay in limbo (Sol) | Connectors declare their reconciliation capability, and `needs-reconciliation` has a deadline and an escalation | §6.1 |
| T16 | Deletion missed some copies and backups (Sol, Opus, Astra) | Every copy is encrypted under subject keys, tombstones survive restores, and the proof lists every store | §6.2 |
| T17 | The Observatory's zoom levels, live step and time travel weren't buildable as written (Opus, Sol) | Linked logical and physical trees, a presence stream from pi hooks, as-of snapshots plus replay, a p95 event-to-pixel target for Z0–Z5, and per-viewer telemetry authorization | §6.12 |
| T18 | The DoD was gameable or incomplete (Opus, Sol) | A soak workload, a varied desire suite including Unreal, sandbox-appropriate ad predicates, planted variants for D4, and a new D8 for mental-model fidelity, Ownership gains and usability with real humans | §11 |
| T19 | Execution authority contradicted attenuation (Astra) | Capabilities have `exercise` and `delegate` modes, and kernel-minted Builder grants are anchored in the human's mandate | §3, §5, §6.1, §6.8 |
| T20 | "Ownership at all levels" was missing (Opus) | `why.challenge` and `how.proposal` events from any identity to its requester | §6.8 |
| T21 | Inbound Ext Comm routing was undefined (Opus) | Reply keys recorded in the effect ledger, then the sender's identity, then the cascade, then triage. New flow F27 | §6.5 |
| T22 | CDC and SaaS admission was ambiguous (Sol) | Connectors are service identities, authoritative only for their registered namespaces | §6.2.1 |
| T23 | A poisoned fact left tainted derivatives (Sol) | PROV-lineage taint, then re-derivation and re-verification | §6.13 |
| T24 | Non-Builder timeouts had no flag (Sol) | A generic `flag.raised` with its source; replanning belongs to the thread owner | §6.1 |
| T25 | Hosted model retirement had no owner (Astra) | Provider-lifecycle tracking, tested fallbacks and a safe pause. The kernel CVE path is a signed release plus human approval, with pre-approved mitigations meanwhile | §6.11 |
| T26 | Real chat-app constraints (Opus) | A focus thread for flat chats, receipts with undo, WhatsApp templates, native fallbacks, outbound modes or a relay behind NAT, and a labelled routing oracle | §6.12 |
| T27 | Appendix C inaccuracies (Opus) | The row 7 waiver text is fixed, 🟡 is in the legend, and the V rows point at concrete mechanisms | App. C |
| T28 | The mental-model feedback had no contract (Sol) | `prediction.recorded`, `choice.observed` → `fidelity.scored` | §7.3 |
| T29 | Flows missing from the catalog (Opus) | F23 new Ownership · F24 steering · F25 access grants · F26 waits · F27 inbound Ext Comm · F28 channel lifecycle · F29 asking about the state | §7.0 |

## Round 2 (v0.4): every aspect × every model

Each of the three models reviewed each of the three aspects: nine independent reviews. The three Round-1 reviewers each continued with their own aspect, and six new reviewers were started for the other cells.
Each reviewer checked every Round-1 resolution (T1–T29) against v0.4 and then looked for new blockers.

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 6 new blockers, 6 partial resolutions | **NO**: 4 new blockers; T6, T7/T13, T10 and T22 partial | **NO**: 4 new blockers; T1, T6 and T21 partial and blocking |
| **A2 Autonomy, safety and resilience** | **NO**: 3 new blockers, 12 partial resolutions | **NO**: 4 new blockers; T10 and T12 partial | **NO**: 6 new blockers |
| **A3 Human experience, fidelity and DoD** | **NO**: 1 new blocker (L6 fidelity to INTRO); T3, T4, T5, T10, T11/T17 and T18 partial and blocking | **NO**: 1 new blocker; T6, T7, T10 and T18 partial and blocking; T17 and T26 partial | **NO**: 3 new blockers, 7 partial resolutions |

About 115 findings in total. Many overlapped across cells, which is the point of running every model on every aspect: the strongest themes were raised independently by up to eight of the nine reviews.
They consolidate into 34 themes.

### Consolidated themes and how v0.5 resolves them

| # | Theme (raised by) | Resolution in v0.5 | Where |
|---|---|---|---|
| U1 | `solo` couldn't meet D3's "no data loss"; the canary cell had one node (8 of 9 reviews) | The failure model is declared **per profile**. `solo` adds off-box WAL, an escrowed root key, an external dead-man's service and synchronous off-box commits for effect-dispatch markers, so machine loss never loses an effect record. `solo+replica` loses nothing. `cluster` and `fleet` use Patroni + etcd quorums with fencing (RPO = 0). The Phase 3 canary has two nodes | §6.9, §9.2, §11 D3, Phase 3 |
| U2 | The human's own chat channels are an exfiltration path (Opus A3, Sol A1, Sol A3) | Principal channels are external by transport: DLP and taint per channel, no model-authored URLs from tainted contexts, unfurls off, pinned recipients, and group chats treated as Ext Comm. D6 now tests zero-click exfiltration | §6.5, §6.12, §11 D6 |
| U3 | Steering had no paused state, epoch or receipts (7 reviews) | A **control epoch** is checked at spawn and at dispatch; `paused` / `resume`; voided, compensated and irreversible effects are listed on a receipt; connectors declare compensation; each steering intent has an assurance floor | §5, §6.1, §6.12 |
| U4 | Access grants didn't reach a running Builder (6 reviews) | An `access.grant` transaction binds the delegate capability, the ReBAC relation and the secret; `capability.added` reaches the running Builder, which resumes; covers preflight, refresh, `grant.broken`, `human.task` and non-OAuth providers | §6.12, §7.1 |
| U5 | Waits held resources and conservative mode was too blunt (6 reviews) | Parking frees reservations and freezes deadlines; `resume` re-entry revalidates; question answers are typed; conservative mode blocks only new action classes, mandates continue, risk-reducing effects are always allowed, and leaving it needs a passkey; must-deliver items bypass the Inbox cap | §5, §6.1, §6.3 |
| U6 | Continuing liabilities (subscriptions, running campaigns) had no model (4 reviews) | A **Commitment** aggregate reserves rate × horizon. Autonomous launch requires an enforceable cap. Commitments are reconciled daily, and stopping one is always allowed. New flow F30 | §5, §6.1, §7.0, §7.1 |
| U7 | Duplicate dispatch after failover or retry (5 reviews) | One **EffectDispatcher**; a `dispatching{tokenId}` marker before sending; the gateway validates the token at consumption and fails closed; the dispatch lease TTL is shorter than failover; per-target ordering by `conflictKey` | §5, §6.1 |
| U8 | Failover and the healer's own recovery (6 reviews) | Quorum + fencing (Patroni + etcd), supervisor recovery leases, an emergency trust-boundary exception, a runbook-executor role and a model-free incident proof profile | §6.9, §6.10, §6.13 |
| U9 | Enforcement points were scattered, and hermes could escape (Opus A2, Sol A2) | A **signed TCB manifest** lists every enforcement point; hermes runs a locked profile with a conformance test; D6 tests an escape | §6.1, §6.8, §11 D6 |
| U10 | Upgrades and rollbacks could corrupt state (5 reviews) | Expand/contract schemas, downcasters and a rollback replay test; Temporal worker versioning; migration moves only drained workflows, with an abort path and signed forwarding routes; a `solo` GitOps reconciler | §6.2.2, §6.9, §6.11, §9.5 |
| U11 | Several humans (4 reviews) | `humanOwners` per Ownership, one accountable principal, a sponsor per intent, conservative-wins conflicts, and private personal threads and mental models. New risk 17 | §6.1, §12 |
| U12 | Losing the only passkey or channel (4 reviews) | At least two authenticators, an M-of-N recovery kit, time-delayed ledgered recovery, a stable HTTPS origin through a relay in `solo`, and at least two channels | §7.0, F28 |
| U13 | Genesis could be half-done or hijacked (4 reviews) | Reproducible builds + Rekor + a pinned signer; idempotent, resumable genesis; a pending level-0 principal with a one-time enrolment capability; bootstrap authority destroyed after enrolment; genesis moves to Phase 0 | §7.0, §11 Phase 0 |
| U14 | State machines were incomplete (5 reviews) | Verifier reject, tie-break, attempt cap and abandonment; blocked and parked exits; Effect `dispatching`, `voided` and `indeterminate`; new machines for Approval, Commitment, Variant, Incident, Ownership, grants, pairings, clones and migrations; registry naming; commands declare risk, deadline and shard affinity | §5 |
| U15 | Orphaned threads and waits (3 reviews) | Every thread and wait has a live owner. Flags route deterministically owner → parent → human; retiring an Ownership hands everything over in one transaction. New flow F31 | §5, §6.1, §6.6 |
| U16 | Critical events could be dropped by attention caps (Astra A1) | Must-deliver routing for subscriptions, hangers, dependencies and control events | §6.3 |
| U17 | Atomic invariants across shards (Sol A2) | Shard affinity for atomic invariants, otherwise a pending saga | §6.2 |
| U18 | No control when every model is down; stale presence (3 reviews) | Deterministic no-LLM commands from every channel and the Observatory (new flow F32); presence bound to leases with heartbeat expiry and resync | §6.12, §6.13 |
| U19 | The Communicator's speech acts could steer (Opus A3, Sol A3) | A `comm.speak` capability; proposals are extractive; the utterance must confirm an explicit scope; quotes are filtered by the asker's rights, with completeness cues | §6.5 |
| U20 | Inbound Ext Comm could write state directly (Opus A1, Sol A3) | An inbound message is a membrane claim, then a reply-router accept, and it stays tainted; ambiguous matches are held | §6.5 |
| U21 | The DoD was still gameable (5 reviews) | A channel × assurance × failure matrix including voice; delivery deadlines and permitted parking; degraded-mode scoring and catch-up; hidden soak generators; pre-registered D8 thresholds and sample sizes, a real-human panel and prediction coverage; per-profile D3; the bootstrap verifier re-checks every proof | §11 |
| U22 | "L6 is human-only" contradicted INTRO (Sol A3) | L6 changes outside the TCB may promote themselves through the full pipeline; only the TCB needs a human. New question Q8 | §6.11, §7.6, §12 |
| U23 | A kernel CVE would wait on a human (Sol A2, Opus A2) | An opt-in, pre-authorized emergency security track (Q4) | §6.11, §12 |
| U24 | Escrow wasn't partition-safe (3 reviews) | A spender-generation fence, reclaiming only the acknowledged remainder, budget periods with refill, healing slices on every shard, and mandate aggregates | §6.1 |
| U25 | The time slider can't be exact after crypto-shredding (Sol A3) | "Exact up to permanent redactions" | §6.2, §6.12 |
| U26 | Deletion gaps (3 reviews) | Per-artifact keys for multi-subject data, lineage re-derivation, no personal data in L4 training, and third-party chat copies listed as a residual | §6.2 |
| U27 | Lineage had no standard (Opus A1) | The context manifest is the PROV provenance record | §6.2 |
| U28 | The phase order didn't match dependencies (Sol A1, Opus A1) | Genesis and the bootstrap verifier move into Phase 0; F14 moves to Phase 3 and F20 to Phase 5; every flow F1–F32 is assigned to exactly one phase | §11 |
| U29 | Verifiers could weaken themselves (Opus A1) | Verification-plane changes are attested by the bootstrap verifier and eval-of-evals; gates and proof schemas are in the TCB | §6.10 |
| U30 | Proof requirements differed by kind of work (Opus A1) | ProofPolicy templates per kind of work | §6.10 |
| U31 | Snapshots and envelopes were impractical (Opus A1, Sol A3) | Tiered snapshots (versions, subject cut, world); the kernel fills the envelope for internal commands | §5, §6.1 |
| U32 | Some roles had no level (Opus A1) | Every role has a level and a parent; kernel-routed messages are exempt from the talk rule | §6.1 |
| U33 | Variants couldn't be evaluated against external systems (Opus A1) | A variant evaluation contract with connector simulators | §6.11 |
| U34 | Important smaller items (many) | Autonomy demotion, approval-fatigue detection and cumulative risk windows; wave rollouts from a canary cell; storage headroom; expiry forecasting; holdout query budgets; revocation liveness; a futility rule; proof capacity in the capacity model; durable schema quarantine with replay; forwarding routes; an answer deadline for `why.challenge`; scoped undo; org-graph edges; the clone state machine; cell provisioning and abort; a substantive definition of "active"; a telemetry query gateway and extra views; the Q2 default (thoughts private, audited access); other execution agents run inside pi; hosted-model drift and family diversity; multi-variant attribution; deduplicating the system's own outbound messages | throughout |

## Round 3 (v0.5): verification

The same nine reviewers (every model × every aspect) checked their own Round-2 findings against v0.5, looked for regressions, and reported new blockers only if a flow in F1–F32 became unbuildable or unsafe, or a DoD statement became unreachable or unprovable.

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 5 blockers | **NO**: 1 blocker | **NO**: 1 blocker |
| **A2 Autonomy, safety and resilience** | **NO**: 5 blockers | **YES** | **NO**: 1 blocker |
| **A3 Human experience, fidelity and DoD** | **NO**: 4 blockers | **NO**: 2 blockers | **NO**: 1 blocker |

Almost every Round-2 finding was confirmed resolved. The remaining blockers were mostly **contradictions that the v0.5 edits themselves introduced**, and they converged strongly: seven of the nine reviews found the same steering conflict independently.

### Consolidated themes and how v0.6 resolves them

| # | Theme (raised by) | Resolution in v0.6 | Where |
|---|---|---|---|
| V1 | The halt fence blocked the receipts and stops it promised; `halt` was defined four ways; `resume` was spoofable and ambiguous; `cancel` closed threads (7 of 9 reviews) | One **normative steering table** (pause, halt, stop, cancel, amend, `control.resume`) with its effect on work, effects and commitments, and an assurance floor per command. A **control lane** (receipts, status, approvals, reconciliation reads and spend-halting operations) always dispatches under the current epoch, while the work lane needs `running`. `task.resume` and `control.resume` are separate commands. `cancel` ends work, never the thread. "Stopping is cheap, starting is guarded" | §5, §6.1, §6.12, F24, D5, D6 |
| V2 | The "always authorized" risk-reducing exemption had no limits (Opus A2) | Only operations that the connector contract declares **spend-halting and non-destructive** (pause, suspend, cap to zero, reduce) are exempt, and only the kernel mints them. Destructive operations and compensations follow the approval ladder, and the human's `cancel` counts as the approval for its compensations. Connector contracts are in the TCB manifest | §5, §6.1, §6.12, D6 |
| V3 | The campaign's worst case was reserved after it launched (Sol A1, Sol A2, Opus A3, Astra A2) | A **provisional Commitment** and its worst-case escrow commit together with the launching effect, before dispatch. Observe-back makes it `active`, and a failed launch releases it | §5, §6.1, §7.1, F30 |
| V4 | F31 promised a cross-shard transaction and could strand liabilities without credentials (Sol A1, Sol A2, Opus A1) | A fenced, abortable **handover saga**. Authority moves before liability, the successor is the Portfolio rather than the human, and the retiring Ownership stays responsible until every shard acknowledges | §5, §6.6, F31 |
| V5 | `access.grant` claimed atomicity across three services (Sol A2, Sol A1, Sol A3) | A **grant saga**: staged secret, one ledger commit, an activation barrier at the ReBAC consistency token, then `capability.added`, with compensation on failure | §5, §6.12, F25 |
| V6 | Plain `solo` can't heal the loss of its only machine (Astra A3, Sol A1, Sol A2, Opus A2) | Declared honestly as an **assisted restore**: one command plus the recovery kit, proven by a drill and excluded from autonomous D3 and D1's intervention count. `solo+replica` with a witness heals machine loss autonomously. Key destruction commits synchronously off-box, and a restore re-issues pairings and grants | §6.9, §9.2, §11 D1, D3 |
| V7 | A two-node canary can't host the three-voter quorum, and cells arrived only in Phase 5 (Sol A3, Opus A1, Opus A2) | The canary spans **three failure domains** (two data nodes plus an etcd witness) on the production Patroni protocol, and is pulled forward into Phase 3 | §6.13, §11 Phase 3, D3 |
| V8 | Illegal or missing state transitions (Sol A3, Opus A1) | `dispatching → dispatched \| denied \| unknown`; `already-satisfied`; `resolved-by-human`; Commitment `provisional` and `released`; Ownership `rejected`; attempt caps survive parking; bisection can release a quarantined variant; `dispatching` is in the freeze inventory | §5, §6.2 |
| V9 | F32 over voice with no models; chat apps had no assurance class (Sol A3, Opus A3) | A model-free **keypad (DTMF) menu** for status, halt, pause and stop. Consumer chat apps are low assurance, and Slack or Teams behind SSO is medium | §6.5, §6.12, F32 |
| V10 | Multi-human talk and visibility (Sol A1, Sol A3, Astra A3) | For an Ownership the parent relation is its `humanOwners` set. All Threads shows each human what they may read. Default levels are defined for every role | §6.1, §6.12, D5 |
| V11 | Stale text and smaller gaps (many) | Flag routing, the commit-path envelope, hermes scheduling (4 places), the revocation-completion rule and isolated-shard fencing, the §9.8 assertions, tenet 2 and the §4 label, F4/F25/F28 rows, parking keeps encumbered holds, placement leases anchored in the control plane, the forwarding route replicated before the swap, an untrusted `solo` relay (TLS passthrough), pairing codes from passkey sessions, a signed incident-to-runbook table for no-LLM healing, TCB-manifested policy and risk data, expiring emergency overrides, a system-discovered improvement in D4, fidelity on real humans with an independent denominator in D8, flat-app disambiguation, and exact drill-down despite sampling | throughout |

## Round 4 (v0.6): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 1 blocker | **NO**: 1 blocker | **YES** |
| **A2 Autonomy, safety and resilience** | **NO**: 4 blockers | **NO**: 2 blockers | **YES** |
| **A3 Human experience, fidelity and DoD** | **NO**: 3 blockers | **NO**: 2 blockers | **YES** |

Every Round-3 blocker was confirmed resolved. What remained was narrower: interactions *between* the new v0.6 mechanisms. Two were found independently by five reviews each.

### Consolidated themes and how v0.7 resolves them

| # | Theme (raised by) | Resolution in v0.7 | Where |
|---|---|---|---|
| W1 | A stop could be undone by an older write already in flight, because control-lane stops skipped ordering (Astra A1, A2, A3, Sol A2, Opus A2) | Reads and messages still skip ordering, but a stop gets a **stop fence**: older undispatched writes on the target fail their tokens, the stop is reasserted after each in-flight write resolves (or uses a provider conditional write), and `stopped` needs observe-back after all of them. Until then the receipt says `stopping` and the reserve is held | §5, §6.1, §6.12, D5 |
| W2 | `amend` and `cancel` could lift a passkey-set halt at medium assurance (Astra A2, A3, Sol A2, Opus A1, Opus A3) | **Only `control.resume` leaves `paused` or `halted`.** `cancel` and `amend` keep an existing hold, a cancel's approved compensations still dispatch, and `task.resume` inside a held thread leaves the task frozen. Control applies from every non-terminal Task state | §5, §6.12, F24 |
| W3 | A halt missed launches still in flight (Sol A1, Sol A2, Opus A1, Opus A2) | A halt or stop on a `provisional` Commitment is a **pending stop** that fires the moment the launch is identified, with the reserve held until then | §5, §6.1, §6.12 |
| W4 | "No effect record lost" in plain `solo` was stronger than the design (Sol A2, Sol A3, Astra A3) | Narrowed and made provable: no record of any effect that may have left the machine (everything that reached `dispatching`), with a notice to the human allowed to repeat once | §6.9, D3, Phase 3 |
| W5 | The canary's two data nodes didn't match production's three (Sol A3, Sol A1) | The canary now has the production topology: three data nodes in three failure domains, each also an etcd member | §6.13, D3, Phase 3 |
| W6 | A commitment could launch without any usable way to stop it (Sol A3) | Autonomous launch needs a provider cap, a declared spend-halting operation and a grant covering the whole horizon. A `grant.broken` warning triggers the stop while the grant still works | §6.1, F30 |
| W7 | A revocation could be reported complete while an isolated shard still held a live lease (Sol A2) | An isolated shard counts as fenced only once its current lease has expired; until then the revocation stays pending and visible | §6.1 |
| W8 | Anti-duplicate dispatch pauses also blocked stops and notices (Opus A2, Opus A1) | Idempotent control-lane effects go out with an asynchronous marker during those pauses. A successor cell or the control plane may stop a lost cell's commitments. Placement leases are 15 minutes, so a takeover fits the one-hour RTO | §6.1, §6.9, §9.5 |
| W9 | Smaller items (many) | Authenticated voice (callback or PIN, no content read aloud, rate-limited halts); a sticky `cancel` (goal withdrawn, mandates suspended, action class demoted); `Variant.released`; an Incident `degraded-safe` state; overrides renewed rather than reverted while their incident is open; the grant barrier waits for the secret's activation too; stop-only authority for a retiring Ownership; explicit scope confirms a new thread; phase split for commitment stops and runbook healing | throughout |

### New owner input: the Q&A

Between Rounds 4 and 5 the owner added `YEAR96_Q&A.md`, which answers four design questions. v0.7 folds it in:

| Q&A answer | How v0.7 designs it | Where |
|---|---|---|
| A Duty talks with its executions, and reviews, monitors, converses with and corrects them, like a human with a harness | Live supervision: read-only views of work in progress, guidance through pi's `steer` / `followUp` delivered as Temporal signals, `amend` for goal changes, conversational replicas, supervision by exception at scale. New flow F33 | §6.7, §6.8 |
| An Ownership is a harness that keeps capturing the human's mental model | The Ownership is a running harness whose standing loop captures the mental model and directs its Duties | §6.6, §6.8 |
| Identities replicate, for example to answer the human while busy with a Duty | Per-thread **replicas** sharing the identity's state through optimistic concurrency; a Builder has one executing replica plus read-only forks. New flow F34 | §5, §6.9, §9.6 |
| A harness is an agentic structure, from one specialized agent to hundreds | `HarnessSpec` (single, team or swarm), the `member-launcher` extension routing every member spawn through the kernel, and a D2 scenario with a 100-member Duty | Tenet 14, §2, §5, §6.8, D2 |

## Round 5 (v0.7): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 1 blocker | **NO**: 1 blocker | **YES** |
| **A2 Autonomy, safety and resilience** | **NO**: 1 blocker | **NO**: 1 blocker | **NO**: 1 blocker |
| **A3 Human experience, fidelity and DoD** | **NO**: 2 blockers | **NO**: 1 blocker | **YES** |

Every Round-4 blocker was confirmed resolved, and every model judged the Q&A integration faithful and buildable. Five of the nine reviews independently found the same new gap.

### Consolidated themes and how v0.8 resolves them

| # | Theme (raised by) | Resolution in v0.8 | Where |
|---|---|---|---|
| Y1 | In `solo`, a stop sent with an asynchronous marker during a receiver outage could lose its record on machine loss, which D3 forbade (Sol A1, Sol A2, Astra A1, A2, A3) | Declared honestly. Business effects keep zero record loss. Control-lane records may be lost, so an assisted `solo` restore pauses every live Commitment and asks the human to resume each one. The restore drill covers receiver outage, then a stop, then machine loss | §6.1, §6.9, D3, Phase 3 |
| Y2 | A Duty's pause, cancel and amend had no contract, because steering was defined for humans only (Sol A3; Opus A1, A2, A3; Astra A1, A3) | **Agent steering** inside the agent's own subtree, authorized by ReBAC. An agent lifts only holds it or its ancestors set, never a human hold. An agent's cancel carries no human-override effects. `halt` stays with humans and the kernel. Guidance is bound to its spec version and epoch, and signals reach pi through an ordered, acknowledged relay | §5, §6.7, §6.8, §6.12, F33 |
| Y3 | Replicas could carry one human's private context or untrusted content into another thread (Sol A3; Opus A1, A2, A3; Sol A2) | Sibling pushes go through the ContextAssembler under the receiving thread's read rights, with labels and taint kept. A replica slot is reserved for the human. Identity state is fine-grained, with a re-plan cap. Builder forks are kernel-enforced read-only and charged to the asker | §6.9, F34, D6 |
| Y4 | A hold bound only its own subtree, so a supervising Duty or a sibling replica could restart halted work (Opus A2) | **A hold binds goals and targets.** The kernel refuses new BuilderRequests, launches and work-lane writes whose `why`, `conflictKey` or Commitment belongs to the held subtree, from any thread or replica. A human halt suspends matching standing mandates | §6.12, D6 |
| Y5 | Smaller items (many) | §9.5 now matches the 15-minute lease and the continuing control lane. Every *agent* identity is a harness. Harness members are child identities (INTRO). D2's 100-member team is a Builder, and a Duty runs as a team | §2, §3, §6.8, §9.5, D2 |

### New owner input: the Q&A's fifth answer

The owner added a fifth answer: the reader of the Q&A is part of the state, so the owner may upgrade it, control it, or replace it with a representative Ownership.
v0.8 designs this in. Tenet 10 now covers outside agents, and §5 adds an `external-agent` identity kind. New §6.14 gives every outside agent a versioned harness record, upgrades it as an evaluated variant, steers it like any identity, and replaces it with a representative Ownership through the handover saga.
New flow F35 proves it in Phase 4, where a Year96 Engineering Ownership takes over building Year96. Today's harness record is the repository's `AGENTS.md`.

## Round 6 (v0.8): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 1 blocker | **YES** | **YES** |
| **A2 Autonomy, safety and resilience** | **NO**: 2 blockers | **YES** | **YES** |
| **A3 Human experience, fidelity and DoD** | **NO**: 2 blockers | **YES** | **YES** |

Every Round-5 blocker was confirmed resolved. Six of the nine reviews passed. The three blocking reviews and five of the passing ones raised the same two gaps, both in the new §6.14.

### Consolidated themes and how v0.9 resolves them

| # | Theme (raised by) | Resolution in v0.9 | Where |
|---|---|---|---|
| Z1 | Registering an outside agent doesn't control it. It keeps its own credentials, so a pause or revocation can't stop it (Sol A1, A2, A3; Opus A1, A2, A3; Astra A2, A3) | Control is enforced where writes land. A **managed** agent acts only through short-lived credentials that Year96 issues and revokes, lands writes on protected branches that require Year96's proof check, calls tools and models through the gateways, and attests its harness at session start. An **unmanaged** agent is recorded but Year96 never claims to control it, and today's builder is unmanaged. F35's proof keeps the old session running and checks denial at every boundary. D6 gains the case | §6.14, F35, D6, `AGENTS.md` |
| Z2 | The handover saga starts from an Ownership, not an outside agent (Sol A2, A3; Opus A1) | Outside agents get their own lifecycle and fenced takeover saga. The saga pauses the agent, drains its sessions while the sponsor keeps stop authority, maps its `exercise` grants to `delegate` rights for the successor's Duties, carries holds over and verifies that old credentials are dead before retiring it | §6.14 |
| Z3 | Smaller items (many) | The `external-agent` level and parent. Outside reads treated as leaving the org. Builders sealed from hidden evals. The corrected rule for lifting agent holds (an ancestor's hold binds, and no agent steers its own task). The cancel-compensation exception. The `solo` restore relaunch wording, now covering provisional launches. `AGENTS.md` wording in §6.8 and §9.9. An explicit allowlist for Builder forks | §6.1, §6.8, §6.9, §6.12, §6.14, §9.9, D3 |

## Round 7 (v0.9): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **NO**: 2 blockers | **YES** | **YES** |
| **A2 Autonomy, safety and resilience** | **NO**: 2 blockers | **YES** | **YES** |
| **A3 Human experience, fidelity and DoD** | **NO**: 2 blockers | **YES** | **YES** |

Six of nine passed. The three blocking reviews, all from one model, named the same two gaps independently, and one passing review raised the first of them too.

### Consolidated themes and how v0.10 resolves them

| # | Theme (raised by) | Resolution in v0.10 | Where |
|---|---|---|---|
| P1 | A pause only stopped issuing credentials, so a live token could still push until it expired (Sol A1, A2, A3; Astra A1; Opus A1) | **A pause is a barrier.** It revokes every outstanding credential at its provider. The agent shows as `pausing` until a probe with the old token is denied, and only then is the pause acknowledged. The repository's proof check also reads the agent's current control epoch | §5, §6.14, F35 |
| P2 | The takeover converted an agent's `exercise` grants into `delegate` rights, which widens authority (Sol A1, A2, A3) | **Fresh authority, never converted authority.** The successor's Duties get new delegations from the sponsor's existing delegable mandate, with caveats kept and a risk check, or from new consent. Work without one stays parked while the old agent stays fenced. D6 and F35 test that an `exercise`-only grant can't be converted | §6.14, F35, D6 |
| P3 | Smaller items (Opus A1, A2, A3; Sol A1, A3) | A managed agent runs in a Year96 sandbox, out of reach of the sponsor's own credentials, and CLIs that can't route their model calls can never be managed. Its trusted launcher attests the harness, not the agent itself. Threads, waits and commitments are re-homed. Unmanaged agents get a sponsor-attested retirement. The outside-agent lifecycle is in §5's table. agentgateway is named as the tool gateway. The sealed-eval author is independent, with hashes pinned by the bootstrap verifier. The `AGENTS.md` opening is qualified | §5, §6.14, `AGENTS.md` |

## Round 8 (v0.10): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **YES** | **YES** | **YES** |
| **A2 Autonomy, safety and resilience** | **YES** | **YES** | **YES** |
| **A3 Human experience, fidelity and DoD** | **NO**: 1 blocker | **YES** | **YES** |

Eight of nine passed, and every Round-7 blocker was confirmed resolved. All nine reviews independently named the same last item, and one counted it as blocking. The new external-agent state table had no transition out of `paused`, so a paused agent could be neither resumed nor retired, and F35's takeover couldn't complete.

### How v0.11 resolves it

| # | Theme (raised by) | Resolution in v0.11 | Where |
|---|---|---|---|
| Q1 | No way out of `paused` (all nine; blocking for Sol A3) | `paused → managed` only through an authorized `control.resume` that revalidates authority, issues fresh credentials and requires a new launcher attestation. `paused → retiring → retired` is the takeover. `pausing` has a deadline that raises a flag while the agent stays fenced | §5 |
| Q2 | Takeover order (Opus A1) | Authority before liability, as §6.6 requires. The successor gets its fresh delegations before threads, waits and commitments are re-homed | §6.14 |
| Q3 | The owner's read rights (Opus A3) | Sealed evaluation material is the second named exception to the owner reading everything, and the owner authors it with a separate credential | §6.1 |

### New owner input: the Q&A's sixth answer

The owner added a sixth answer. The main flow is the same as in any app: data comes in, a state snapshot is created, the scope effect spreads through Ownerships and Duties, tasks are derived, and the loop repeats.
v0.11 makes this the organizing loop of the architecture. §4's "The main flow" has a diagram and a step-by-step mapping to the components. The three older loops become this loop at different speeds, and every catalog flow is one path through it.
§5 adds `TaskOrigin`, so every derived task records its data, snapshot cut and scope prediction. D2 proves that each derived task traces back and replays. Appendix C gains QA6.

## Round 9 (v0.11): verification

| Aspect | gpt-6-sol | gpt-6-astra | claude-opus-5.5 |
|---|---|---|---|
| **A1 Flows and buildability** | **YES** | *pending* | **YES** |
| **A2 Autonomy, safety and resilience** | **YES** | **YES** | **YES** |
| **A3 Human experience, fidelity and DoD** | **NO**: 1 blocker | **YES** | **YES** |

The state-table item was confirmed resolved, and every reviewer judged the main flow faithful to the Q&A, consistent with the contracts and safe. The one blocking review asked for the origin record to cover every derived task, and five passing reviews raised the same point as non-blocking.

### How v0.12 resolves it, and the owner's Q5 decision

| # | Theme (raised by) | Resolution in v0.12 | Where |
|---|---|---|---|
| R1 | `TaskOrigin` covered only BuilderRequests and one event, and its route was optional (Sol A3; Opus A1, A3; Sol A1, A2; Astra A3) | `origin` now sits on every derived intent: Duty changes, BuilderRequests, questions and flags. It lists several source events for coalesced wakes, and its route is always named, either a scope prediction or a deterministic rule | §4, §5 |
| R2 | Replay was underspecified (Opus A2, A3; Sol A2; Astra A2) | D2 replays in the harness with recorded model I/O and no live EffectDispatcher, exact up to permanent redactions. A second test replays an old cut after a live revocation, halt or exhausted budget, and a live resubmission must pass today's gates | D2 |
| R3 | The loop's cycles weren't covered by the causal caps (Opus A2) | `TaskOrigin` carries the root event and a causal depth, so §6.3's caps bound every turn of the main flow, not only thoughts | §4, §5, §6.3 |
| R4 | Snapshot pins and exceptional paths (Sol A3, Sol A1) | The cheap `versions` snapshot also pins the offsets of the projections it read. Genesis and kernel-down supervisor recovery are named as the two paths outside the main flow | §4, §5 |

**The owner decided Q5**: no local models, and every model call goes through hosted APIs and subscriptions. §8.1 was rewritten around hosted providers with approved data terms, routed across at least two of them. §6.9, §6.13, §6.14 and the §8, §9 and §12 tables follow it, and a new risk 19 covers provider dependence and subscription terms.

## Round 10 (v0.12): verification

*This section is filled in as the nine reviews come back.*
