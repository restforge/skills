# RESTForge — Claude Code Plugin

[RESTForge](https://restforge.dev) is a definition-first REST API generation framework.
You write a JSON definition file — RESTForge generates a complete, production-ready
backend (database schema + REST endpoints) and frontend (HTML/JS/CSS) from it.
No boilerplate, no hand-coding repetitive CRUD.

This plugin connects RESTForge to Claude Code so you can build entire applications
through conversation.

**What you get after installing this plugin:**

- **Skill** — Claude understands the full RESTForge workflow end-to-end: project setup,
  schema definition, database migration, endpoint generation, frontend generation,
  and runtime management. Claude knows the correct order of operations and will
  guide you through each step.
- **MCP server** — 61 RESTForge tools are registered into Claude Code automatically,
  giving Claude direct access to execute RESTForge operations on your behalf.

---

## Prerequisites

Before installing the plugin, make sure the following are in place:

**1. Node.js >= 18 and npm >= 9**

**2. RESTForge MCP server** — install globally so Claude Code can start it:

```bash
npm install -g @restforgejs/mcp-server
```

> **No global install?** You can use `npx` instead. After installing this plugin,
> open `.claude/plugins/restforge/.mcp.json` and replace its content with:
> ```json
> { "mcpServers": { "restforge": { "command": "npx", "args": ["-y", "@restforgejs/mcp-server"] } } }
> ```

**3. A valid RESTForge license key**

Required for schema migration, endpoint generation, frontend generation, and runtime
operations. Get a license at [restforge.dev](https://restforge.dev).

---

## Installation

Open Claude Code and run these two commands in the chat — **not in a terminal**:

**Step 1 — Register the RESTForge plugin source:**
```
/plugin marketplace add restforge/skills
```

**Step 2 — Install the plugin:**
```
/plugin install restforge@restforge-skills
```

When prompted, choose the installation scope:
- **Install for you (user scope)** — available on your machine across all projects *(recommended for personal use)*
- **Install for all collaborators on this repository (project scope)** — shared via the repo's `.claude/settings.json`
- **Install for you, in this repo only (local scope)** — active only in the current project

**Step 3 — Apply the plugin:**
```
/reload-plugins
```

> Both Step 1 and Step 2 are required. Step 1 tells Claude Code where to find the
> RESTForge plugin; Step 2 installs it. Skipping Step 1 will result in
> "Marketplace not found" when running Step 2.

---

## Team Installation

To automatically provide the plugin to everyone who opens a shared repository,
commit the following to `.claude/settings.json` in the project root:

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

Team members get the plugin automatically when they open the repo — no manual
install steps needed.

---

## Usage

Once installed, just describe what you want to build in plain language.
Claude will load the RESTForge skill automatically and guide you through the steps:

```
I want to set up a new RESTForge project with PostgreSQL
```
```
Create a CRUD endpoint for a products table with name, price, and stock fields
```
```
Generate a frontend app for the orders and customers endpoints
```
```
Migrate the database schema after I updated the SDF file
```

To start the RESTForge workflow explicitly:

```
/restforge:restforge-skills
```

---

## What Can RESTForge Generate?

| Output | Description |
|---|---|
| Database schema (DDL) | Tables, columns, indexes, foreign keys — from a JSON schema definition |
| REST API endpoints | Full CRUD, workflow, master-detail, aggregate, import/export — from a payload definition |
| Frontend application | HTML/JS/CSS CRUD pages and dashboards — from a UI definition |

Supports PostgreSQL, MySQL, Oracle, and SQLite.

---

## License

MIT.
