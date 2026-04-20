# Plan: Clarify Assets Access with Full GitHub URLs

## Context
The guide is rendered as a web page on quickstarts.snowflake.com. Users cannot access `assets/` as a relative path. All file references must be full GitHub URLs pointing to:
**https://github.com/Snowflake-Labs/sfguide-end-to-end-external-ai-agent-observability**

The assets/ folder contents will live at the root of that repo (i.e., `setup_snowflake.sql`, `src/`, `server.py`, etc. are at repo root, not inside an `assets/` subfolder).

## Changes

### 1. Metadata header
Add: `fork repo link: https://github.com/Snowflake-Labs/sfguide-end-to-end-external-ai-agent-observability`

### 2. New "Get the Source Code" subsection in Setup
Add as the FIRST subsection (before "Create Database and Sample Data"):
- `git clone` command
- Full directory tree of the repo
- Brief descriptions of key files/folders

### 3. Replace ALL vague `assets/` references with full GitHub URLs
Every file reference throughout the guide gets updated:

| Current text | New text |
|---|---|
| "run the `setup_snowflake.sql` script from the `assets/` folder" | link to `https://github.com/.../blob/main/setup_snowflake.sql` |
| "Upload `semantic_model.yaml` from the assets folder" | link to `https://github.com/.../blob/main/semantic_model.yaml` |
| "Copy the `assets/` folder to a working directory" | "Navigate into the cloned repository" |
| "`src/services/config.py`" | link to `https://github.com/.../blob/main/src/services/config.py` |
| "`src/agent/tools.py`" | link to full URL |
| "`src/agent/app.py`" | link to full URL |
| "`src/observability/trulens_setup.py`" | link to full URL |
| "`src/eval/ground_truth.py`" | link to full URL |
| "`src/eval/metrics.py`" | link to full URL |
| "`run_eval.py`" | link to full URL |
| "`server.py`" | link to full URL |
| "`monitoring_dashboard/streamlit_app.py`" | link to full URL |
| "`monitoring_dashboard/snowflake.yml`" | link to full URL |
| "`frontend/`" references | link to full URL |

### 4. Section headers with file references
Update headers like `### Connection Config (\`src/services/config.py\`)` to include the GitHub link in the parenthetical.
