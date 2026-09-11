# copilot-agent-foundry

Central hub for reusable GitHub Copilot **custom agents** and **skills**. This repo is the canonical source ("agent-repo") that other repositories sync agent definitions and domain-knowledge skills from, so teams can share a consistent set of specialist agents and best-practice references across every codebase they work in.

## What's in here

```
.github/
├── agents/       # Custom agent definitions (*.agent.md)
├── skills/       # Domain-specific knowledge packages (SKILL.md + reference files)
└── workflows/    # CI checks (skill frontmatter validation)
```

- **Agents** are Copilot custom chat modes — each `.agent.md` file defines an agent's identity, allowed tools, sub-agents it can delegate to, and behavioral instructions.
- **Skills** are self-contained `SKILL.md` packages (with optional `references/` files) that inject deep, topic-specific knowledge into an agent when a matching task is detected.
- Agents and skills are designed to be composed: orchestrator-style agents delegate to specialist agents, which in turn dynamically load the skills relevant to the task at hand.

## Agents (`.github/agents/`)

| Agent | Purpose |
| --- | --- |
| **RUG** (`rug-orchestrator`) | Pure orchestration agent — decomposes requests, delegates all work to subagents, validates outcomes, and repeats until complete. |
| **Architect** | Designs scalable systems, reviews architecture, and produces actionable specs for implementation agents. |
| **Backend Engineer** | Server-side specialist for APIs, databases, and auth; dynamically loads framework skills (.NET, etc.) based on project detection. |
| **Frontend Engineer** | Builds, reviews, and optimizes frontend apps — component architecture, state management, styling, accessibility. |
| **Full-Stack Engineer** | Implements features end-to-end across frontend, backend, database, and infrastructure. |
| **Code Reviewer** | Post-implementation review focused on correctness, security, performance, and style consistency. |
| **Test Writer** | Generates comprehensive, maintainable test suites, loading framework skills as needed. |
| **Product Owner** | User story generation, backlog management, and product discovery. |
| **UI Interpreter** | Analyzes screenshots/wireframes/mockups and turns them into structured implementation briefs for specialist agents. |
| **Foundry** | Creates and maintains the agent/skill infrastructure itself (`.agent.md`, `.instructions.md`, `.prompt.md`, `SKILL.md`). |
| **Context Manager** | Extracts minimal, token-bounded context packets for a target agent instead of dumping whole files. |
| **Context7-Expert** | Answers library/framework questions using up-to-date documentation. |
| **Handoff Agent** | Emits a structured JSON state payload to resume work in a new chat or hand off to another agent. |
| **Software Engineer Agent** | General-purpose, expert-level engineering agent for production-ready, spec-driven implementation. |

## Skills (`.github/skills/`)

| Skill | Domain |
| --- | --- |
| `rug-routing` | Authoritative routing table the RUG orchestrator reads to decide which agent handles each task. |
| `local-routing` | Repo-specific routing overrides layered on top of `rug-routing` for optional agents. |
| `agent-builder` / `skill-builder` | Meta-skills for building, editing, and reviewing agent customization files and SKILL.md packages. |
| `dotnet-server` / `dotnet-migration` | ASP.NET Core best practices and .NET Framework → modern .NET migration guidance. |
| `ef-core-pro` | Entity Framework Core modeling, querying, migrations, and performance. |
| `mediatr-pro` | CQRS and MediatR handler/pipeline patterns. |
| `postgres-pro` | PostgreSQL schema design, indexing, partitioning, and query tuning. |
| `respawn-pro` / `testcontainers-dotnet-pro` | Database reset and containerized integration testing for .NET. |
| `docker-pro` | Dockerfile, multi-stage build, and container security best practices. |
| `react19-pro` / `react-native-pro` | React 19 APIs and React Native cross-platform patterns. |
| `insomnia-pro` | Insomnia API client collections, scripting, and CI workflows. |
| `workmaker-pro` | User story, epic, and backlog generation using INVEST and story mapping. |
| `ui-interpreter` | Translating UI screenshots/mockups into implementation briefs. |

## CI

[`validate-skills.yml`](.github/workflows/validate-skills.yml) runs on every push/PR to `main` and checks that every `SKILL.md` in the repo has valid YAML frontmatter with the required `name` and `description` fields.

## Using this repo

Downstream repositories reference this foundry as their source of agents/skills (e.g. via a `.copilot-deps.json` manifest) and sync the specific `.agent.md` and `SKILL.md` files they need. Within a consuming repo, `local-routing` is extended with repo-specific rules that layer on top of the canonical `rug-routing` table.
