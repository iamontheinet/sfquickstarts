# Plan: End-to-End External AI Agent Observability Guide

## Summary

Create a new Snowflake developer guide at `site/sfguides/src/end-to-end-external-ai-agent-observability/` that teaches developers how to build, instrument, batch-test, and production-monitor an external AI agent. The guide markdown references code files that ship in the `assets/` subfolder, so readers can follow along without cloning an external repo.

## File Inventory

All files live under `site/sfguides/src/end-to-end-external-ai-agent-observability/`:

```
end-to-end-external-ai-agent-observability.md   # The guide itself

assets/
  readme.md                          # Required placeholder per sfquickstarts convention

  # Root-level project files
  setup_snowflake.sql                # Snowflake DDL (database, tables, search service, ground truth)
  semantic_model.yaml                # Cortex Analyst semantic model for TICKET_METRICS
  pyproject.toml                     # Python dependencies (uv-managed)
  .gitignore                         # Standard ignores

  # Entry points
  run_eval.py                        # Batch evaluation runner
  server.py                          # FastAPI production server with @trace_with_run

  # src/ package
  src/__init__.py
  src/agent/__init__.py
  src/agent/app.py                   # AgentApp + SnowflakeAsyncOpenAI wrapper + Agent definition
  src/agent/tools.py                 # @function_tool + @instrument tools (Analyst + Search)
  src/services/__init__.py
  src/services/config.py             # Snowflake connection params and session factory
  src/eval/__init__.py
  src/eval/ground_truth.py           # Ground truth loaders + test query suites
  src/eval/metrics.py                # All metric implementations + build_metrics() factory
  src/eval/sql_result_comparator.py  # Multi-strategy SQL result-set comparator
  src/observability/__init__.py
  src/observability/trulens_setup.py # TruLens setup: TruApp, SnowflakeConnector, metrics

  # React frontend
  frontend/index.html
  frontend/package.json
  frontend/tsconfig.json
  frontend/vite.config.ts
  frontend/src/main.tsx
  frontend/src/App.tsx
  frontend/src/index.css

  # Streamlit monitoring dashboard
  monitoring_dashboard/streamlit_app.py
  monitoring_dashboard/snowflake.yml
  monitoring_dashboard/pyproject.toml
```

Total: **1 markdown guide + ~27 source files** in assets.

## Guide Structure (10 Sections)

### Section 1: Overview
- Architecture diagram (ASCII art)
- What You Will Learn (7 bullets covering full lifecycle)
- What You Will Build (agent, batch eval, production server, monitoring dashboard)
- Prerequisites (Snowflake account, Python 3.12+, uv, Node.js 18+, Snowflake CLI v3.14.0+)

### Section 2: Setup Environment
- Copy assets/ contents to a working directory
- Run `setup_snowflake.sql` to create database, tables, Cortex Search service, ground truth tables
- Upload semantic model to stage
- Install Python + frontend deps

### Section 3: Build the Agent
- Walk through `src/agent/app.py` and `src/agent/tools.py`
- SnowflakeAsyncOpenAI wrapper, Agent definition, tool implementations
- Cortex Analyst (structured) + Cortex Search (unstructured) tools

### Section 4: Instrument with TruLens
- Walk through `src/observability/trulens_setup.py` and `@instrument` decorators
- Span types: RECORD_ROOT, TOOL, RETRIEVAL, GENERATION
- Show the span tree structure and attribute mapping

### Section 5: Define Ground Truth and Metrics
- Walk through `src/eval/ground_truth.py` and `src/eval/metrics.py`
- Ground truth in Snowflake tables (not hardcoded)
- Server-side metrics (4 string names) + custom client-side metrics (3 Metric objects)
- The Metric + Selector API pattern

### Section 6: Run Batch Evaluation
- Walk through `run_eval.py`
- RunConfig, add_run, run.start, polling, compute_metrics
- Critical pattern: single compute_metrics() call for all metrics

### Section 7: Production Monitoring with Live Tracing
- Walk through `server.py`
- FastAPI endpoints with @trace_with_run
- Background metrics computation
- React chat frontend

### Section 8: Deploy Monitoring Dashboard
- Walk through `monitoring_dashboard/`
- Streamlit-in-Snowflake deployment
- Dashboard features walkthrough (KPI row, latency, evals, trace explorer)

### Section 9: Navigating AI Observability in Snowsight
- Built-in Snowsight UI for viewing traces, runs, and evaluation scores

### Section 10: Conclusion and Resources
- What You Learned summary
- Links to docs, TruLens, OpenAI Agent SDK, source repo

## Implementation Order

1. Create all `assets/` source code files (download from GitHub branch)
2. Write the guide markdown with all 10 sections
3. Verify file structure matches sfquickstarts conventions
