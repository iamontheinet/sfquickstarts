author: Josh Reini
id: sfguide-manage-specialized-mcp-agents
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/ai
language: en
summary: Build domain-scoped Cortex Agents backed by Snowflake-managed MCP servers and external MCP connectors, all governed via Snowflake RBAC.
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfguides/issues
fork repo link:

# Getting Started with MCP Connectors in Snowflake Intelligence
<!-- ------------------------ -->
## Overview

Connect Cortex Agents to **Snowflake-managed MCP servers** (Cortex Search, Cortex Analyst) and **external MCP connectors** (Atlassian, GitHub, Glean, Linear, Salesforce). Data, tools, MCP servers, and agents are all governed in Snowflake via RBAC.

This guide uses sample data across HR, Finance, and IT schemas to create Cortex Agents backed by Snowflake-managed MCP servers, with optional external MCP connectors for SaaS tools. You'll interact with all of them through **Snowflake Intelligence**.

```
Snowflake Intelligence (MCP Client)
  |
  +-- HR Assistant          --> handbook_comp_server, benefits_server, org_server
  +-- Finance Assistant     --> budget_server, product_usage_server, invoice_search_server, ...
  +-- IT Ops Assistant      --> incident_server, infra_monitor_server
  |
  +-- External MCP Connectors (optional): Atlassian | GitHub | Glean | Linear | Salesforce
```

### What You'll Build

| Object | Count | Details |
|---|---|---|
| Cortex Agents | 3 | `hr_agent`, `finance_agent`, `it_agent` |
| MCP Servers (internal) | 10 | Cortex Search + Cortex Analyst tools |
| External MCP Connectors | 0+ (optional) | Atlassian, GitHub, Glean, Linear, and/or Salesforce |
| Cortex Search Services | 3 | Employee handbook, IT incidents, invoices |
| Semantic Views | 8 | Compensation, benefits, org, budget, product usage, spend, P&L, infra, SLA |
| Tables | 16 | Across HR, Finance, and IT schemas |
| RBAC Roles | 2 | `hr_analyst_role`, `finance_analyst_role` |

### What You'll Learn

- How to create **external MCP connectors** for SaaS providers via OAuth
- How to create **Snowflake-managed MCP servers** with Cortex Search and Cortex Analyst tools
- How to wire both types of MCP servers into **Cortex Agents**
- How **RBAC** governs which roles can access which agents and MCP servers

### Prerequisites

- A [Snowflake account](https://signup.snowflake.com/?utm_source=snowflake-devrel&utm_medium=developer-guides&utm_cta=developer-guides) with `ACCOUNTADMIN` (or a role with `CREATE DATABASE`, `CREATE ROLE`, `CREATE INTEGRATION` privileges)
- A warehouse (this guide uses `COMPUTE`)
- Access to [Snowflake Intelligence](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-intelligence/getting-started)
- _(Optional)_ For external connectors: an account with one or more SaaS providers (Atlassian, GitHub, Glean, Linear, or Salesforce)

### Architecture

There are two kinds of MCP servers in this guide:

| | Snowflake-Managed (Internal) | External MCP Connector |
|---|---|---|
| Created with | `CREATE MCP SERVER ... FROM SPECIFICATION` | `CREATE API INTEGRATION` + `CREATE EXTERNAL MCP SERVER` |
| Tools | Cortex Search, Cortex Analyst, Execute SQL | Discovered from remote MCP endpoint (`tools/list`) |
| Auth | Snowflake RBAC | OAuth (user authenticates in Snowflake Intelligence) |
| Use case | Query your Snowflake data | Connect to SaaS tools (Jira, GitHub, Salesforce, etc.) |

<!-- ------------------------ -->
## Foundation Setup

Create the `ACME_CORP` database with 16 tables across three domain schemas.

### Create Database and Schemas

```sql
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE;

CREATE DATABASE IF NOT EXISTS ACME_CORP;
CREATE SCHEMA IF NOT EXISTS ACME_CORP.HR;
CREATE SCHEMA IF NOT EXISTS ACME_CORP.FINANCE;
CREATE SCHEMA IF NOT EXISTS ACME_CORP.IT;
```

### Create Tables

The full table definitions and sample data are in [setup.sql](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql). Run the table creation section (Step 1) to get all 16 tables populated:

| Schema | Tables |
|---|---|
| `HR` | `employees`, `compensation_bands`, `handbook_docs`, `org_chart`, `benefits_plans`, `benefits_enrollments` |
| `FINANCE` | `budgets`, `expenses`, `financial_reports`, `spend_approvals`, `product_usage`, `invoices` |
| `IT` | `incidents`, `services`, `sla_records`, `infrastructure_assets` |

> **Tip:** You can run the entire [setup.sql](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql) file at once in a Snowsight SQL worksheet to create all objects in one shot.

<!-- ------------------------ -->
## Semantic Views

Create semantic views to enable natural-language-to-SQL via Cortex Analyst. The setup script creates 8 semantic views. Here are two examples:

### Compensation (HR)

```sql
CREATE OR REPLACE SEMANTIC VIEW ACME_CORP.HR.comp_semantic_view
  TABLES (
    employees AS ACME_CORP.HR.employees PRIMARY KEY (employee_id),
    compensation_bands AS ACME_CORP.HR.compensation_bands PRIMARY KEY (band_id)
      UNIQUE (level, department)
  )
  RELATIONSHIPS (
    employees (level, department) REFERENCES compensation_bands (level, department)
  )
  FACTS (
    employees.salary AS salary,
    compensation_bands.min_salary AS min_salary,
    compensation_bands.max_salary AS max_salary
  )
  DIMENSIONS (
    employees.name AS name COMMENT = 'Full name of the employee',
    employees.department AS department COMMENT = 'Department',
    employees.title AS title COMMENT = 'Job title',
    employees.level AS level COMMENT = 'Career level from L4 to L9'
  )
  METRICS (
    employees.headcount AS COUNT(employee_id) COMMENT = 'Total number of employees',
    employees.average_salary AS AVG(salary) COMMENT = 'Average employee salary',
    employees.total_payroll AS SUM(salary) COMMENT = 'Total salary expenditure'
  );
```

### Budgets (Finance)

```sql
CREATE OR REPLACE SEMANTIC VIEW ACME_CORP.FINANCE.budget_semantic_view
  TABLES (
    budgets AS ACME_CORP.FINANCE.budgets PRIMARY KEY (budget_id),
    expenses AS ACME_CORP.FINANCE.expenses PRIMARY KEY (expense_id)
  )
  FACTS (
    budgets.allocated_amount AS allocated_amount,
    budgets.spent_amount AS spent_amount,
    expenses.amount AS amount
  )
  DIMENSIONS (
    budgets.department AS department COMMENT = 'Department name',
    budgets.fiscal_quarter AS fiscal_quarter COMMENT = 'Fiscal quarter (Q1-Q4)',
    expenses.category AS category COMMENT = 'Expense category'
  )
  METRICS (
    budgets.total_allocated AS SUM(allocated_amount) COMMENT = 'Total budget allocated',
    budgets.total_spent AS SUM(spent_amount) COMMENT = 'Total budget spent',
    expenses.total_expenses AS SUM(amount) COMMENT = 'Total expense amount'
  );
```

See [setup.sql](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql) (Step 3) for all 8 semantic view definitions.

<!-- ------------------------ -->
## Internal MCP Servers

MCP servers wrap your Cortex Search services and Semantic Views as discoverable tools for agents.

### Create a Specialized MCP Server

```sql
CREATE OR REPLACE MCP SERVER ACME_CORP.HR.handbook_comp_server
  FROM SPECIFICATION $$
    tools:
      - name: "handbook-search"
        type: "CORTEX_SEARCH_SERVICE_QUERY"
        identifier: "ACME_CORP.HR.handbook_search_svc"
        title: "Employee Handbook Search"
        description: "Search the employee handbook for policies and procedures."

      - name: "compensation-analyst"
        type: "CORTEX_ANALYST_MESSAGE"
        identifier: "ACME_CORP.HR.comp_semantic_view"
        title: "Compensation Analyst"
        description: "Ask natural language questions about employee compensation and salary bands."

      - name: "execute-sql"
        type: "SYSTEM_EXECUTE_SQL"
        title: "Execute SQL"
        description: "Execute SQL queries generated by analyst tools against Snowflake."
  $$;
```

### All MCP Servers

The setup script creates **10 specialized MCP servers** across the three schemas.

| Server | Schema | Tools | Purpose |
|---|---|---|---|
| `handbook_comp_server` | HR | handbook-search, compensation-analyst, execute-sql | Policies + pay |
| `benefits_server` | HR | benefits-cost-analysis, enrollment-stats, execute-sql | Benefits data |
| `org_server` | HR | org-explorer, execute-sql | Org structure |
| `budget_server` | FINANCE | budget-analyst, execute-sql | Budgets + expenses |
| `product_usage_server` | FINANCE | product-analytics, execute-sql | Product metrics |
| `invoice_search_server` | FINANCE | invoice-search | Invoice lookup |
| `spend_approvals_server` | FINANCE | approval-search, spend-analytics, execute-sql | Spend requests |
| `reporting_server` | FINANCE | financial-reporting, execute-sql | P&L reports |
| `incident_server` | IT | incident-search | IT incident search |
| `infra_monitor_server` | IT | infra-health, sla-compliance, execute-sql | Infra + SLAs |

<!-- ------------------------ -->
## External MCP Connectors

This step is **optional**. External MCP connectors let your agents reach into SaaS tools like Jira, GitHub, or Salesforce. You can add **any combination** of the connectors below -- or skip this step entirely and come back to it later.

Each connector follows the same pattern:

1. Set up OAuth with the provider
2. Create an API Integration in Snowflake
3. Create an External MCP Server that references the integration
4. Add it to an agent (next step)

> aside positive
> **Callback URLs:** Your OAuth app configuration needs this redirect URI for Snowflake Intelligence:
> - `<your_snowsight_url>/oauth/complete-secret`
>
> Get the Snowsight URL from `SELECT SYSTEM$ALLOWLIST()` -- pick the entry of type `SNOWSIGHT_DEPLOYMENT` starting with `apps-api`.

### Option A: Atlassian (Jira & Confluence)

Atlassian uses **Dynamic Client Registration (DCR)** -- no client ID/secret needed.

**Provider setup:**
1. Go to [admin.atlassian.com](https://admin.atlassian.com)
2. Navigate to **Apps > AI Settings > Rovo MCP Server**
3. Under **Your domains**, add both callback URLs

**Snowflake setup:**
```sql
CREATE API INTEGRATION atlassian_mcp_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://mcp.jira.atlassian.com')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH_DYNAMIC_CLIENT,
    OAUTH_RESOURCE_URL = 'https://mcp.atlassian.com/v1/mcp'
  )
  ENABLED = TRUE;

CREATE EXTERNAL MCP SERVER ACME_CORP.IT.atlassian_mcp_server
  WITH NAME = 'Atlassian (Jira & Confluence)'
  SERVER_URL = 'https://mcp.atlassian.com/v1/mcp'
  API_INTEGRATION = atlassian_mcp_integration;
```

### Option B: GitHub

**Provider setup:**
1. Go to [github.com/settings/apps](https://github.com/settings/apps) and create an OAuth App
2. Set the callback URLs as the redirect URIs
3. Copy the Client ID and generate a Client Secret

**Snowflake setup:**
```sql
CREATE API INTEGRATION github_mcp_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://api.githubcopilot.com')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH,
    OAUTH_CLIENT_ID = '<your_github_client_id>',
    OAUTH_CLIENT_SECRET = '<your_github_client_secret>',
    OAUTH_TOKEN_ENDPOINT = 'https://github.com/login/oauth/access_token',
    OAUTH_AUTHORIZATION_ENDPOINT = 'https://github.com/login/oauth/authorize'
  )
  ENABLED = TRUE;

CREATE EXTERNAL MCP SERVER ACME_CORP.IT.github_mcp_server
  WITH NAME = 'GitHub'
  API_INTEGRATION = github_mcp_integration;
```

### Option C: Glean

**Provider setup:**
1. In your Glean admin console, register an OAuth application
2. Set the callback URLs as the redirect URIs
3. Copy the Client ID and Client Secret

**Snowflake setup:**
```sql
CREATE API INTEGRATION glean_mcp_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://<your-company>-be.glean.com')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH,
    OAUTH_CLIENT_ID = '<your_glean_client_id>',
    OAUTH_CLIENT_SECRET = '<your_glean_client_secret>',
    OAUTH_TOKEN_ENDPOINT = 'https://<your-company>-be.glean.com/api/v1/oauth/token',
    OAUTH_AUTHORIZATION_ENDPOINT = 'https://<your-company>-be.glean.com/api/v1/oauth/authorize'
  )
  ENABLED = TRUE;

CREATE EXTERNAL MCP SERVER ACME_CORP.IT.glean_mcp_server
  WITH NAME = 'Glean Enterprise Search'
  API_INTEGRATION = glean_mcp_integration;
```

### Option D: Linear

**Provider setup:**
1. Go to [linear.app/settings/api](https://linear.app/settings/api) and create an OAuth application
2. Set the callback URLs as the redirect URIs
3. Copy the Client ID and Client Secret

**Snowflake setup:**
```sql
CREATE API INTEGRATION linear_mcp_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://api.linear.app')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH,
    OAUTH_CLIENT_ID = '<your_linear_client_id>',
    OAUTH_CLIENT_SECRET = '<your_linear_client_secret>',
    OAUTH_TOKEN_ENDPOINT = 'https://api.linear.app/oauth/token',
    OAUTH_AUTHORIZATION_ENDPOINT = 'https://linear.app/oauth/authorize'
  )
  ENABLED = TRUE;

CREATE EXTERNAL MCP SERVER ACME_CORP.IT.linear_mcp_server
  WITH NAME = 'Linear'
  API_INTEGRATION = linear_mcp_integration;
```

### Option E: Salesforce

**Provider setup:**
1. In Salesforce Setup, go to **App Manager** and create a new Connected App
2. Enable OAuth, add the callback URLs as redirect URIs
3. Select the scopes you need (e.g., `api`, `refresh_token`)
4. Copy the Consumer Key (Client ID) and Consumer Secret

**Snowflake setup:**
```sql
CREATE API INTEGRATION salesforce_mcp_integration
  API_PROVIDER = external_mcp
  API_ALLOWED_PREFIXES = ('https://<your-instance>.my.salesforce.com')
  API_USER_AUTHENTICATION = (
    TYPE = OAUTH,
    OAUTH_CLIENT_ID = '<your_salesforce_consumer_key>',
    OAUTH_CLIENT_SECRET = '<your_salesforce_consumer_secret>',
    OAUTH_TOKEN_ENDPOINT = 'https://<your-instance>.my.salesforce.com/services/oauth2/token',
    OAUTH_AUTHORIZATION_ENDPOINT = 'https://<your-instance>.my.salesforce.com/services/oauth2/authorize'
  )
  ENABLED = TRUE;

CREATE EXTERNAL MCP SERVER ACME_CORP.FINANCE.salesforce_mcp_server
  WITH NAME = 'Salesforce'
  API_INTEGRATION = salesforce_mcp_integration;
```

<!-- ------------------------ -->
## Cortex Agents

Create agents that wire together internal MCP servers and (optionally) your external connectors. These agents are accessed through **Snowflake Intelligence**, which acts as the MCP client.

### Domain Agent Example

```sql
CREATE OR REPLACE AGENT ACME_CORP.HR.hr_agent
  COMMENT = 'HR domain agent'
  PROFILE = '{"display_name": "HR Assistant"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-sonnet-4-5"},
    "instructions": {
      "orchestration": "You are an HR assistant for Acme Corp. Use handbook_search for policy questions. Use compensation_analyst for salary questions. Use benefits tools for benefits questions. Use org_explorer for org structure. Present data in markdown tables. Be concise.",
      "sample_questions": [
        {"question": "What is the total employee benefits cost by department?"},
        {"question": "What is the average salary by department?"},
        {"question": "What is the PTO policy?"}
      ]
    },
    "mcp_servers": [
      {"server_spec": {"name": "ACME_CORP.HR.handbook_comp_server"}},
      {"server_spec": {"name": "ACME_CORP.HR.benefits_server"}},
      {"server_spec": {"name": "ACME_CORP.HR.org_server"}}
    ]
  }
  $$;
```

### Adding External Connectors to an Agent

To add external connectors, include them in the `mcp_servers` array alongside the internal servers:

```sql
CREATE OR REPLACE AGENT ACME_CORP.IT.it_agent
  COMMENT = 'IT domain agent with external connector'
  PROFILE = '{"display_name": "IT Ops Assistant"}'
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-sonnet-4-5"},
    "instructions": {
      "orchestration": "You are an IT operations assistant. Use incident_search for incident lookups. Use infra_health and sla_compliance for infrastructure monitoring. Present data in markdown tables. Be concise."
    },
    "mcp_servers": [
      {"server_spec": {"name": "ACME_CORP.IT.incident_server"}},
      {"server_spec": {"name": "ACME_CORP.IT.infra_monitor_server"}},
      {"server_spec": {"name": "ACME_CORP.IT.atlassian_mcp_server"}}
    ]
  }
  $$;
```

> aside negative
> Replace or add to the external MCP server entries with whichever connectors you created in the previous step. If you skipped that step, simply omit the external server entries.

See [setup.sql](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql) for all 3 agent definitions (HR, Finance, IT).

<!-- ------------------------ -->
## RBAC for MCP Servers

Every object in this guide is governed by Snowflake RBAC. The grant chain is:

```
Role  -->  Agent  -->  MCP Server  -->  Tools  -->  Data
```

### Create Roles

```sql
CREATE ROLE IF NOT EXISTS hr_analyst_role;
CREATE ROLE IF NOT EXISTS finance_analyst_role;

GRANT USAGE ON WAREHOUSE COMPUTE TO ROLE hr_analyst_role;
GRANT USAGE ON WAREHOUSE COMPUTE TO ROLE finance_analyst_role;

GRANT USAGE ON DATABASE ACME_CORP TO ROLE hr_analyst_role;
GRANT USAGE ON DATABASE ACME_CORP TO ROLE finance_analyst_role;
```

### HR Analyst Grants

```sql
GRANT USAGE ON SCHEMA ACME_CORP.HR TO ROLE hr_analyst_role;
GRANT SELECT ON ALL TABLES IN SCHEMA ACME_CORP.HR TO ROLE hr_analyst_role;
GRANT USAGE ON CORTEX SEARCH SERVICE ACME_CORP.HR.handbook_search_svc TO ROLE hr_analyst_role;
GRANT SELECT ON SEMANTIC VIEW ACME_CORP.HR.comp_semantic_view TO ROLE hr_analyst_role;
GRANT SELECT ON SEMANTIC VIEW ACME_CORP.HR.benefits_semantic_view TO ROLE hr_analyst_role;
GRANT SELECT ON SEMANTIC VIEW ACME_CORP.HR.org_semantic_view TO ROLE hr_analyst_role;
GRANT USAGE ON MCP SERVER ACME_CORP.HR.handbook_comp_server TO ROLE hr_analyst_role;
GRANT USAGE ON MCP SERVER ACME_CORP.HR.benefits_server TO ROLE hr_analyst_role;
GRANT USAGE ON MCP SERVER ACME_CORP.HR.org_server TO ROLE hr_analyst_role;
GRANT USAGE ON AGENT ACME_CORP.HR.hr_agent TO ROLE hr_analyst_role;
```

### What RBAC Achieves

| Role | Sees | Doesn't See |
|---|---|---|
| `hr_analyst_role` | HR Assistant, HR MCP servers | Finance, IT agents |
| `finance_analyst_role` | Finance Assistant, Finance MCP servers | HR, IT agents |

Restricted roles don't see a "permission denied" error. The agent simply isn't visible in their namespace.

<!-- ------------------------ -->
## Test Your Agents

### Test Internal Tools

1. Open [Snowflake Intelligence](https://app.snowflake.com)
2. Select the **HR Assistant** agent
3. Ask: _"What is the total employee benefits cost by department?"_
4. The agent should use `benefits-cost-analysis` -> `org-explorer` -> `execute-sql` to cross-reference benefits enrollment with org chart data

### Test External Connector

1. Switch to the **IT Ops Assistant** agent (or whichever agent you added external connectors to)
2. Open the **Sources** panel and select **Connectors**
3. Click **Connect** next to your external connector -- you'll be redirected to authenticate with the provider
4. Ask a question that requires the external tool (e.g., _"Show me my open Jira tickets"_ for Atlassian)

### Test RBAC Isolation

1. Grant the roles to your user and disable secondary roles:
```sql
GRANT ROLE hr_analyst_role TO USER <your_username>;
GRANT ROLE finance_analyst_role TO USER <your_username>;
ALTER USER <your_username> SET DEFAULT_SECONDARY_ROLES = ();
```
2. In Snowflake Intelligence, switch to `hr_analyst_role` in the role picker
3. Only the **HR Assistant** is visible -- Finance and IT agents don't appear
4. Switch to `finance_analyst_role` -- only the **Finance Assistant** is visible

<!-- ------------------------ -->
## Teardown

To drop all demo artifacts, uncomment and run the teardown section at the bottom of [setup.sql](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql). This drops agents, MCP servers, semantic views, search services, tables, schemas, and roles.

<!-- ------------------------ -->
## Conclusion And Resources

Congratulations! You've successfully connected Snowflake Intelligence to Cortex Agents backed by Snowflake-managed MCP servers and external MCP connectors, all governed via Snowflake RBAC.

### What You Learned

- How to create external MCP connectors using OAuth (standard and DCR)
- How to create Snowflake-managed MCP servers with Cortex Search and Cortex Analyst tools
- How to wire internal and external MCP servers into Cortex Agents
- How Snowflake RBAC governs data, tools, MCP servers, and agents

### Key Takeaways

1. Snowflake-managed MCP servers and external connectors can be combined in a single agent
2. `GRANT USAGE ON MCP SERVER` governs tool access at the role level. Restricted roles don't see tools they shouldn't use
3. One API integration + one external MCP server = your agent can query Jira, GitHub, or Salesforce alongside Snowflake data

### Related Resources

- [MCP Connectors](https://docs.snowflake.com/en/LIMITEDACCESS/snowflake-cortex/mcp-connectors) -- External MCP connector docs
- [CREATE MCP SERVER](https://docs.snowflake.com/en/sql-reference/sql/create-mcp-server) -- SQL reference
- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents) -- Agent orchestration
- [Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/overview) -- Semantic models for Cortex Analyst
- [Cortex Search](https://docs.snowflake.com/en/sql-reference/sql/create-cortex-search) -- Unstructured search services
- [Setup Script](https://github.com/Snowflake-Labs/sfquickstarts/blob/master/site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql) -- Complete setup SQL with teardown at the bottom
