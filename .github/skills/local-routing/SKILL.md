---
name: local-routing
description: >-
  Repo-specific routing overrides for the RUG orchestrator. Customizations here
  take precedence over the canonical rug-routing rules synced from agent-repo.
  Add file-pattern overrides, custom triage rules, and repo-specific routing preferences.
---

# Local Routing Overrides

This file extends the canonical `rug-routing/SKILL.md` with repo-specific routing rules. The base rug-routing skill (synced from agent-repo) covers **core agents** that are always present: RUG, Foundry, code-reviewer, software-engineer-agent-v1, Handoff, and context7.

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
| 1     | **architect**                   | System design              | API design, system diagrams, architecture decisions, service decomposition, contract design | api-design-pro                                                                         |
| 2     | **backend-engineer**            | Server-side                | API routes, database queries, server plugins, auth middleware, WebSocket handlers           | fastify-pro, supabase-pro, pocketbase-pro, dotnet-server, dotnet-migration, golang-api |
| 3     | **frontend-engineer**           | Frontend web               | UI components, state management, composables, service workers, web app manifests            | vue-pro, pwa-pro, react19-pro                                                          |
| 4     | **Full-Stack Engineer**         | Cross-layer implementation | End-to-end features spanning frontend + backend + infra in a single coherent task           | All relevant framework skills (loaded dynamically)                                     |
| 5     | **test-writer**                 | Testing                    | Test generation for any language/framework. Writes comprehensive test suites                | Framework skills loaded dynamically per test target                                    |
| 6    | **Product Owner**               | Product & backlog          | User stories, job stories, epics, features, backlog decomposition, story splitting, story mapping, acceptance criteria, product discovery | workmaker-pro                                                               |
| 7    | **UI Interpreter**              | UI from images             | Any task providing a screenshot, wireframe, mockup, or design image with an ask to build, analyze, or plan its implementation            | ui-interpreter                                                              |
| 8    | **Context Manager**             | Context pruning            | Assembling a bounded context packet (code slices + diffs) before dispatching a large or cross-file task to an implementation agent       | — (read-only, no skills)                                                   |
| 9    | **terraform-expert**            | Infrastructure as Code     | Terraform/OpenTofu configs, modules, state operations, providers, backends, plan/apply pipelines, IaC refactoring (`moved`/`import`)       | — (domain knowledge in agent body)                                         |
| 10   | **aws-cloud-expert**            | AWS architecture           | AWS service selection, IAM policies, VPC/networking, serverless/containers, observability, cost optimization, Well-Architected reviews   | — (domain knowledge in agent body)                                         |

---

## 2. File Pattern Overrides

Add rows to route specific file patterns to the correct specialist agent. Uncomment rows for agents you've enabled, and adjust patterns to match your repo's directory structure.

| File Pattern / Path | Route To | Notes |
| ------------------- | -------- | ----- |
| `src/**/*.cs`, `src/**/*.csproj`, `src/**/appsettings*.json` | **backend-engineer** | .NET source code — controllers, handlers, repositories, infrastructure |
| `tests/**` | **test-writer** | xUnit test projects — unit and integration tests |
| `docs/ARCHITECTURE*.md`, `docs/DESIGN_PATTERNS.md`, `docs/REPO_ANALYSIS.md`, `docs/MES_QUALITY_ANALYSIS.md` | **architect** | Architecture and design documentation |
| `docs/**` | **architect** | Any docs-layer file — design, runbooks, analysis |
| `Dockerfile*`, `docker-compose*`, `azure-pipeline.yml`, `local/**` | **software-engineer-agent-v1** | Container and CI/CD config — no dedicated Infra agent in this repo |
| `**/*.tf`, `**/*.tfvars`, `**/*.tftest.hcl`, `**/.terraform.lock.hcl`, `**/terragrunt.hcl` | **terraform-expert** | Terraform/OpenTofu code — route AWS design questions to aws-cloud-expert first |
| `**/cdk.json`, `**/template.y*ml` (CloudFormation/SAM), `**/*.iam.json`, IAM policy documents, AWS architecture docs | **aws-cloud-expert** | AWS-specific infrastructure and IAM definitions |

---

## 3. Task Phase Overrides

Route tasks to the correct agent based on the current phase of work. Uncomment rows for agents you've enabled.

| Phase | Route To | Notes |
| ----- | -------- | ----- |
| **Implementation** — .NET handlers, controllers, repositories, DB queries, auth | **backend-engineer** | Server-side production code in `src/` |
| **Implementation** — API design, system architecture, service decomposition, docs | **architect** | Architecture decisions and design docs |
| **Implementation** — End-to-end feature spanning multiple layers | **Full-Stack Engineer** | Cross-layer feature implementation |
| **Implementation** — UI components, state, composables, service workers, PWA | **frontend-engineer** | Browser-side production code |
| **Implementation** — Terraform/OpenTofu modules, state, providers, IaC pipelines | **terraform-expert** | Infrastructure code; never applies without user approval |
| **Implementation** — AWS service selection, IAM policy design, VPC/networking, cost review | **aws-cloud-expert** | Design first, then hand off to terraform-expert for IaC implementation |
| **Testing** — writing or updating tests post-implementation | **test-writer** | Launch AFTER implementation for test coverage |
| **Story writing** — user stories, job stories, acceptance criteria | **Product Owner** | Story and work item generation |
| **Backlog** — epics, features, story decomposition, story splitting, story mapping | **Product Owner** | Backlog management and discovery |
| **UI from image** — any task with a screenshot, wireframe, mockup, or design file | **UI Interpreter** | Route here BEFORE frontend-engineer; produces briefs then delegates |
| **Context assembly** — task touches >5 files or spans layers | **Context Manager** | Route here BEFORE the implementation agent; emits a bounded context packet |
| **Session transfer** — context window near limit, or state must move between agents | **Handoff Agent** | Emits the structured JSON state payload (`spec`, `modified_files`, `blockers`) |

---

## 4. Bug Triage Overrides

Add rows to route specific bug symptoms to the correct diagnosis agent. Uncomment rows for agents you've enabled.

| Symptoms | Primary Diagnosis Agent | Notes |
| -------- | ----------------------- | ----- |
| API errors, HTTP 4xx/5xx, database query failures, auth (`x-api-key`) failures | **backend-engineer** | ASP.NET Core server-side errors |
| MediatR pipeline failures, FluentValidation errors, `ValidationException` | **backend-engineer** | Application layer errors |
| Npgsql / SQL errors, `DbOperationException`, `IUnitOfWork` failures | **backend-engineer** | Data access errors |
| UI rendering bugs, state management issues, routing errors | **frontend-engineer** | Browser-side rendering |
| API contract mismatches, schema validation errors across services | **architect** | Cross-service design issues |
| `terraform plan/apply` errors, state lock or drift, provider/version conflicts, module errors | **terraform-expert** | IaC tooling failures |
| `AccessDenied`, IAM policy errors, VPC connectivity/security group issues, Lambda/ECS/RDS runtime failures on AWS, unexpected AWS costs | **aws-cloud-expert** | AWS platform and permission issues |

---

## 5. Handoff Matrix Extension

Shows which optional agents can hand off to which. Uncomment the full matrix when you've enabled optional agents. A ✅ means the row agent can initiate a handoff to the column agent.

| From ↓ \ To →         | context7 | Backend | Frontend | architect | Full-Stack | code-reviewer | test-writer | SW Engineer | Foundry | Product Owner | UI Interpreter | Terraform | AWS Cloud |
| --------------------- | -------- | ------- | -------- | --------- | ---------- | ------------- | ----------- | ----------- | ------- | ------------- | -------------- | --------- | --------- |
| **architect**         | ✅       | ✅      | ✅       | —         | —          | ✅            | —           | —           | —       | ✅            | —              | —         | —         |
| **backend-engineer**  | ✅       | —       | ✅       | ✅        | —          | ✅            | ✅          | —           | —       | —             | —              | —         | —         |
| **frontend-engineer** | ✅       | ✅      | —        | —         | —          | ✅            | ✅          | —           | —       | —             | —              | —         | —         |
| **Full-Stack Engineer**| ✅      | —       | —        | ✅        | —          | ✅            | ✅          | —           | —       | —             | —              | —         | —         |
| **test-writer**       | ✅       | —       | —        | —         | —          | ✅            | —           | —           | —       | —             | —              | —         | —         |
| **Product Owner**     | —        | —       | —        | ✅        | —          | —             | —           | ✅          | —       | —             | —              | —         | —         |
| **UI Interpreter**    | —        | ✅      | ✅       | —         | —          | ✅            | ✅          | —           | —       | —             | —              | —         | —         |
| **terraform-expert**  | ✅       | —       | —        | ✅        | —          | ✅            | —           | ✅          | —       | —             | —              | —         | ✅        |
| **aws-cloud-expert**  | ✅       | —       | —        | ✅        | —          | ✅            | —           | ✅          | —       | —             | —              | ✅        | —         |

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
| **Tier 1** | Heavy reasoning — design, judgment, correctness | `Claude Opus 5.5 (copilot)`, `GPT-6.1 Sol (copilot)` | architect, code-reviewer |
| **Tier 2** | Standard execution — writing and modifying code | `Claude Sonnet 5.5 (copilot)`, `GPT-6.1 Terra (copilot)`, `Gemini 3.8 Flash (copilot)` | backend-engineer, frontend-engineer, Full-Stack Engineer, software-engineer-agent-v1, terraform-expert, aws-cloud-expert, Product Owner, Foundry |
| **Tier 3** | Low-latency routing — retrieval, pruning, extraction, scaffolding | `Claude Haiku 5.5 (copilot)`, `GPT-6 Luna (copilot)`, `Gemini 3.8 Flash (copilot)` | Context Manager, context7, test-writer, Handoff Agent, UI Interpreter |

Rules:

- Never assign a Tier 1 model to a Tier 3 agent — retrieval and extraction do not need frontier reasoning.
- Model strings use the exact `Model Name (vendor)` format. An unrecognized string is dropped silently by VS Code and the agent falls back to the default — verify against the model picker after any change.
- RUG itself is exempt from the tier table; it delegates rather than reasons deeply, but must retain enough capability to decompose and validate.
