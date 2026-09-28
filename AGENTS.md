# AGENTS.md: the harness record for agents working on Year96

Agents that build or operate Year96 from outside are part of its state (`docs/YEAR96_TECHNICAL_ARCHITECTURE.md` §6.14). This file is their versioned harness record.
It records how they are configured and what they may do. Once an agent is managed (§6.14), Year96 enforces this record. Today the owner applies it by hand.

**Today's outside builder is unmanaged** in §6.14's terms. This file records it, but nothing enforces it yet: the owner controls the builder directly. Year96 will manage outside agents only once they act through
credentials it issues and revokes, with gated writes and an attested harness.

## Methodology precedence

When two skills disagree, the higher rule wins.

1. Year96's own rules: `docs/YEAR96_SPEC.md`, `docs/YEAR96_INTRO.md`, `docs/YEAR96_Vision.md` and the owner's Q&A.
2. **pstack** is the main methodology. For any nontrivial task, apply `/poteto-mode` first and follow its playbooks, principles and reply rules.
3. **superpowers** is the secondary methodology. Use its skills where pstack has no counterpart, such as brainstorming a vague desire, writing plans, subagent-driven development and verification before completion.
4. **mattpocock/skills** is the third. Use it to manage and organize work: `grill-me`, `to-spec`, `to-tickets`, `triage`, `domain-modeling`, `wayfinder` and `handoff`.
5. Other repository skills come last.

Two packs can ship a skill with the same name. Copilot then prefixes them (`pstack:tdd`, `mattpocock-skills:tdd`). Prefer the pstack one.

## Harness record

| Agent | Harness | Methodology packs, pinned |
|---|---|---|
| Outside builder | GitHub Copilot CLI 1.0.88 | pstack 0.15.5 (cursor/plugins `adf3218`), superpowers v6.4.2, mattpocock-skills v1.2.3 |
| Year96's own agents | pi, `@earendil-works/pi-coding-agent` 0.87.1 | The same three packs, pinned the same way |

On the owner's machine, install and update steps live in `../.plugins/README.md`, outside this repository.

## Working rules

- Work on `main` with per-task commits, and tag each design or release milestone.
- Large-scale task management uses the GitHub project "Year96" at https://github.com/users/MenachemBarak/projects/3.
- Agent implementations use pi as the base harness. Extend it, and fork only when an extension can't do the job.
- Code must deploy across servers for millions of agents and also run on one machine.
