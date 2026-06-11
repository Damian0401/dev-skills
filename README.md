# Dev Skills

A collection of Claude Code skills useful for dotnet and typescript development

## Project Setup

### `setup-dotnet-api-client`
Creates a complete Refit-based API client generation workflow for .NET solutions.

- Adds a `*.ApiClient` classlib project to your solution
- Installs and configures **refitter** as a local tool
- Writes a `generate.sh` script that starts the API, polls the OpenAPI endpoint, regenerates the typed client, and packs it as a NuGet package
- Configures a local NuGet feed and updates `CLAUDE.md`

**Trigger phrases:** "add dotnet api client", "add ApiClient project", "generate refit client", "setup refitter", "create generate.sh"

---

### `setup-integration-tests`
Sets up a complete xUnit v3 integration test project for a .NET minimal API with PostgreSQL.

- Creates `*.IntegrationTests` with **WebApplicationFactory** for in-process testing
- Spins up a PostgreSQL database via **Testcontainers**
- Uses **Respawn** for fast per-test database resets (no container restarts)
- Generates test classes (GET all, GET by id, POST, DELETE) for every entity in the project
- Verifies the build before finishing

**Trigger phrases:** "setup integration tests", "add integration tests", "add testcontainers", "setup xunit integration tests", "add respawn"

---

### `setup-ts-api-client`
Configures a type-safe API client workflow in a Vite + React + TypeScript frontend.

- Installs **openapi-fetch** and **openapi-typescript**
- Creates `src/libs/api.ts` with an auth middleware (Bearer token + 401 redirect)
- Creates `src/libs/env.ts` with Zod-validated environment variables
- Adds `generate` / `generate:api` scripts to `package.json`
- Runs the generator against the live backend and updates `CLAUDE.md`

**Trigger phrases:** "set up TypeScript API client", "add openapi-fetch", "generate TypeScript API types", "setup type-safe fetch"