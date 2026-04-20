# Getting Started with the `developer-advocate` Profile

A shared Cortex Code profile for the devrel team. It sets a consistent system prompt, voice, and tooling baseline so we all get the same behavior out of the agent.

## Prerequisites

- Access to the `SFDEVREL-ENTERPRISE` Snowflake account (ask your team lead if you need access)
- [Cortex Code CLI](https://docs.snowflake.com/en/developer-guide/cortex-code/) installed
- A Snowflake connection named `DEVREL_ENTERPRISE` configured in your `~/.snowflake/connections.toml`

If you don't have the connection yet, add this to `~/.snowflake/connections.toml`:

```toml
[DEVREL_ENTERPRISE]
account = "SFDEVREL-ENTERPRISE"
user = "<your-username>"
authenticator = "externalbrowser"
role = "ACCOUNTADMIN"
warehouse = "COMPUTE"
```

## Setup

### 1. Add the profile

```bash
cortex profile add developer-advocate -c DEVREL_ENTERPRISE
```

This fetches the profile definition and skills from Snowflake to your local machine.

### 2. Start a session with the profile

```bash
cortex --profile developer-advocate
```

You're now running with the shared system prompt and all published skills.

### 3. (Optional) Set as default

If you want every session to use this profile automatically:

```bash
cortex profile set-default developer-advocate
```

Then just `cortex` starts with the profile applied.

## Staying Up to Date

When someone updates the system prompt or adds a new skill, pull the latest:

```bash
cortex profile sync developer-advocate -c DEVREL_ENTERPRISE
```

## Available Skills

### `$update-profile-system-prompt`

Edit and republish the profile's system prompt. Run it inside a session:

```
$update-profile-system-prompt
```

It will:
1. Download the current system prompt from the stage
2. Show it to you
3. Let you make edits
4. Upload the updated version back to Snowflake

Everyone gets the change on their next sync.

### `$add-profile-skill`

Create a new skill and wire it into the profile. Run it inside a session:

```
$add-profile-skill
```

It will ask you:
- What the skill should be called (e.g., `quickstart-author`)
- What it should do

Then it:
1. Writes the SKILL.md file
2. Shows it for your approval
3. Uploads it to the shared stage
4. Updates the profile registry
5. Bumps the version

After that, everyone gets the new skill on their next sync.

### `$unblock-profile-registry`

Diagnose and fix lock contention on the profile registry table. Run it inside a session:

```
$unblock-profile-registry
```

It will:
1. Check for active locks on the registry
2. Identify the blocking query and who owns it
3. Show you what's going on (stuck warehouse, stale transaction, etc.)
4. Cancel the blocking query (with your approval)
5. Verify the lock is released
6. Preserve any skills the blocked session was trying to add

## How It Works (Reference)

The profile is stored in the `DEVREL_ENTERPRISE` account:

| Component | Location |
|---|---|
| Registry | `CORTEX_CODE.CONFIG.PROFILE_REGISTRY` |
| System prompt | `@CORTEX_CODE.CONFIG.PROFILE_ASSETS/developer-advocate/system_prompt.md` |
| Skills | `@CORTEX_CODE.CONFIG.PROFILE_ASSETS/skills/<skill-name>/SKILL.md` |

The registry table tracks the profile definition (name, description, version, skill repos, system prompt location). The stage holds the actual files. The CLI reads the registry to know where to fetch everything.

## Troubleshooting

**"Missing required argument: connection"**
Add `-c DEVREL_ENTERPRISE` to the command.

**Profile not updating after someone made changes**
Run `cortex profile sync developer-advocate -c DEVREL_ENTERPRISE` to pull the latest.

**Skills not showing up**
Make sure you synced after the skill was added. Run `cortex profile sync` and try `$skill-name` again.

**Connection not found**
Check that `DEVREL_ENTERPRISE` exists in your `~/.snowflake/connections.toml` and that you can authenticate.

**Profile registry UPDATE hangs or times out**
Run `$unblock-profile-registry` inside a session. It will check for locks, identify the blocking query, cancel it (with your approval), and make sure any skills the blocked session was trying to add aren't lost.
