---
name: local-routing
description: >-
  Repo-specific routing overrides for the RUG orchestrator. Customizations here
  take precedence over the canonical rug-routing rules synced from agent-repo.
  Add file-pattern overrides, custom triage rules, and repo-specific routing preferences.
---

# Local Routing Overrides

This file extends the canonical `rug-routing/SKILL.md` with repo-specific routing rules. The base rug-routing skill (synced from agent-repo) covers **core agents** that are always present: RUG, Foundry, Code Reviewer, Software Engineer Agent, Handoff, and Context7-Expert.

When you add **optional specialist agents** to your `.copilot-deps.json` `agents` array, you should add their routing rules here so RUG knows how to use them.

**How it works:**

- RUG reads `rug-routing` FIRST for the core agent roster and default rules
- RUG reads this file (local-routing) SECOND — rules here **override or extend** the defaults
- Core agent routing is already handled; this file is for optional agents and repo-specific customizations

**How to use this file:**

1. Add an optional agent to your `.copilot-deps.json` `agents` array (e.g., `"backend-engineer"`)
2. Run the sync workflow to pull the agent definition into your repo
3. Uncomment the corresponding rows in each section below
4. Adjust file patterns, triage rules, and handoffs to match your repo structure

---

## 1. Optional Agent Roster Extension

The core agent roster lives in `rug-routing`. The table below lists all **optional specialist agents** available from agent-repo. Uncomment agents you've added to your `.copilot-deps.json` `agents` array.

| #    | Agent | Domain                          | When to Route              | Skills Loaded                                                                               |
| ---- | ----- | ------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------- | 
| 1     | **Architect**                   | System design              | API design, system diagrams, architecture decisions, service decomposition, contract design | api-design-pro                                                                         |
| 2     | **Backend Engineer**            | Server-side                | API routes, database queries, server plugins, auth middleware, WebSocket handlers           | fastify-pro, supabase-pro, pocketbase-pro, dotnet-server, dotnet-migration, golang-api |
| 3     | **Frontend Engineer**           | Frontend web               | UI components, state management, composables, service workers, web app manifests            | vue-pro, pwa-pro, react19-pro                                                          |
| 4     | **Full-Stack Engineer**         | Cross-layer implementation | End-to-end features spanning frontend + backend + infra in a single coherent task           | All relevant framework skills (loaded dynamically)                                     |
| 5     | **Test Writer**                 | Testing                    | Test generation for any language/framework. Writes comprehensive test suites                | Framework skills loaded dynamically per test target                                    |
| 6    | **Product Owner**               | Product & backlog          | User stories, job stories, epics, features, backlog decomposition, story splitting, story mapping, acceptance criteria, product discovery | workmaker-pro                                                               |
| 7    | **UI Interpreter**              | UI from images             | Any task providing a screenshot, wireframe, mockup, or design image with an ask to build, analyze, or plan its implementation            | ui-interpreter                                                              |
| 8    | **Context Manager**             | Context pruning            | Assembling a bounded context packet (code slices + diffs) before dispatching a large or cross-file task to an implementation agent       | — (read-only, no skills)                                                   |

---

## 2. File Pattern Overrides

Add rows to route specific file patterns to the correct specialist agent. Uncomment rows for agents you've enabled, and adjust patterns to match your repo's directory structure.

| File Pattern / Path | Route To | Notes |
| ------------------- | -------- | ----- |
| `src/**/*.cs`, `src/**/*.csproj`, `src/**/appsettings*.json` | **Backend Engineer** | .NET source code — controllers, handlers, repositories, infrastructure |
| `tests/**` | **Test Writer** | xUnit test projects — unit and integration tests |
| `docs/ARCHITECTURE*.md`, `docs/DESIGN_PATTERNS.md`, `docs/REPO_ANALYSIS.md`, `docs/MES_QUALITY_ANALYSIS.md` | **Architect** | Architecture and design documentation |
| `docs/**` | **Architect** | Any docs-layer file — design, runbooks, analysis |
| `Dockerfile*`, `docker-compose*`, `azure-pipeline.yml`, `local/**` | **Software Engineer Agent** | Container and CI/CD config — no dedicated Infra agent in this repo |

<!-- Uncomment if you have a repo-specific override for an existing core route: -->
<!-- | `apps/admin/**` | **Frontend Engineer** | Admin panel uses Vue, not generic SW Engineer | -->

<!-- Uncomment Backend Engineer patterns when backend-engineer is enabled: -->
<!-- | `apps/backend/**`, `routes/**`, `plugins/**`, `**/server.*`, `**/api/**` | **Backend Engineer** | Server-side code — adjust paths to your repo structure | -->

<!-- Uncomment Frontend Engineer patterns when frontend-engineer is enabled: -->
<!-- | `apps/frontend/**`, `components/**`, `views/**`, `stores/**`, `composables/**` | **Frontend Engineer** | Browser-side code — adjust paths to your repo structure | -->

<!-- Uncomment Mobile Engineer patterns when mobile-engineer is enabled: -->
<!-- | `**/*.swift`, `**/*.swiftui`, `*.xcodeproj/**`, `*.xcworkspace/**`, `**/Info.plist` | **Mobile Engineer** | iOS / SwiftUI files | -->
<!-- | `**/*.kt`, `**/*.kts`, `**/AndroidManifest.xml`, `**/build.gradle*`, `**/res/**` | **Mobile Engineer** | Android / Kotlin files | -->
<!-- | `**/lib/**` (Flutter), `**/pubspec.yaml`, `**/*.dart` | **Mobile Engineer** | Flutter / Dart files | -->

<!-- Uncomment Infrastructure Engineer patterns when infrastructure-engineer is enabled: -->
<!-- | `Dockerfile*`, `docker-compose*`, `Caddyfile*`, `.github/workflows/**`, `*.yml` (CI) | **Infrastructure Engineer** | Container and CI/CD config | -->
<!-- | `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, monorepo config | **Infrastructure Engineer** | Monorepo tooling | -->

<!-- Uncomment Architect patterns when architect is enabled: -->
<!-- | `docs/architecture*`, `docs/ARCHITECTURE.md`, API design docs | **Architect** | Architecture documentation and design | -->

<!-- Uncomment Test Writer patterns when test-writer is enabled: -->
<!-- | `**/*.test.*`, `**/*.spec.*`, `**/__tests__/**` | **Test Writer** | Test files — route here for test generation/updates | -->

<!-- Uncomment Full-Stack Engineer patterns when full-stack-engineer is enabled: -->
<!-- | `libs/shared/**` (shared types used by both frontend + backend) | **Full-Stack Engineer** | Cross-layer shared code | -->
<!-- | Cross-layer changes spanning frontend + backend in one feature | **Full-Stack Engineer** | End-to-end feature work | -->

---

## 3. Task Phase Overrides

Route tasks to the correct agent based on the current phase of work. Uncomment rows for agents you've enabled.

| Phase | Route To | Notes |
| ----- | -------- | ----- |
| **Implementation** — .NET handlers, controllers, repositories, DB queries, auth | **Backend Engineer** | Server-side production code in `src/` |
| **Implementation** — API design, system architecture, service decomposition, docs | **Architect** | Architecture decisions and design docs |
| **Implementation** — End-to-end feature spanning multiple layers | **Full-Stack Engineer** | Cross-layer feature implementation |
| **Implementation** — UI components, state, composables, service workers, PWA | **Frontend Engineer** | Browser-side production code |

<!-- | **Implementation** — iOS, Android, or Flutter native code | **Mobile Engineer** | Native mobile production code | -->
<!-- | **Implementation** — Docker, Caddy, CI/CD, monorepo tooling, deployment | **Infrastructure Engineer** | DevOps and build system work | -->

| **Testing** — writing or updating tests post-implementation | **Test Writer** | Launch AFTER implementation for test coverage |
| **Story writing** — user stories, job stories, acceptance criteria | **Product Owner** | Story and work item generation |
| **Backlog** — epics, features, story decomposition, story splitting, story mapping | **Product Owner** | Backlog management and discovery |
| **UI from image** — any task with a screenshot, wireframe, mockup, or design file | **UI Interpreter** | Route here BEFORE Frontend Engineer; produces briefs then delegates |
| **Context assembly** — task touches >5 files or spans layers | **Context Manager** | Route here BEFORE the implementation agent; emits a bounded context packet |
| **Session transfer** — context window near limit, or state must move between agents | **Handoff Agent** | Emits the structured JSON state payload (`spec`, `modified_files`, `blockers`) |

<!-- | **CI monitoring** — pipeline status, build failures | **CI Monitor Subagent** | Thin helper, single tool-call per invocation | -->
<!-- | **App store deployment** — signing, submission, profiles | **App Store Deployment Expert** | Code signing, provisioning, store metadata | -->

---

## 4. Bug Triage Overrides

Add rows to route specific bug symptoms to the correct diagnosis agent. Uncomment rows for agents you've enabled.

| Symptoms | Primary Diagnosis Agent | Notes |
| -------- | ----------------------- | ----- |
| API errors, HTTP 4xx/5xx, database query failures, auth (`x-api-key`) failures | **Backend Engineer** | ASP.NET Core server-side errors |
| MediatR pipeline failures, FluentValidation errors, `ValidationException` | **Backend Engineer** | Application layer errors |
| Npgsql / SQL errors, `DbOperationException`, `IUnitOfWork` failures | **Backend Engineer** | Data access errors |
| UI rendering bugs, state management issues, routing errors | **Frontend Engineer** | Browser-side rendering |
| API contract mismatches, schema validation errors across services | **Architect** | Cross-service design issues |

<!-- | iOS crashes, SwiftUI layout issues, Xcode build errors | **Mobile Engineer** | iOS platform issues | -->
<!-- | Docker build failures, container networking issues | **Infrastructure Engineer** | Container issues | -->
<!-- | CI pipeline failures, deployment errors | **Infrastructure Engineer** | CI/CD issues | -->
<!-- | Code signing errors, provisioning profile issues, store rejection | **App Store Deployment Expert** | Distribution issues | -->

---

## 5. Handoff Matrix Extension

Shows which optional agents can hand off to which. Uncomment the full matrix when you've enabled optional agents. A ✅ means the row agent can initiate a handoff to the column agent.

| From ↓ \ To →         | Context7 | Backend | Frontend | Architect | Full-Stack | Code Reviewer | Test Writer | SW Engineer | Foundry | Product Owner | UI Interpreter |
| --------------------- | -------- | ------- | -------- | --------- | ---------- | ------------- | ----------- | ----------- | ------- | ------------- | -------------- |
| **Architect**         | ✅       | ✅      | ✅       | —         | —          | ✅            | —           | —           | —       | ✅            | —              |
| **Backend Engineer**  | ✅       | —       | ✅       | ✅        | —          | ✅            | ✅          | —           | —       | —             | —              |
| **Frontend Engineer** | ✅       | ✅      | —        | —         | —          | ✅            | ✅          | —           | —       | —             | —              |
| **Full-Stack Engineer**| ✅      | —       | —        | ✅        | —          | ✅            | ✅          | —           | —       | —             | —              |
| **Test Writer**       | ✅       | —       | —        | —         | —          | ✅            | —           | —           | —       | —             | —              |
| **Product Owner**     | —        | —       | —        | ✅        | —          | —             | —           | ✅          | —       | —             | —              |
| **UI Interpreter**    | —        | ✅      | ✅       | —         | —          | ✅            | ✅          | —           | —       | —             | —              |

---

## 6. Custom Routing Rules

Add any repo-specific routing notes or constraints here. These are free-form instructions that RUG will follow.

- **Never pass a chat transcript between agents.** State transfer goes through the Handoff Agent's JSON payload (`spec`, `modified_files`, `blockers`, `next_action`, `constraints`) — nothing else.
- **Never pass a repository dump downstream.** For any task touching more than five files, route to **Context Manager** first and dispatch the implementation agent with its context packet.
- Context Manager is read-only by design. If it is asked to edit, that is a routing error — re-route to the correct specialist.

---

## 7. Model Tier Policy

Every agent must declare a `model:` in its frontmatter. An agent with no `model:` silently inherits the VS Code default — treat a missing `model:` as an audit failure.

| Tier | Purpose | Approved models | Agents |
| ---- | ------- | --------------- | ------ |
| **Tier 1** | Heavy reasoning — design, judgment, correctness | `Claude Opus 5 (copilot)`, `GPT-5.5 (copilot)` | Architect, Product Owner, Code Reviewer, Foundry |
| **Tier 2** | Standard execution — writing and modifying code | `Claude Sonnet 5 (copilot)`, `DeepSeek V4 Pro (copilot)`, `Gemini 3.6 Flash (copilot)` | Backend Engineer, Frontend Engineer, Full-Stack Engineer, Software Engineer Agent |
| **Tier 3** | Low-latency routing — retrieval, pruning, extraction, scaffolding | `Gemini 3.6 Flash (copilot)`, `GPT-5.3 Instant (copilot)`, `DeepSeek V4 Flash (copilot)` | Context Manager, Context7-Expert, Test Writer, Handoff Agent, UI Interpreter |

Rules:

- Never assign a Tier 1 model to a Tier 3 agent — retrieval and extraction do not need frontier reasoning.
- Model strings use the exact `Model Name (vendor)` format. An unrecognized string is dropped silently by VS Code and the agent falls back to the default — verify against the model picker after any change.
- RUG itself is exempt from the tier table; it delegates rather than reasons deeply, but must retain enough capability to decompose and validate.
