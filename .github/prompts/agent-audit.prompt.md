---
description: Quarterly audit of my Copilot custom agents — models, cost, and instruction efficiency
agent: agent
model: Claude Sonnet 5.5 (copilot)
---

Goal: Audit and update every Copilot custom agent and prompt file in this workspace so each
uses the right current model and minimizes AI-credit usage. Read-only until I approve.

1. INVENTORY
   Find all Copilot customization files:
   - .github/agents/*.agent.md (custom agents) and .github/chatmodes/*.chatmode.md (older format)
   - .github/prompts/*.prompt.md
   - .github/copilot-instructions.md, .github/instructions/*.instructions.md, AGENTS.md
   Output a table: File | Type | Purpose (1 line) | `model` value (or "none – uses picker") |
   Tools listed | Approx instruction length.

2. MODEL CHECK
   Fetch the current official GitHub docs for (a) Copilot supported models and
   (b) usage-based per-token model pricing (Models and pricing page; premium request
   multipliers are legacy and apply only to old annual plans). Do not rely on training data; cite URL + date.
   For each agent/prompt flag:
   - model names that are retired, renamed, or not available in Copilot
   - newer versions available
   - fit vs cost: simple tasks (formatting, lookup, boilerplate) → a lightweight, low per-token-price model;
     complex reasoning/multi-file work → a stronger model only where it earns the extra cost
   - if no `model` is set, whether pinning one would reduce AI-credit usage

3. EFFICIENCY REVIEW
   For each file identify: bloated or duplicated instructions, content repeated across agents
   that belongs in copilot-instructions.md or a shared *.instructions.md, tools listed but not
   needed (fewer tools = less context), vague instructions that cause retries, and missing
   output format/length guidance. Check frontmatter matches the current documented format.

4. PLAN — THEN STOP
   Show a prioritized table: File | Change | Why | Est. AI-credit impact | Risk.
   Show the proposed diff for each file. Wait for my explicit approval.

5. APPLY (approved items only)
   Edit the files. Never delete an agent. Preserve each agent's purpose and behavior.

6. SUMMARY
   Create docs/agent-audit-YYYY-MM-DD.md: changes made, sources, and what to re-check next quarter.