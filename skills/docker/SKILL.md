---
name: docker
description: Run the app, its database and every test tier in Docker Compose, so any agent on any machine can start the project with one command. Use when setting up a project, adding a dependency service, or running checks.
---

# Docker

Nothing needs to be installed on the host except Docker. The SDK, the database and the tools run in containers.

## Files

```
Dockerfile          # multi-stage: sdk (build + test) -> AOT publish -> runtime-deps
.dockerignore       # bin/, obj/, .git/, TestResults/, StrykerOutput/
compose.yaml        # db, api, and one service per check
db/init.sql         # schema, mounted into postgres at /docker-entrypoint-initdb.d
```

## Dockerfile

- Build stage: `mcr.microsoft.com/dotnet/sdk:<version>-noble-aot` (has clang and the native toolchain for Native AOT).
- Copy project files first and `dotnet restore`, then copy the rest (layer caching).
- Publish with `-r linux-x64` and `PublishAot=true`.
- Final stage: `mcr.microsoft.com/dotnet/runtime-deps:<version>-noble-chiseled`. Run as non-root. No SDK and no shell in the final image.

## compose.yaml

- `db`: PostgreSQL with a healthcheck (`pg_isready`). Mount `db/init.sql`.
- `api`: built from the Dockerfile, `depends_on: db: condition: service_healthy`, connection string from an environment variable.
- Check services share one SDK image and mount the source, behind `profiles` so `docker compose up` starts only the app:
  - `test`: `dotnet test` for all test projects (needs `db`).
  - `mutation`: `dotnet tool restore && dotnet stryker`.
- Keep a NuGet cache volume so repeated runs are fast.

## Commands (put them in the project README)

| Goal | Command |
| --- | --- |
| Run the app | `docker compose up --build api` |
| Agent loop / PR tests | `docker compose run --rm test` |
| Mutation testing | `docker compose run --rm mutation` |
| Clean up | `docker compose down -v` |

## Rules

- A check that does not run in Compose does not exist. Add a service for it.
- Connection strings and secrets come from environment variables, never from committed files. Local development defaults may live in `compose.yaml`.
- Pin image tags to a major version (`postgres:17`, `sdk:10.0-noble-aot`).
