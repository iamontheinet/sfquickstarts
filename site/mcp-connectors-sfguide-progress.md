# MCP Connectors SFGuide Progress

## Repos
- **sfquickstarts repo**: `/Users/jreini/Desktop/development/git-sfc/sfquickstarts`
- **Source repo**: `/Users/jreini/Desktop/development/git-sfc/sfguide-manage-specialized-mcp-agents`
- **Branch**: `josh/mcp-connectors-dev-guide`
- **PR**: https://github.com/Snowflake-Labs/sfquickstarts/pull/3156
- **Fork remote**: `fork` → `https://sfc-gh-jreini@github.com/sfc-gh-jreini/sfquickstarts.git`

## Key Files
- **Guide**: `site/sfguides/src/sfguide-manage-specialized-mcp-agents/sfguide-manage-specialized-mcp-agents.md`
- **Setup SQL**: `site/sfguides/src/sfguide-manage-specialized-mcp-agents/assets/setup.sql`

## Push Command (SSH/HTTPS workaround)
```bash
GIT_SSH_COMMAND="none" GIT_TERMINAL_PROMPT=0 git -c "credential.https://github.com.helper=!gh auth git-credential" -c "url.https://github.com/.insteadOf=ssh://git@github.com/" push fork josh/mcp-connectors-dev-guide --force-with-lease
```

## Guide Status (as of 2026-04-09)
- Title: "Getting Started with MCP Connectors in Snowflake Intelligence"
- Author: Josh Reini
- Enterprise monolith: REMOVED entirely
- Cortex Search Services section: REMOVED (implementation detail)
- Framing: Connecting SI to external + Snowflake MCP servers (not "fictional company")
- RBAC: Framed as inherent governance, not sequential step
- External MCP connectors listed first in learn bullets
- Deslopified
- sfguide style validated (UTM params, Congratulations opening, H2 lengths, etc.)
- 3 agents (HR, Finance, IT), 10 MCP servers, 2 RBAC roles

## Pending
- Source repo changes uncommitted (deleted files, updated README, modified setup.sql)
- setup.sql still has enterprise_server/enterprise_agent -- may need cleanup to match guide
