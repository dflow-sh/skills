---
name: dflow-deploy
description: Deploy a git app, Docker image, database, local archive, or template onto a dFlow worker node. Use when the user wants to create an application, attach compute, add a service, or ship a deployment through the dashboard or dFlow MCP.
---

# Deploy on dFlow

Use this skill to ship a workload. For connecting MCP, domains, scale, backups, or reading logs of something already running, use `dflow-mcp`.

## Product facts

- Dashboard: https://app.dflow.sh
- Docs: https://docs.dflow.sh/
- MCP: `https://app.dflow.sh/api/mcp` (OAuth only, no API keys)
- MCP docs: https://docs.dflow.sh/articles/3167609-dflow-mcp
- Status: https://status.dflow.sh

dFlow builds from git, a Docker image, or an archive, wires the database, issues the certificate, and keeps the workload running. Compute is attached on the environment. Every service in that environment runs on that worker node.

## Resource model

Application → Environment (one compute target) → Service (app, database, or Docker) → Deployment

- Application names and service names are different. Resolve a service with `get_service_by_id` by id or case-insensitive name.
- Service names must be lowercase letters, numbers, and hyphens (`my-api`). No uppercase, underscores, or spaces.
- Application and environment status are system-owned. They start as draft and become active when the first service is created. Do not set status.
- Prefer MCP over inventing HTTP calls. Server card: `https://dflow.sh/.well-known/mcp/server-card.json`.

## Ship a stack

1. `create_application`, or `list_applications` / `get_application_by_id` when one already exists.
2. `list_worker_nodes`, then `create_environment` with the application id and a worker node id as `computeId`.
3. On a self-managed node, `check_worker_node_resources` with the node id and the service kind (`app`, `docker`, or `database`) before creating a service or deploying. If `capable` is false, stop and report `reason` and `status`.
4. `create_app_service`, `create_database_service`, or `create_docker_service` with the environment id.
5. `update_service` with the source, builder, volumes, and variables.
6. `create_deployment` with the service id. This starts a job. Poll `get_deployments_by_service_id` until it finishes. `no-cache` pulls the latest image. `cache` rebuilds the existing image.
7. Build and deploy logs are on the deployment. After the service is up, runtime output is `get_service_runtime_logs`.
8. `set_default_environment` when the user wants that environment to be the application default. `update_application` and `update_environment` change name or description only, not status.

Volumes should use host path `/var/lib/dokku/data/storage/{service-name}/{folder-name}`. For a non-root image that writes to a volume, set `DFLOW_USER_ID` and `DFLOW_GROUP_ID` (often `1000`) on the service. Skip that when the process runs as root.

## Source

- Git: `update_service` with the git provider, repository, and branch. List providers, repos, and branches with the GitHub tools in `dflow-mcp` before guessing ids.
- Docker: `create_docker_service`, then `update_service` with the image. Registry credentials go through the Docker registry tools, and a connection test runs before anything is saved.
- Database: `create_database_service`. ClickHouse backups are unsupported.
- Local archive: app services only, and only when there is no git remote. Do not send archive bytes through MCP.
- Template: `deploy_template_to_application` creates the services and deploys them one by one in list order. Put **database** services first in the template `services` array (before apps/docker) so apps can reference `{{ db.POSTGRES_URI }}` after the DB exists. Check capacity first. Do not call `create_deployment` afterward.

## Configure the service

Use `update_service` for source, builder, commands, ports, volumes, and variables. Variable and volume changes need a redeploy or restart to take effect.

### Variables

Values can be:

- **Literal** — static string (e.g. `NODE_ENV=production`)
- **Reference** — `{{ service-name.VARIABLE_NAME }}` from another service in the same environment
- **Combo** — mix literals with multiple `{{ … }}` refs (and optional `secret()`) in one value
- **Secret helper** — `{{ secret(64, "aA1!") }}` (length + charset); resolved once and kept while the template string is unchanged (including inside a combo)

Examples:

- `DATABASE_URI={{ my-db.POSTGRES_URI }}`
- `DATABASE_URL=postgres://{{ my-db.POSTGRES_USERNAME }}:{{ my-db.POSTGRES_PASSWORD }}@{{ my-db.POSTGRES_HOST }}:{{ my-db.POSTGRES_PORT }}/{{ my-db.POSTGRES_NAME }}`
- `APP_URL={{ my-api.DFLOW_PUBLIC_DOMAIN }}`
- `PAYLOAD_SECRET={{ secret(64, "abcdefghijklMNOPQRSTUVWXYZ") }}`

Database keys (prefix by type): `POSTGRES_*`, `MONGO_*`, `MYSQL_*`, `MARIADB_*`, `REDIS_*` — e.g. `URI`, `USERNAME`, `PASSWORD`, `HOST`, `PORT`, `NAME`, plus `PUBLIC_*` after expose. Prefer the full `*_URI` when the app accepts it; use combo when it needs a custom string. Reads never return literal secret values; `{{ reference }}` templates may still appear.

### System variables (volume ownership)

For non-root images that write to bind mounts (often UID `1000`, e.g. Directus or PocketBase):

| Variable | Purpose |
| --- | --- |
| `DFLOW_USER_ID` | Numeric UID for `chown` on volume host paths |
| `DFLOW_GROUP_ID` | Numeric GID for `chown` on volume host paths |

Both optional. Integers `0`–`2147483647`. If only one is set, that value is used for both uid and gid. Skip when the process runs as root. Prefer host path `/var/lib/dokku/data/storage/{service-name}/{folder-name}`.

### Commands

On `update_service`: `startCommand`, `buildCommand`, `installCommand` (also `dockerfilePath`, `procfilePath`, `staticRoot`, `staticIndexFile` when the builder supports them). Support is builder-gated (e.g. Railpack allows install/build/start; Docker image services support a start override). If the API rejects a field, switch builder or put the command in the repo config instead.

### Ports

- **App (git):** set the listen port on the git settings (e.g. `githubSettings.port`).
- **App (file archive):** `fileSourceSettings.port` (default often `3000`).
- **Docker:** `dockerDetails.ports` as `{ hostPort, containerPort, scheme: http|https }[]`.

Public traffic reaches the service hostname (`*.up.dflow.sh` or a custom domain), not the worker node's public IP on port 80.

### Proxy / large uploads

App and Docker services have a dashboard **Proxy** tab for nginx. Default `client-max-body-size` is `1m` — raise it for large uploads. Also `client-body-timeout` / `client-header-timeout` (default `60s`). MCP has no nginx get/set tools yet; tell the user to use the dashboard Proxy tab.

### Database linking and expose

"Attach a database" means create **and** wire:

1. `create_database_service` in the same environment (after capacity check).
2. `update_service` on the app with a variable like `DATABASE_URI={{ my-db.POSTGRES_URI }}` (match the DB type key above).
3. `create_deployment` on the app (and the database if it still needs a first deploy).

Do not leave a standalone DB with no app variable. Public connection vars (`POSTGRES_PUBLIC_URI`, etc.) stay off until the user exposes the database in the dashboard; MCP cannot expose/unexpose ports yet.

## Local archive

1. Pack a `.zip`, `.tar`, or `.tar.gz` of the app, 1 KiB to 30 MiB. Exclude `node_modules` and `.git`.
2. Generate a UUID `artifactId`.
3. `request_file_source_upload` with filename, size, content type, archive type, and that `artifactId`. Use `application/octet-stream` when the MIME type is unknown.
4. HTTP PUT the file to `uploadUrl` with the returned `Content-Type`. The size must match. Finish before the URL expires (1 hour).
5. `update_service` with `sourceType: file` and `fileSourceSettings` using the same key, filename, size, content type, archive type, `artifactId`, `uploadedAt`, build path, and port.
6. `create_deployment` and poll.

If the file cannot be read or uploaded, tell the user to use the dashboard or git. Do not paste the archive into a tool argument.

## After deploy

- `get_deployments_by_service_id` for status, history, and build logs.
- `get_deployment_logs` for one deployment.
- `get_service_runtime_logs` for runtime output.
- `redeploy_from_deployment` from a completed deployment.
- `cancel_deployment`, `approve_deployment`, and `discard_deployment` for in-progress or gated deploys.
- `update_deployment_settings` for auto-deploy and approval.

Deployments and deletes are asynchronous and can take several minutes. After a delete, wait and list again. Do not retry immediately.

## Guardrails

- Confirm the service name and id before `delete_service`, `delete_environment`, or `delete_application`.
- Leave backup-deletion flags false unless the user explicitly asks to destroy dumps.
- Do not state names, ids, log lines, or deploy status that a tool result in this turn did not return.
