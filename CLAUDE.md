# ShapeRouter API MCP

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`/`git push`/`gh pr create`/`scripts/push_all.sh` **only when the user
> asks in that turn** — earlier approval does not carry forward, and finishing a task is
> not permission to commit it. Stay on the current branch; no feature branches for code.

MCP (Model Context Protocol) server that describes and drives the ShapeRouter API —
an LLM structured-output platform — for AI assistants like Claude Code and Claude
Desktop.

**Package**: `@sudobility/shaperouter_api_mcp` (BUSL-1.1)

## Tech Stack

- **Runtime**: Bun
- **Package Manager**: Bun (do not use npm/yarn/pnpm)
- **MCP SDK**: `@modelcontextprotocol/sdk` ~1.30
- **Validation**: Zod 4 (imported as `zod/v4`)
- **Transport**: stdio

## Architecture

```
AI assistant (Claude Code / Claude Desktop)
    ↕ stdio (MCP protocol)
ShapeRouter API MCP server (this project)
    ↕ HTTP / REST
ShapeRouter API (Hono on Bun, port 3000 locally)
```

Thin HTTP client: one tool per REST route, plus embedded documentation exposed as
MCP resources. No database, no shared code with the API — accuracy comes from
keeping the tool descriptions and docs in sync with `../shaperouter_api/src/routes/`.

## Commands

```bash
bun run dev        # Run the MCP server (stdio)
bun run build      # Bundle to dist/index.js
bun run typecheck  # TypeScript check
bun run verify     # typecheck + build
bun run start      # Run the production bundle
```

## Project Structure

```
src/
├── index.ts        # Entry: credential resolution, registration, stdio transport
├── client.ts       # HTTP client: auth-mode routing, envelope unwrapping, ApiError
├── config-file.ts  # ~/.shaperouter/config.json read/write (mode 0600)
├── prompts.ts      # Prompt templates (setup / debug / audit)
├── resources/      # index.ts (registration + DOC_SECTIONS) + content.ts (embedded docs)
└── tools/          # one module per route family; util.ts has ok/fail/run/compact.
                    #   entity-apikeys, apikeys(7), entities(15), keys(5), projects(7),
                    #   endpoints(5), providers(3), ai(2), config(3), health(2), docs,
                    #   analytics, ratelimits(2), storage(4), users(4)

skills/
├── shaperouter-endpoint/   # /shaperouter-endpoint: build/invoke/debug/audit one endpoint
│   ├── SKILL.md            #   + references/{creating-endpoints,schema-design,
│   └── references/         #     model-selection,authentication}.md
└── shaperouter-providers/  # /shaperouter-providers: provider keys, projects,
    └── SKILL.md            #   endpoints, entity API keys (entity-key auth)

.claude-plugin/   # plugin.json + marketplace.json (Claude Code plugin)
.mcp.json         # MCP server declaration (resolves path via ${CLAUDE_PLUGIN_ROOT},
                  #   reads SHAPEROUTER_* from the environment — no secrets in repo)
```

Ships as a Claude Code plugin (`claude plugin marketplace add <repo>` then
`claude plugin install shaperouter@shaperouter`); Claude Code copies the directory
incl. `node_modules` into `~/.claude/plugins/cache/shaperouter/`, so after edits run
`claude plugin marketplace update shaperouter` + `claude plugin update shaperouter@shaperouter`.
**Keep `.claude-plugin/plugin.json` `version` in step with `package.json`** —
`claude plugin update` is a no-op when they match, so an unbumped plugin manifest
strands everyone on a stale copy. `push_all.sh` bumps `package.json` only.
(As of this writing the two are out of sync: plugin `0.0.1` vs package `0.0.4`.)

## Environment Variables

Resolution order: explicit tool argument → environment variable → config file. Full
table in README; the credentials are `SHAPEROUTER_{ENTITY_API_KEY, API_KEY,
AUTH_TOKEN, PROJECT_API_KEY}` plus `API_URL`, `ENTITY_SLUG`, `ORG_PATH`,
`CONFIG_PATH`. The server runs with none; tools that need a credential fail with a
how-to-get-one message. `set_credentials` overrides for the session, and
`persist: true` writes `~/.shaperouter/config.json` (mode 0600). Firebase tokens
are never persisted — they expire within the hour.

## Auth Model (mirrors the API)

- `/api/v1/ai/*` → project API key as `Authorization: Bearer sk_live_...`
- `/api/v1/providers/*`, `/health*`, `/` → no auth
- everything else → personal API key as `X-API-Key: shroute_...`, or a Firebase ID
  token as `Authorization: Bearer <token>`

`client.ts` picks the scheme via the `auth` request option:

| Mode | Behavior |
|---|---|
| `"admin"` (default) | Personal API key if present, else the Firebase token |
| `"firebase_only"` | Firebase token required — create/reveal of API keys, which the API refuses from key-authenticated callers |
| `"project"` | Project API key, for AI invocation |
| `"none"` | Public route |

Never send a personal key to an AI route or a project key to an admin route — the
API routes credentials by prefix, and a mismatch reads as "invalid key".

## Surface

65 tools, 6 resources (`shaperouter://api/{overview,routes,data-model,examples,errors,providers}`),
3 prompts (`setup_structured_endpoint`, `debug_endpoint`, `audit_entity`), and two
skills (`/shaperouter-endpoint`, `/shaperouter-providers`). 11 providers
(`providers.ts` enum), including `jev` (TypeSafe AI).

## Code Patterns

A tool is `server.tool(name, description, zodShape, handler)` in a `src/tools/*.ts`
module, registered from `src/index.ts`. Conventions:

- Descriptions name the HTTP method, path, response shape, and failure modes — they
  are the only API documentation the model sees at call time.
- `run()` converts thrown errors into `isError` results (handlers never throw);
  `compact()` strips `undefined` so PUT bodies carry only provided fields;
  `client.seg()` encodes interpolated path segments; `resolveEntitySlug()` /
  `resolveOrgPath()` apply env defaults and raise a helpful error when unset.
- `src/resources/content.ts` holds the API-doc markdown as template literals
  (backticks/`${` escaped) so `bun build` is self-contained; `describe_shaperouter_api`
  and the MCP resources read the same map — edit the strings directly.

## Task Recipes

### Adding a tool for a new API route

1. Confirm the real route in `../shaperouter_api/src/routes/` (path, auth, Zod
   schema, response type) — do not trust older docs.
2. Add the tool to the matching `src/tools/*.ts` (new file → register in `index.ts`);
   write the description with method, path, response shape, and error cases.
3. Update `src/resources/content.ts` (`ROUTES_MD`, `DATA_MODEL_MD` if new types) and
   the tool tables in `README.md`.
4. `bun run verify`, then smoke test over stdio.

### Editing the skills

Two skills under `skills/`: `shaperouter-endpoint/` (one endpoint) and
`shaperouter-providers/` (workspace keys/projects/endpoints). Deep guidance goes in
`references/`. Keep tool names in `SKILL.md` matching what the server registers (a
call to a nonexistent tool fails silently); do not duplicate the API docs — point
at `describe_shaperouter_api({ section })`. Validate: `claude plugin validate .`
and `claude plugin validate skills`.

### Smoke testing

Pipe MCP JSON-RPC frames over stdio: an `initialize` request, a
`notifications/initialized`, then `tools/list`, into `bun run src/index.ts`. The
public tools (`check_api_health`, `list_providers`) run against production with no
credentials.

## Workspace & downstream

Part of the **ShapeRouter** workspace (`~/projects/shaperouter.code-workspace`)
alongside `shaperouter_api`, `shaperouter_app`, `shaperouter_client`,
`shaperouter_lib`, `shaperouter_types`. A leaf project with **no** `@sudobility/*`
deps — upstream upgrades never break it, but backend route changes do: the tools
are hand-mirrored against `../shaperouter_api/src/routes/`, not generated.

## Gotchas

- **stdout is the MCP transport.** Never `console.log`; diagnostics go to stderr.
- **Import Zod as `zod/v4`.** The SDK's `ZodRawShapeCompat` expects the v4 surface.
- **Local imports use the `.ts` extension** (`./client.ts`), matching
  `allowImportingTsExtensions` in `tsconfig.json`.
- **On GET AI endpoints every query parameter becomes input** (incl. `api_key`,
  `testMode`) — the client always authenticates via the header for that reason.
- **`:memberId` is the member's Firebase UID**, not the membership row id.
- **`organizationPath` in AI URLs is the entity slug** (slug lookup), not the
  `organization_path` from user settings, despite the name.
- **Project API key ≠ LLM provider key.** `sk_live_...` authenticates callers of
  your endpoints; provider keys are ShapeRouter's creds for OpenAI et al. and are
  never returned by the API.
- **A `404` from an invocation can mean "inactive"**, not just "missing" — check
  `is_active` on both project and endpoint.
- **Rate limits are per entity**, shared by every project and endpoint under it.
- **Four credentials**: `shrouteent_` entity and `shroute_` personal go as
  `X-API-Key`; `sk_live_` project and the Firebase token go as `Authorization:
  Bearer`. Distinct prefixes let the API route each to the right verifier.
- **Entity keys cannot reach `/users/*` or manage API keys** (403 by design); the
  `user` auth mode refuses to send one rather than provoke a confusing 403.
- **`create_api_key`/`reveal_api_key` need a Firebase token** (403 for
  key-authenticated callers by design); the shaperouter.com dashboard is the usual path.
- **Secrets never go to stdout as prose.** `get_configuration`/`set_credentials`
  redact; only `create_api_key`, `reveal_api_key`, `get_project_api_key` return a secret.
- **Config file is user-scoped** (`~/.shaperouter/config.json`), never in a repo, so
  a key cannot be committed by accident.
- **Three places describe this server**: tool descriptions (what the model reads at
  call time), `src/resources/content.ts` (the API documentation), and `skills/`
  (the workflows). Adding a route means touching all three plus `README.md`.
