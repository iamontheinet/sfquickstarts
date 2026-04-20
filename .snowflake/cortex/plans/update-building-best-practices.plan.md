# Plan: Update Building Best Practices Guide

## Overview
Update the published "Best Practices for Building Cortex Agents" guide with:
1. Four new features: **Agent Versioning**, **Code Execution**, **Web Search**, **MCP Connectors**
2. Slimmed-down evaluation/deployment sections that defer to the dedicated eval guide

## Current Guide Structure (802 lines)
```
L1-31:   Metadata + Overview + What You'll Learn
L32-51:  How Snowflake Intelligence Works
L53-97:  Building Cortex Agents (purpose, scope, tool mapping)
L98-118: Importance of Cortex Agent Instructions
L120-127: Semantic Views
L128-228: Orchestration Instructions
L229-293: Response Instructions
L296-314: Orchestration vs Response table
L315-400: Tool Descriptions
L401-549: Using Cortex Tools (Cortex Analyst, Cortex Search, example questions)
L551-596: Deploying Your Agent to Production (3 stages)
L597-606: How to Improve Agent Performance
L608-791: Example: Complete Agent Configuration
L792-802: Additional Resources
```

## Changes

### 1. Create new branch
- `git checkout master && git pull origin master && git checkout -b josh/building-best-practices-tools`

### 2. Update "What you'll learn" (L23-29)
Add bullets:
- "How to use built-in tools including code execution, web search, and MCP connectors."
- "How to use agent versioning to manage your deployment lifecycle."

### 3. Add new tools to "Using Cortex Tools" section (after ~L537)

After the Cortex Search parameters best practices, add three new subsections:

#### a. Code Execution
- **What it does**: Agents generate and run Python in a sandboxed environment for calculations, data processing, visualizations
- **Spec snippet**: `tools: [{tool_spec: {type: code_execution, name: code_execution}}]` + `tool_resources: {code_execution: {}}`
- **Best practices**: Grant PyPI only when needed, scope external access integrations to specific domains, remember owner role inheritance, session-scoped sandbox

#### b. Web Search
- **What it does**: Agents search the web via Brave Search API for real-time information
- **Prerequisites**: Account admin enables via Snowsight (AI & ML → Agents → Settings)
- **Best practices**: Use for real-time info your internal data doesn't cover, combine with orchestration instructions, ZDR with Brave, specify when to prefer internal vs web tools

#### c. MCP Connectors (with Private Preview note)
- `> **Preview Feature — Private:** MCP Connectors are available to select accounts.`
- **What it does**: Connects agents to SaaS tools (Atlassian, GitHub, Glean, Linear, Salesforce) via MCP
- **Setup flow**: Provider OAuth → API Integration → External MCP Server → Add to Agent
- **Best practices**: Least-privilege, descriptive names, disable vs drop, hostname hyphens, orchestration instructions for external vs internal

### 4. Add Agent Versioning to "Deploying" section (~L551-596)

Add a concise versioning subsection after the intro paragraph (before Stage 1). Keeps it brief since the eval guide covers versioning in depth.

- `> **Preview Feature — Private:** Agent versioning is available to select accounts.`
- Core concept: Live version (mutable) vs named versions (immutable snapshots) vs aliases (deployment routing)
- Key SQL: `ALTER AGENT <name> COMMIT`, `MODIFY VERSION SET ALIAS = production`
- Brief mapping to the 3-stage process: prototype (live) → evaluate (committed) → production (alias)

### 5. Slim down evaluation content in Stages 2 & 3 (~L566-595)

The current building guide has significant eval detail (golden sets, monitoring UI, GPA framework, eval setup, monitoring setup) that overlaps heavily with the dedicated [Evaluating Cortex Agents guide](https://www.snowflake.com/en/developers/guides/best-practices-to-evaluating-cortex-agents/). Replace the detailed content with a concise summary and a prominent link.

**Current Stage 2 (L566-583)** — Replace with:
- Keep the high-level concept: use evaluation to drive iteration  
- Add link: 👉 For a comprehensive guide on evaluation datasets, metrics, and CI/CD, see [Best Practices for Evaluating Cortex Agents](https://www.snowflake.com/en/developers/guides/best-practices-to-evaluating-cortex-agents/)
- Keep the GPA framework blog link as a resource

**Current "Setup agent evals" subsection (L570-581)** — Slim to 2-3 sentences + link to eval guide

**Current Stage 3 (L585-595)** — Replace with:
- Keep the high-level concept: monitor production usage, collect feedback, run regular evals
- Add link to eval guide for detailed observability, alerts, and scheduled evaluation setup
- Remove the detailed monitoring paragraphs (now covered in eval guide's "Production Observability" section)

### 6. Slim down "How to improve agent performance" (L597-606)
- Keep the 3 bullet points (they're already concise)
- Add link to eval guide for detailed iteration workflows

### 7. Update "Additional resources" (L797-802)
- Add link: [Best Practices for Evaluating Cortex Agents](https://www.snowflake.com/en/developers/guides/best-practices-to-evaluating-cortex-agents/)
- Add link: [Code execution tool documentation](https://docs.snowflake.com/en/LIMITEDACCESS/cortex-agents-code-interpreter)
- Add link: [MCP Connectors documentation](https://docs.snowflake.com/en/LIMITEDACCESS/snowflake-cortex/mcp-connectors)

### 8. Push and create PR
- Push branch to fork
- Create PR against Snowflake-Labs/sfquickstarts with summary of all changes
