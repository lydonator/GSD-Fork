<overview>
Plugins extend GSD functionality through self-contained packages. This reference defines the plugin manifest schema, folder structure, and registration patterns.

**Key insight:** A plugin is a self-contained folder with a `plugin.json` manifest that tells GSD what the plugin provides (commands, workflows, agents), what it needs (GSD version, services), and how to activate it.
</overview>

<manifest_schema>
Every plugin requires a `plugin.json` manifest in its root directory.

## Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Plugin identifier (kebab-case) |
| `version` | string | Semantic version (MAJOR.MINOR.PATCH) |
| `description` | string | One-line plugin description |
| `author` | string | Author name or contact |

**Example (minimal plugin):**
```json
{
  "name": "git-hooks",
  "version": "1.0.0",
  "description": "Git workflow automation for GSD",
  "author": "Developer Name"
}
```

## Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `repository` | string | Git URL for plugin source |
| `license` | string | SPDX identifier (MIT, Apache-2.0) |
| `keywords` | array | Discovery tags for search |

## GSD Configuration

The `gsd` object contains plugin-specific configuration:

```json
{
  "gsd": {
    "minVersion": "1.4.0",
    "commands": [],
    "workflows": [],
    "agents": [],
    "hooks": {},
    "services": null
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `minVersion` | string | Minimum GSD version required |
| `commands` | array | Commands the plugin provides |
| `workflows` | array | Workflow files for GSD integration |
| `agents` | array | Agent definitions for subagent spawning |
| `hooks` | object | GSD lifecycle event handlers |
| `services` | object | Docker/service configuration |

</manifest_schema>

<command_registration>
## Command Registration

Commands register as `/pluginname:command-name` in the GSD command namespace.

**Structure:**
```json
{
  "commands": [
    {
      "name": "neo4j:query",
      "file": "commands/query.md",
      "description": "Query the knowledge graph"
    },
    {
      "name": "neo4j:ingest",
      "file": "commands/ingest.md",
      "description": "Ingest research data into graph"
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Command name (format: `pluginname:command`) |
| `file` | Yes | Path relative to plugin root |
| `description` | No | Help text for command listing |

**Command file format:**
Command files follow the standard GSD command format with YAML frontmatter:

```markdown
---
name: neo4j:query
description: Query the knowledge graph
argument-hint: "<cypher-query>"
allowed-tools: [Read, Write, Bash]
---

<process>
[Command implementation]
</process>
```

**Namespace convention:**
- Plugin commands use the format `pluginname:command`
- Avoids collision with core GSD commands (`gsd:*`)
- Multiple commands per plugin share the same prefix
</command_registration>

<workflow_registration>
## Workflow Registration

Workflows integrate with existing GSD workflows or provide new standalone workflows.

**Structure:**
```json
{
  "workflows": [
    {
      "name": "knowledge-capture",
      "file": "workflows/capture.md"
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Workflow identifier (kebab-case) |
| `file` | Yes | Path relative to plugin root |

**Usage:**
Plugin workflows can be invoked by:
1. Plugin commands that delegate to them
2. GSD hooks that trigger them
3. Direct reference in plans (advanced)

**Integration pattern:**
Plugin workflows follow the same structure as GSD core workflows but live in the plugin folder.
</workflow_registration>

<agent_registration>
## Agent Registration

Agents become available for subagent spawning during plan execution.

**Structure:**
```json
{
  "agents": [
    {
      "name": "neo4j-indexer",
      "file": "agents/indexer.md"
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Agent identifier (kebab-case) |
| `file` | Yes | Path relative to plugin root |

**Agent file format:**
```markdown
# [Agent Name]

<role>
[Agent's purpose and capabilities]
</role>

<process>
[Execution flow]
</process>
```

**Spawning:**
Plugin agents can be spawned by orchestrator commands when executing plans that require specialized processing.
</agent_registration>

<hooks_overview>
## Hooks Overview

Hooks allow plugins to respond to GSD lifecycle events.

**Structure:**
```json
{
  "hooks": {
    "post-plan": "hooks/capture-context.md",
    "post-execute": "hooks/index-results.md",
    "pre-commit": "hooks/validate-changes.md"
  }
}
```

**Available lifecycle events:**

| Hook | Trigger | Use Case |
|------|---------|----------|
| `post-plan` | After plan creation | Capture planning context |
| `post-execute` | After plan execution | Index results, notify systems |
| `pre-commit` | Before git commit | Validate changes, run checks |
| `post-commit` | After git commit | Trigger external systems |
| `on-error` | When execution fails | Log errors, send alerts |

**Hook file format:**
Hook files are markdown prompts that execute when triggered:

```markdown
# Post-Execute Hook

<trigger>post-execute</trigger>

<process>
[What to do when triggered]
</process>
```

*Detailed hook implementation in Phase 01-03 (Plugin Hooks and Lifecycle Events)*
</hooks_overview>

<services_overview>
## Services Overview

Plugins can include Docker containers and services that run alongside GSD.

**Structure:**
```json
{
  "services": {
    "docker-compose": "services/docker-compose.yml",
    "healthCheck": "services/health.sh"
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `docker-compose` | Yes | Path to Docker Compose file |
| `healthCheck` | No | Script to verify service health |

**Example docker-compose.yml:**
```yaml
version: '3.8'
services:
  neo4j:
    image: neo4j:5.15
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/password
    volumes:
      - neo4j_data:/data

volumes:
  neo4j_data:
```

**Service lifecycle:**
- Services start when plugin is enabled
- Services stop when plugin is disabled
- Health checks verify service availability

**For plugins without services:**
Set `services` to `null` or omit the field entirely.

*Detailed service implementation in Phase 5 (Self-Contained Dependencies)*
</services_overview>

<validation_rules>
## Validation Rules

A valid plugin manifest must satisfy these rules:

**Name validation:**
- Kebab-case only: `my-plugin` (valid), `myPlugin` (invalid), `my_plugin` (invalid)
- Lowercase letters, numbers, hyphens
- Must start with a letter
- 3-50 characters

**Version validation:**
- Semantic versioning: `MAJOR.MINOR.PATCH`
- Valid: `1.0.0`, `2.3.1`, `0.1.0`
- Invalid: `1.0`, `v1.0.0`, `1.0.0-beta`

**File path validation:**
- Paths are relative to plugin root
- Referenced files must exist
- Forward slashes only (cross-platform)
- No `..` path traversal

**Command name validation:**
- Format: `pluginname:command-name`
- Plugin prefix must match manifest `name`
- Command portion is kebab-case

**Validation errors:**
```
Error: Invalid plugin name "myPlugin" - must be kebab-case
Error: File not found: commands/missing.md
Error: Command prefix "other" doesn't match plugin name "example"
```
</validation_rules>

<examples>
## Complete Examples

### Simple Command Plugin

```json
{
  "name": "git-stats",
  "version": "1.0.0",
  "description": "Git statistics for GSD projects",
  "author": "Developer",
  "license": "MIT",
  "gsd": {
    "minVersion": "1.4.0",
    "commands": [
      {
        "name": "git-stats:summary",
        "file": "commands/summary.md",
        "description": "Show commit statistics"
      },
      {
        "name": "git-stats:contributors",
        "file": "commands/contributors.md",
        "description": "List contributors"
      }
    ]
  }
}
```

### Complex Plugin with Services

```json
{
  "name": "neo4j-knowledge-graph",
  "version": "1.0.0",
  "description": "Knowledge graph for research capture",
  "author": {
    "name": "Developer Name",
    "email": "dev@example.com"
  },
  "repository": "https://github.com/user/gsd-neo4j-plugin",
  "license": "MIT",
  "keywords": ["knowledge-graph", "neo4j", "research"],
  "gsd": {
    "minVersion": "1.4.0",
    "commands": [
      {
        "name": "neo4j:query",
        "file": "commands/query.md",
        "description": "Query knowledge graph"
      },
      {
        "name": "neo4j:ingest",
        "file": "commands/ingest.md",
        "description": "Ingest research data"
      }
    ],
    "workflows": [
      {
        "name": "capture-research",
        "file": "workflows/capture.md"
      }
    ],
    "agents": [
      {
        "name": "neo4j-indexer",
        "file": "agents/indexer.md"
      }
    ],
    "hooks": {
      "post-plan": "hooks/capture-context.md",
      "post-execute": "hooks/index-results.md"
    },
    "services": {
      "docker-compose": "services/docker-compose.yml",
      "healthCheck": "services/health.sh"
    }
  }
}
```
</examples>

<anti_patterns>
## Anti-patterns

**Avoid these common mistakes:**

<pattern name="wrong-naming">
### Wrong: Non-kebab-case names

```json
{
  "name": "myPlugin",
  "commands": [
    {"name": "myPlugin:doThing"}
  ]
}
```

Correct: Use kebab-case everywhere
```json
{
  "name": "my-plugin",
  "commands": [
    {"name": "my-plugin:do-thing"}
  ]
}
```
</pattern>

<pattern name="missing-prefix">
### Wrong: Command name without plugin prefix

```json
{
  "name": "analytics",
  "commands": [
    {"name": "report", "file": "commands/report.md"}
  ]
}
```

Correct: Prefix commands with plugin name
```json
{
  "name": "analytics",
  "commands": [
    {"name": "analytics:report", "file": "commands/report.md"}
  ]
}
```
</pattern>

<pattern name="absolute-paths">
### Wrong: Absolute or traversal paths

```json
{
  "commands": [
    {"name": "x:y", "file": "/usr/local/commands/x.md"},
    {"name": "x:z", "file": "../other-plugin/cmd.md"}
  ]
}
```

Correct: Relative paths within plugin folder
```json
{
  "commands": [
    {"name": "x:y", "file": "commands/x.md"},
    {"name": "x:z", "file": "commands/z.md"}
  ]
}
```
</pattern>

<pattern name="colliding-namespace">
### Wrong: Using gsd: prefix

```json
{
  "name": "my-extension",
  "commands": [
    {"name": "gsd:my-command", "file": "commands/cmd.md"}
  ]
}
```

Correct: Use plugin name as prefix
```json
{
  "name": "my-extension",
  "commands": [
    {"name": "my-extension:my-command", "file": "commands/cmd.md"}
  ]
}
```
</pattern>

<pattern name="service-without-health">
### Risky: Services without health checks

```json
{
  "services": {
    "docker-compose": "services/docker-compose.yml"
  }
}
```

Better: Include health check for reliability
```json
{
  "services": {
    "docker-compose": "services/docker-compose.yml",
    "healthCheck": "services/health.sh"
  }
}
```
</pattern>
</anti_patterns>

<summary>
## Summary

**Plugin manifest essentials:**
1. Required: `name`, `version`, `description`, `author`
2. Optional: `repository`, `license`, `keywords`
3. GSD config: `minVersion`, `commands`, `workflows`, `agents`, `hooks`, `services`

**Key conventions:**
- Names are kebab-case
- Commands use `pluginname:command` format
- File paths are relative to plugin root
- Services are optional (set to `null` if not needed)

**Validation ensures:**
- Name format is correct
- Version is semver
- Referenced files exist
- Command prefixes match plugin name

*See template at `get-shit-done/templates/plugin/plugin.json`*
</summary>
