# Reference: Auth Extension

> **Offline mirror.** This file mirrors the auth commands of the installed
> RESTForge platform and Designer (`restforge project auth`, `restforge-designer
> auth`). The live tools/CLI are authoritative — when this file and them disagree,
> trust the tools, then update this file.

**This file documents the auth EXTENSION — auth WITHOUT RBAC.** It is one of two
distinct auth mechanisms; do not confuse them:

| Mechanism | RBAC? | Where |
|---|---|---|
| **Plugin auth** — built into the frontend at generation | **Yes (auth + RBAC)** | `vanilla-js-auth` / `vanilla-js-custom` plugin, chosen at `designer_init_project`; toggle off with `noAuth: true` (`--no-auth`). See `udf-catalog.md § Plugins`. |
| **Auth extension** (this file) | **No RBAC** | `project_auth` (backend) + `designer_auth_create` (frontend `rfx_auth`) |

The extension is an optional add-on installed into a project that already exists.
Backend auth and frontend auth are **independent**: install either, both, or
neither. Both are JWT-based.

**Out of scope for the extension:** Google Sign-In, RBAC, and the
`@restforgejs/auth` package. The embedded frontend flow is independent of the
`vanilla-js-auth` plugin — if the user needs RBAC, use plugin auth, not this.

---

## Backend

MCP tool: `project_auth` — wraps `npx restforge project auth --create`.

```
npx restforge project auth --create --project=<name> [options]
```

| Flag | Required | Default | Notes |
|---|---|---|---|
| `--create` | yes | `false` | Required trigger; must be present |
| `--project <name>` | yes* | — | Target project name |
| `--name <name>` | yes* | — | Alias of `--project` |
| `--schema-path <dir>` | no | `./schema` | Output folder for auth SDF files |
| `--config <file>` | no | `config/db-connection.env` | DB config for the migrate step |
| `--force` | no | `false` | Overwrite existing files (backup still made) |

\* one of `--project` / `--name` is required.

MCP params: `cwd` (project folder, must contain `node_modules/@restforgejs/platform`),
`project`, `schemaPath?`, `config?`, `force?`.

**What it does (in order):**
1. Generates auth SDF files (prefix `rfx`) to `--schema-path`.
2. Creates auth DB tables via dbschema-kit (idempotent, `IF NOT EXISTS`).
3. Writes auth middleware + router.
4. Writes six processors: `register`, `login`, `refresh`, `logout`, `me`,
   `reset-password`.
5. Injects auth env vars (random `JWT_SECRET`) into the config file.
6. Records `bcrypt` + `jsonwebtoken` as runtime dependencies in the project's
   `package.json`.

**Prerequisites:** the project already exists (run `endpoint create` /
the standard backend pipeline first), `@restforgejs/platform` is installed in the
project, and the DB is active and reachable with the config credentials.

Idempotent (re-runnable). Non-destructive aside from `--force` overwrites, which
still create a backup.

---

## Frontend

Embedded login / signup / forget-password overlay (`rfx_auth`), mounted at route
`/api/<project>/rfx_auth`. Independent of the `vanilla-js-auth` plugin.

MCP tools: `designer_auth_create`, `designer_auth_remove` — wrap
`restforge-designer auth --create | --remove`.

```
restforge-designer auth --create --project=<name> [options]
restforge-designer auth --remove --project=<name> [--frontend-path <path>] --force
```

`--create` and `--remove` are mutually exclusive; exactly one is required.

| Flag | Applies to | Default | Notes |
|---|---|---|---|
| `--create` | create | — | Install the overlay |
| `--remove` | remove | — | Uninstall the overlay |
| `--project <name>` | both | — | App code, localStorage prefix, route `/api/<project>/rfx_auth` |
| `--frontend-path <path>` | both | `./frontend/apps` | Apps root; target app = `<frontend-path>/<project>` |
| `--api-base-url <url>` | create | from `app-config.json` | Override backend base URL |
| `--plugins-dir <dir>` | both | auto-detect | Plugins directory |
| `--overwrite` | create | `false` | Overwrite existing auth files (+ archive backup) |
| `--force` | remove | `false` | Skip the y/N removal prompt |

MCP params — `designer_auth_create`: `cwd`, `project`, `frontendPath?`,
`apiBaseUrl?`, `overwrite?`. `designer_auth_remove`: `cwd`, `project`,
`frontendPath?` (the MCP tool always passes `--force`).

**`--create` does (in order):**
1. Renders `login.html`, `signup.html`, the forget-password overlay, and
   `js/rfx_auth.js` from embedded templates (no Google Sign-In).
2. Writes the artifacts to `<frontend-path>/<project>/`.
3. Injects an auth guard `<script src="js/rfx_auth.js">` into all existing
   `*.html` pages in the target dir (except `login.html` / `signup.html`).
4. Writes the `embeddedAuth` marker to `payload/app-config.json` (non-destructive;
   other keys untouched).

If no app pages exist yet, guard injection is skipped with a warning; the guard is
injected automatically when `designer_generate` creates pages later.

**`--remove` does (in order):**
1. Detects whether auth is installed (files + `embeddedAuth` marker).
2. Deletes `login.html`, `signup.html`, the forget-password overlay,
   `js/rfx_auth.js`.
3. Strips the auth guard from all `*.html` pages (other content untouched).
4. Removes the `embeddedAuth` key from `payload/app-config.json` (other keys kept).

**Prerequisites:** the `restforge-designer` binary is installed and on PATH.

Both are idempotent. **`--remove` is destructive** — confirm project name and
intent with the user before running (MCP always passes `--force`).

---

## Two frontend auth approaches — do not combine

| Approach | RBAC? | When | How |
|---|---|---|---|
| Plugin auth (`vanilla-js-auth`, `vanilla-js-custom`) | **Yes (auth + RBAC)** | A new app that needs auth/RBAC | Choose the plugin in `designer_init_project`; auth is built in at generation. Disable with `noAuth: true` (`--no-auth`) |
| Embedded `rfx_auth` | **No RBAC** | An existing app generated WITHOUT plugin auth | `designer_auth_create` bolts auth on |

Do not add `rfx_auth` to an app already built with plugin auth — they are two
separate mechanisms. If the user needs RBAC, only plugin auth provides it.

---

## Common errors

| Symptom | Cause | Recovery |
|---|---|---|
| Backend: "package not installed" precondition | `@restforgejs/platform` missing in the project | Install the package, then re-run |
| Backend: project does not exist / DB not reachable | Auth runs against an existing project + live DB | Create the project + endpoints first; verify DB config |
| Frontend: "Designer not on PATH" precondition | `restforge-designer` binary missing | Install RESTForge Designer and ensure it is on PATH |
| Frontend: files already exist, not overwritten | Auth already installed | Use `--overwrite` (create) only if you intend to replace |
