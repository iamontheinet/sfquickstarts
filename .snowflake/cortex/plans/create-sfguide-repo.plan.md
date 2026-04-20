# Plan: Create sfguide-end-to-end-external-ai-agent-observability repo

## Overview
Move all files from `assets/` into a new standalone repo at `/Users/jreini/Desktop/development/git-sfc/sfguide-end-to-end-external-ai-agent-observability/`, init git, and update the guide markdown.

## Repo structure (files at root, no assets/ prefix)
```
sfguide-end-to-end-external-ai-agent-observability/
├── setup_snowflake.sql
├── semantic_model.yaml
├── pyproject.toml
├── run_eval.py
├── server.py
├── .gitignore
├── README.md
├── src/
│   ├── __init__.py
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── app.py
│   │   └── tools.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── config.py
│   ├── eval/
│   │   ├── __init__.py
│   │   ├── ground_truth.py
│   │   ├── metrics.py
│   │   └── sql_result_comparator.py
│   └── observability/
│       ├── __init__.py
│       └── trulens_setup.py
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       └── index.css
└── monitoring_dashboard/
    ├── streamlit_app.py
    ├── snowflake.yml
    └── pyproject.toml
```

## Steps

1. Create repo directory structure
2. Copy all 29 files from assets/ to repo root (stripping the assets/ prefix)
3. Update README.md to be a proper repo README (not placeholder)
4. `git init` the repo
5. Update the sfquickstarts guide markdown:
   - Add `fork repo link:` to metadata
   - Add "Get the Source Code" subsection with clone instructions + directory tree
   - Replace all relative `assets/` references with full GitHub URLs
6. Optionally delete the now-redundant assets/ folder from sfquickstarts
