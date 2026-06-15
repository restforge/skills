# RESTForge — Claude Code Plugin

Claude Code plugin for RESTForge, a definition-first deterministic REST API generation framework (SDF). One install delivers two layers at once:

1. **Skill** — end-to-end RESTForge workflow guidance (operation sequence, grounding-first rules, safety guardrails).
2. **MCP server** — automatic registration of `@restforgejs/mcp-server` (61 tools) into Claude Code.

---

## Prerequisites

- **Node.js >= 18** and **npm >= 9**.
- **MCP server** installed globally:

  ```bash
  npm install -g @restforgejs/mcp-server
  ```

- A valid **RESTForge license key** for `codegen_*` / `runtime_*` operations.

> **Alternative without global install:** replace the contents of `.mcp.json` with:
> ```json
> { "mcpServers": { "restforge": { "command": "npx", "args": ["-y", "@restforgejs/mcp-server"] } } }
> ```

---

## Installation

Inside Claude Code, run the following two commands:

```
/plugin marketplace add restforge/skills
/plugin install restforge@restforge-skills
```

Restart the Claude Code session after installation.

---

## Team installation

To automatically distribute the plugin to all team members when they open a repo, add the following to `.claude/settings.json` inside the project repository:

```json
{
  "marketplaces": [
    { "source": "restforge/skills" }
  ],
  "plugins": [
    { "name": "restforge", "marketplace": "restforge-skills" }
  ]
}
```

---

## Usage

Simply describe the task in natural language — the skill loads automatically when relevant:

```
Set up a new RESTForge project at ./api with PostgreSQL
Create a CRUD endpoint for the orders table
Validate the SDF before migrating to the database
```

Explicit invocation:

```
/restforge:restforge-skills
```

---

## License

MIT.
