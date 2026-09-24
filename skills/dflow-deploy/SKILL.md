---
name: dflow-deploy
description: Deploy git apps, Docker images, and databases on dFlow worker nodes from the dashboard or MCP.
---

# Deploy on dFlow

Use this skill when a user wants to ship an application, database, or Docker
image to dFlow compute, their own cloud, or a connected VPS.

## Product facts

- Dashboard: https://app.dflow.sh
- Docs: https://docs.dflow.sh/
- MCP docs: https://docs.dflow.sh/articles/3167609-dflow-mcp
- Status: https://status.dflow.sh

dFlow builds from git, wires the database, issues the certificate, and keeps
the workload running. Compute is attached at the environment level. Services
in that environment run on that worker node.

## Resource model

Application → Environment (compute) → Service (app, database, or Docker)

- Application and environment names are not interchangeable with service names.
- Service names must be lowercase letters, numbers, and hyphens.
- Deployments are asynchronous. Poll deployment status after create.

## Human flow

1. Sign in at https://app.dflow.sh/sign-in
2. Create or open an application
3. Attach compute by creating an environment on a worker node
4. Add a git, Docker, or database service
5. Deploy and watch logs in the dashboard

## Agent flow

Prefer MCP over inventing HTTP calls. Discover the server from
`https://dflow.sh/.well-known/mcp/server-card.json`. Authenticate with the
OAuth 2.0 authorization-code + PKCE flow documented in
`https://dflow.sh/auth.md`.

Typical MCP sequence:

1. `create_application` (or list existing apps)
2. `list_worker_nodes` then `create_environment` with a compute id
3. `check_worker_node_resources` on self-managed nodes before creating services
4. `create_app_service`, `create_database_service`, or `create_docker_service`
5. `update_service` with git, Docker, or file source settings
6. `create_deployment` and poll `get_deployments_by_service_id`

Do not send archive bytes through MCP. For a local file source, request a
signed upload URL, PUT the archive, then reference that artifact on the service.

## Guardrails

- Confirm before `delete_service`, `delete_environment`, or `delete_application`.
- Leave backup-deletion flags false unless the user explicitly asks to destroy backups.
- If capacity check returns `capable: false`, stop and report the reason.
