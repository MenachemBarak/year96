# Year96 research: team reports

These are the findings of the Year96 research sprint (2026-09-28): 15 parallel research teammates, each owning one layer of the imagined 2027 Agentic OS.
Every report has a TL;DR, a landscape section, a component scorecard with verified licenses, a concrete design with provider interfaces, open problems, and sources.

**The combined architecture is in [../YEAR96_TECHNICAL_ARCHITECTURE.md](../YEAR96_TECHNICAL_ARCHITECTURE.md).**

| # | Report | Owns |
|---|---|---|
| 00 | [00-TEAM_BRIEF.md](00-TEAM_BRIEF.md) | Shared context, roster, output contract, rules of engagement |
| 01 | [01-state-fabric.md](01-state-fabric.md) | World-as-state, the event log, bitemporal time travel, world freeze, unified search, privacy |
| 02 | [02-scope-effect-engine.md](02-scope-effect-engine.md) | Relevance cascade, calibration, watchlists, Thought Generator |
| 03 | [03-thread-memory.md](03-thread-memory.md) | Never-closing threads, memory tiers, milestones, rehydration, clones |
| 04 | [04-communication-hub.md](04-communication-hub.md) | Hub planes, MCP/A2A/AG-UI, Communicators, talk rule, gateways |
| 05 | [05-runtime-durable-execution.md](05-runtime-durable-execution.md) | Virtual actors, Temporal Builders, sandboxes, model gateway, time awareness |
| 06 | [06-identity-permissions-gates.md](06-identity-permissions-gates.md) | Agent identity, ReBAC/ABAC/capabilities, gate pipeline, audit, regulation |
| 07 | [07-ownership-duty-cognition.md](07-ownership-duty-cognition.md) | Ownership and Duty object models, why-graph, optimal vectors, authority calibration |
| 08 | [08-self-improvement-evolution.md](08-self-improvement-evolution.md) | L0–L6 mutation ladder, immutable kernel, safe rollout |
| 09 | [09-verification-proof-observability.md](09-verification-proof-observability.md) | Proof-of-Done, verifier independence, OTel, the 70% |
| 10 | [10-fortune100-frontier-labs.md](10-fortune100-frontier-labs.md) | Enterprise agent platforms, convergence, failure data |
| 11 | [11-harness-methodology-stack.md](11-harness-methodology-stack.md) | pi, hermes-agent, pstack, superpowers, mattpocock/skills |
| 12 | [12-github-trending-oss-agentic-os.md](12-github-trending-oss-agentic-os.md) | 50+ trending repos, 2026 patterns, agentic-OS lineage |
| 13 | [13-world-feeds-state-membrane.md](13-world-feeds-state-membrane.md) | Internal vs external state, RSS-like world feeds, internalization, observe-back |
| 14 | [14-scale-one-machine-to-millions.md](14-scale-one-machine-to-millions.md) | One artifact from one machine to millions of agents: profiles, deployability contract, cells, capacity model |
| 15 | [15-community-models-registries.md](15-community-models-registries.md) | Hugging Face model profiles (permissive), Docker Hub supply chain, Reddit/HN/Meta practitioner signals |

**Notes**

- Licences were checked against GitHub repos where possible. Some claims are marked `(unverified)` inside the reports. The lead corrected four of them during
  synthesis; see Appendix B of the architecture document.
- Star counts and dates are as of 2026-09-28. Fast-moving projects relicense often, so re-run license checks before adopting anything.
