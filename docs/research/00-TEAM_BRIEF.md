# Year96 Research Team — Shared Brief

Date: 2026-09-28 · Lead: GitHub Copilot CLI, acting as the Year96 "Owner". The lead coordinates; teammates research.
**Mission:** work out concretely and technically how the Year96 "Agentic OS" for 2027 could be built. Base it on current
research, trending GitHub projects, articles, libraries, and work by Fortune-100 companies and frontier AI labs.

---

## 1. Read first (mandatory)
- `C:\projects\year96_!\year96\docs\YEAR96_INTRO.md`: the concepts
- `C:\projects\year96_!\year96\docs\YEAR96_SPEC.md`: engineering principles and the required stack
- `C:\projects\year96_!\year96\docs\YEAR96_Vision.md`: the end-to-end vision

All three are short. Read them in full. The summary below does not replace them.

## 2. Year96 on one page
Year96 is a 2027 **Agentic OS**. It takes *any* human desire and fully offloads it, and it keeps improving itself to fit that human.

- **STATE**: the whole world is one big state. That covers clocks, text, a rock in a field, bits, networks, and even *thoughts*.
  The entire state must be **searchable**. The system is part of the state too, so it can fix itself or re-architect itself,
  evaluate the change over time, and upgrade once it is satisfied. Picture "freezing the entire world".
  **Update (2026-09-28): two state domains.** **Internal organization state** is everything that happens inside the organization.
  **External state** is the rest of the world. External events arrive through **RSS-like feeds and subscriptions to outside
  events**, and **routing rules** update the organization's *relevant* state when relevant external state changes.
- **SCOPE EFFECT**: when one part of the state changes, which other parts should change or be taken into account? This is hard to predict.
  Example: a rocket company later uses its technology to compete in food ingredients. Scope effect must be predicted
  **per identity and per thread** and must **improve over time**: learn what matters to a stream, and what to track because it may
  matter later. **Thoughts alone create scope effect**. In the "starving Yossi" example the world is frozen but his thoughts are not, so he
  seeks food. For this reason harnesses **generate thoughts** that keep ownership moving even when the world is quiet.
- **ORGANIZATION**: a unit that delivers value and receives money in return. It is made of identities.
- **IDENTITY**: any object with at least one permission in the organization (humans, agents, services, and so on).
- **CAPABLE AGENT**: *owns* its task rather than just doing it. It improves the **why** over time and supplies the optimal
  **how**. It has the best tools available, plus the permissions and environments to own entire workflows.
- **COMMUNICATION HUB**: all communication passes through it. **Ext Comm** is with identities outside the org; **Int Comm** stays inside.
  It is made of **Communicators**, which lead and manage threads and keep each thread's goal and scope on track. Their role is **oversight only**:
  they must never interfere with decisions, actions, conclusions, thought processes, or outputs.
- **THREAD**: every process in the org has a representative thread, from changing a button colour to choosing Mars-craft
  materials. Parts: creation date, last active, **memory**, **meta-memory** (the mental model of *why* the thread
  exists), **chats** (including one identity thinking to itself), and **milestones**. **Threads never close.** A thread can sit dormant
  for years and reactivate. Identities can **clone themselves**, for example to brainstorm with their clones. They can **listen** to a thread without taking part,
  or **hang** on a thread until a state change affects it.
- **OWNERSHIP LAYER**: stores the human's **mental model** and extends it into ongoing ownership. That covers the **why** of everything
  (decisions, processes, bugs, preferences), **strategies** (long-term and short-term), **sensors**, **optimal vectors**, and **memory**.
  Permissions: CRUD its own Duties; *discuss* CRUD of Ownerships.
- **DUTY LAYER**: memory for sub-ownership topics: user memory, project settings, topic knowledge, and ongoing processes. It turns the
  mental model into **actionable insights**. Permissions: *discuss* CRUD of sibling Duties; *request* execution-level Builders.
- **EXECUTION LAYER**: everything descends from **Builder**, which takes (goal, capabilities, permissions, tools) and reaches the goal.
  It may prototype and may hit barriers. If the goal can't be reached in the agreed time, it **raises a flag** through the communication channels.
- **Architecture notes**: each component may contain many internal identities and tools. An identity can talk to any identity
  *at its level or below*, **unless the thread has its parent as a participant**.
- **Vision flow**: a human has a desire, opens a thread, and talks to a Communicator. The Communicator brings in the right identities, and the layers
  then self-manage: they wake up, hand off tasks, and self-improve. Examples: "capture my mental model better over time", "start
  Facebook ads for my business", "build an Unreal Engine game". These yield *thousands of endless agent processes*.

## 3. Engineering principles (from the spec)
- Everything is **interface-based and agnostic**. **Logic is separated from state.** Functional and stateless designs are preferred over stateful ones.
- **About 70% of effort goes into validation and QA frameworks** that prove each feature or fix works "at 100%". Levels: unit, integration, mocked integration,
  E2E, mocked E2E, **agentic verifiers**, several environments, and several starting states. **No task is complete without proofs at every level.**
- **Observability at every level**: screenshots, console dumps, logs, audit. Before any command, test, or process: (1) capture the state before,
  (2) set up tracing so bugs are traceable and reproducible, (3) monitor the state while it runs, (4) define the expected end state, (5) capture the end state.
- **Time awareness**: every command is wrapped in a timeout. Every agent session sets a timer and checks the clock **every 15 min**.
  Durations are measured and compared against how often the operation runs, so that bottlenecks get flagged.
- **Don't reinvent the wheel**: research first (GitHub trending, most-starred and most-contributed projects, articles, Reddit, Fortune-500 launches,
  Hugging Face, Docker Hub).
- **The Owner never executes.** It manages Duty agents, and Duty agents launch Executors with well-defined context (why, task, references,
  a detailed goal). Use **NousResearch hermes-agent** for repetitive automations and monitored ongoing processes.
- **Programmatic enforcement gates at every level**: identity harness, tasks, environments, coding, messaging.
- **Required stack**: base agent harness = **https://pi.dev/** (adapt, extend, or fork). Methodologies: **cursor/plugins `pstack`**
  (primary), **obra/superpowers** (secondary), **mattpocock/skills** (tertiary). Repo: `github.com/MenachemBarak/year96` (currently empty).
- **Code deployability** *(added to the spec on 2026-09-28)*: "should be deployable across servers, i.e. in case of millions of agents, but also can run on one machine."
  The **same code** must run on a single machine and scale out across many servers to millions of agents.

## 4. The user's standing preferences (apply to every recommendation)
- Prefer **established, highly regarded, actively maintained** solutions over reinventing them.
- **Core technology must be permissively licensed**: MIT, Apache-2.0, BSD, ISC, or public domain. Mark **copyleft** licences (GPL/AGPL/LGPL/MPL) and
  **use-restricted** licences (BSL/BUSL, SSPL, Elastic License, Commons Clause, "fair-code", custom non-OSI) as **"Excluded-license (core)"**.
  Such projects may still appear as *external integrations*. External harness integrations can have separate terms.
- **Composable, provider-based architecture** with explicit interfaces at every layer (the provider pattern is preferred over the factory pattern).
  Every major capability sits behind an interface, and implementations are swappable providers.
- **TDD, red first**, verified at the E2E or visual level where applicable. Work is organized in git, with per-task commits and version tags.

## 5. Team roster (12 teammates). Stay in your lane and cross-reference the others by number.
| # | Teammate | Focus | Report file (in `docs\research\`) |
|---|---|---|---|
| 01 | state-fabric | Universal STATE: world-as-state, event logs, time travel and snapshots, CDC and connectors, knowledge graphs, unified search, "thoughts as state", system-as-state | `01-state-fabric.md` |
| 02 | scope-effect | SCOPE EFFECT engine: predicting which state changes matter to whom, how much, and over what horizon; learning over time; generating thoughts | `02-scope-effect-engine.md` |
| 03 | thread-memory | THREADS: memory, meta-memory, chats, milestones; dormancy and reactivation; context engineering; bounded growth | `03-thread-memory.md` |
| 04 | comm-hub | COMMUNICATION HUB and Communicators; agent protocols (MCP, A2A, …); messaging substrate; Ext/Int gateways; clones, listeners, hangers | `04-communication-hub.md` |
| 05 | runtime | Durable runtime: virtual actors, durable workflows, schedulers, sandboxes, model gateways, time awareness | `05-runtime-durable-execution.md` |
| 06 | identity-gates | IDENTITIES and permissions: agent authentication, ReBAC and capabilities, clone attenuation, guardrails, policy gates, audit, regulation | `06-identity-permissions-gates.md` |
| 07 | ownership-duty | OWNERSHIP and DUTY layers: mental models, capturing the "why", strategies, sensors, optimal vectors, planning, cognitive architectures, org-design analogues | `07-ownership-duty-cognition.md` |
| 08 | self-improvement | Self-improvement and self-re-architecture: evolutionary agent design, prompt/skill/workflow optimisation, RL for agents, safe promotion | `08-self-improvement-evolution.md` |
| 09 | verification | The 70%: proofs of done, test levels, agentic verifiers, formal methods, deterministic simulation, observability, audit, time measurement | `09-verification-proof-observability.md` |
| 10 | fortune100 | Agent platforms at Fortune-100 companies and frontier labs; how enterprise agentic OSes are converging; lessons learned and failure data | `10-fortune100-frontier-labs.md` |
| 11 | harness-stack | The required stack: pi.dev, hermes-agent, pstack, superpowers, mattpocock/skills, Agent Skills and AGENTS.md; design of the Year96 harness | `11-harness-methodology-stack.md` |
| 12 | github-trending | Trending GitHub projects 2025–2026; OSS agent orchestrators and "AI company" projects; the agentic-OS lineage (AIOS, UFO², …) | `12-github-trending-oss-agentic-os.md` |
| 13 | world-feeds | *(added after the State update)* Internal vs external state; RSS-like world feeds, subscriptions and routing into org-relevant state; the state membrane | `13-world-feeds-state-membrane.md` |
| 14 | scale-out | *(added after the Code requirement)* Same code from one machine to millions of agents: run profiles, cells, partitioning, capacity model, scale proofs | `14-scale-one-machine-to-millions.md` |
| 15 | community-registries | *(gap fill)* Reddit, HN and Facebook signals; Hugging Face model choices for the core (with licenses); Docker Hub images and supply chain | `15-community-models-registries.md` |

## 6. Output contract (every teammate)
1. Write your **full report** as Markdown to your file in `C:\projects\year96_!\year96\docs\research\`. Write a skeleton early,
   at about 10 minutes, and update it as you go so partial work survives.
2. Length: **about 3,000–6,000 words**, dense, with no filler.
3. Required sections, in this order:
   1. `# NN — Title` and a one-paragraph scope.
   2. `## TL;DR for the Year96 architect`: 8–15 bullets with the findings that matter most for decisions.
   3. `## Landscape`: papers and researchers, OSS projects, products, articles. For each one: what it is (1–3 sentences), **why it matters to
      Year96**, and its date (month/year). Prioritise 2025–2026. Say if it is deprecated, archived, acquired, or relicensed.
   4. `## Component scorecard`: a table with columns
      `| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |`.
      The verdict is one of **Adopt / Trial / Watch / Avoid / Excluded-license**.
   5. `## How I would build this part of Year96`: a concrete, opinionated technical design. Include TypeScript-style **provider interfaces**,
      the data model (types and schemas), key algorithms, the sequence of a typical flow, how it plugs into the other layers (cite teammates by
      number), a scaling path (laptop to cluster), and how it is **tested and proven** (the 70% rule).
   6. `## What is still unsolved (late 2026)`: open problems, risks, and where Year96 must innovate rather than adopt.
   7. `## Sources`: a numbered list of URLs you actually opened. Cite them inline as [n].
4. **Verify.** Check GitHub repos directly (LICENSE file, stars, last commit or release), not from memory. Mark anything you could not
   verify as `(unverified)`. **The leads in your assignment come from the lead's memory and may be wrong or out of date.** Confirm
   or correct them, and actively look for newer work from 2026 that the lead does not know about.
5. **Final message to the lead** (at most 900 words): TL;DR bullets, the top Adopt/Trial components with their licences, a design headline
   (3–6 bullets), the biggest open problem, and the path of the file you wrote.

## 7. Rules of engagement
- **Time-box: about 35 minutes.** At about 30 minutes, stop researching and finalise the file (the spec's time awareness).
- Wrap every shell command in a timeout. Prefer the web_search, web_fetch, and GitHub tools over the shell.
- Write **only your own report file**. Do not modify the Year96 docs, this brief, or other teammates' files. No git operations and no
  global installs. Never stop or kill processes you did not start.
- Prefer primary sources: arXiv papers, official docs, GitHub repos, company newsrooms, engineering blogs. Avoid SEO summaries.
