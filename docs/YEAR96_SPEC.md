# Year96 Spec



## Implementation mental model:

- Everything is interface based and agnostic.

- Logic is seperated from state.

- Functional programmina and Stateless functions and objects are better than statefull.

- Testing, QA and Proof:
    - Invest most of the time on building massive Validation and QA frameworks that can validate at 100% accuracy that any feautre/bug fix, delivered perfectly. liek about 70% must go on building those frameworks. 30% on the rest of work.
    - Test anything! at any level!:
        - Unit tests
        - Integration tests
        - Mocked integration tests.
        - E2E tests.
        - Mocked e2e tests.
        - Agentic tests (verifiers).
        - Testing in different environments.
        - Testing in different starting states
    - Task never can be completed without solid proofs with *all* levels of tests done successfuly.

- Monitoring, Visability, Observability are must:
    - those must be done at all level, in all environments, and for any tool used:
        - screenshots.
        - Console dumps.
        - Log files.
        - Audit framworks.
    - Never launch a command/test/process/etc without first: 1. capturing the state before. 2. taking care of solid obsevability for any part of the process to be traceable and if bug appear so reproduceable 3. capture the state ongoing and monitor it 4. define the expected end state 5. capture teh end state.

- Time awareness
    - any command must be wrapped in timeout. 
    - any session of any agnet must be setting a timer, and checking the current clock time and repetative monitored reminder time check every 15 min.
    - the time processes,command, tests, etc must be measured and according to theire rareness, need to decide if that a good time performents, or one that creates buttle neck that must be solved!

- Dont invent the wheel:
    - If any algorithem, tool, library, framwork, etc needed - never start coding, never inventing the wheel if not needed, 99% of the stuff and tech already developed in the world we prefered to use well maintained and trusted sources.
    - first launch winde research on github find the most trending projects in this area, most started, most contributed, read articales, search in reddit, facebook, fortune 500 companies launches and git organizations, well known registries like hugging face, docker hub etc.
    -  Its relevant for anything you try to do no matter in what dicipline it is: dev expiriance, devops, ai ops, infra, product, front, back, etc...

- You are the Owner of Year96:
    - never execute task by yourself. Instead Manage team of duties agents.
    - Each Duty Agent will launch executors that get well defined context, the why, task, references, higly detaild goal.
    - Use https://github.com/nousresearch/hermes-agent for taking repetative automations, and ongoing processes that needs to be monitored.


- Progrematic Enforcing gates at all levels, from identities harness level, though any task, though any environment, coding, messeging, everything must have solid quality checks.

## Implemetation requirements:

Repo:
- Use this repo https://github.com/MenachemBarak/year96.git.

Project:
- Use this project if needed massive managing https://github.com/users/MenachemBarak/projects/3 the project called Year96.

Agentic work:
- Use https://pi.dev/ as your base harness for any agent implemenation - adapt, extend, fork, it according to the needs.
- Use https://github.com/cursor/plugins/tree/main/pstack as the main work methodoloty.
- Use https://github.com/obra/superpowers as secondary work methodology.
- Use https://github.com/mattpocock/skills for mental models of how manage, orgenizing, wokring, and finding solutions with more orgenized workflows. Use it as thirds work methodology.

Code:
- should be deployable across servers i.e in case of millions of agents, but also can run on one machine