# Agent Audit — 2026-11-03

Scope: 16 agents in `.github/agents/`, `.github/prompts/agent-audit.prompt.md`, and `.github/skills/local-routing/SKILL.md` (model tier policy). No `chatmodes/`, `instructions/`, `copilot-instructions.md`, or `AGENTS.md` exist.

## Sources (fetched 2026-11-03)

- Supported models: https://docs.github.com/en/copilot/reference/ai-models/supported-models
- Models and pricing (per 1M tokens): https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing
- Usage-based billing: https://docs.github.com/en/copilot/concepts/billing-and-usage/individuals/billing
- Legacy multipliers (annual plans only): https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/model-multipliers-for-annual-plans

Key fact: since 2026-06-01 Copilot bills by tokens in AI credits (1 credit = $0.01). Premium-request multipliers are legacy and cover none of this repo's models.

## Changes made

| Area | Change |
| --- | --- |
| Invalid model | `Gemini 3.6 Flash` (not a supported model) replaced: context-manager and handoff → `Claude Haiku 5.5`; context7 and test-writer → `Gemini 3.8 Flash` |
| Newer versions | architect, code-reviewer, rug-orchestrator: Opus 5 → `Claude Opus 5.5` ($4/$20 vs $5/$25). backend, frontend, full-stack, software-engineer, terraform, aws: Sonnet 5 → `Claude Sonnet 5.5` (same price, cheaper cached input) |
| Cost fit | ui-interpreter: Opus 5 → `Gemini 3.8 Flash` (was a Tier 3 agent on a frontier model). product-owner, foundry: Opus 5 → `Claude Sonnet 5.5` |
| Tools | Removed unused `vscode/extensions`, `vscode/getProjectSetupInfo`, `vscode/runCommand`, `read/terminalSelection`, `agent/runSubagent` (where no body reference or `agents:` list), `azure-mcp/search`, `vscode/installExtension`, `vscode/newWorkspace`, `browser`; narrowed foundry's `vscode` wildcard to `vscode/askQuestions`; architect lost terminal/execute tools |
| rug-orchestrator | Removed §16 (HTML build verbosity rules); merged duplicate "Routing" headings; renumbered §13–15 and updated the one cross-reference. 36.4 KB → 34.6 KB |
| context7 | Condensed 931 lines / 27 KB → ~6.9 KB; kept the critical rule, resolve/fetch workflow, version-check procedure, registry URLs, upgrade template, quality gates and checklist; removed per-language catalogs and long examples |
| local-routing | Model tier table updated to supported models (Product Owner and Foundry moved to Tier 2); removed commented-out template blocks (which also split two tables) |
| agent-audit.prompt | Pinned `Claude Sonnet 5.5`; replaced premium-request multiplier wording with usage-based pricing |

## Not changed (by design)

- RUG keeps `edit/*` and `execute/*` tools; removing them would enforce pure delegation but changes behavior.
- No `copilot-instructions.md` created: cross-agent duplication is only ~1.5 KB and the file would load on every chat request.
- `code-reviewer`, `test-writer`, `software-engineer-agent-v1` bodies (200–325 lines) not trimmed.

## Re-check next quarter

- Confirm each `model:` string appears in the VS Code model picker (an unrecognized string is dropped silently), especially `Claude Haiku 5.5`, `Gemini 3.8 Flash`, and the `.5` Claude versions.
- Spot-check behavior of agents that changed tier: ui-interpreter (image input), product-owner and foundry (Sonnet vs Opus quality), context7 (condensed prompt).
- Re-fetch the supported-models and pricing pages; Gemini 3.8 Flash and Gemini 4 Argon carry promo footnotes that may change prices.
- Consider trimming the largest remaining bodies (rug-orchestrator ~35 KB, code-reviewer, test-writer, software-engineer-agent-v1) and the routing skills (~25 KB loaded per RUG session).
- Confirm no agent still needs `agent/runSubagent` or the removed tools in practice.
