---
name: dflow-mcp
description: Connect an OAuth-capable MCP client to a dFlow workspace to create apps, environments, services, and deployments.
---

# dFlow MCP

Use this skill when an agent needs to operate a dFlow workspace: applications,
environments, services, deployments, domains, scale, or database backups.

## Endpoints

- Transport: streamable HTTP at `https://app.dflow.sh/api/mcp`
- Server card: `https://dflow.sh/.well-known/mcp/server-card.json`
- OAuth discovery: `https://dflow.sh/.well-known/oauth-authorization-server`
- Protected resource: `https://dflow.sh/.well-known/oauth-protected-resource`
- Human + agent auth notes: `https://dflow.sh/auth.md`
- Product docs: https://docs.dflow.sh/articles/3167609-dflow-mcp

## Authentication

dFlow MCP is OAuth-protected. Do not invent API keys.

1. Register an OAuth client at `https://app.dflow.sh/api/oauth/register`
   (RFC 7591). Public clients use `token_endpoint_auth_method: "none"`.
2. Send the user through `https://app.dflow.sh/oauth/authorize` with
   `response_type=code`, PKCE `S256`, and scope `mcp`.
3. Exchange the code at `https://app.dflow.sh/api/oauth/token`.
4. Call MCP with `Authorization: Bearer <access_token>`.

Tokens are RS256 JWTs issued by `https://app.dflow.sh`. JWKS lives at
`https://app.dflow.sh/.well-known/jwks.json`.

## Tools

The server exposes tools for templates, GitHub integrations, Docker
registries, applications, worker nodes, environments, services, deployments,
and backups. Resolve a service by id or case-insensitive name before logs or
deploy actions.

## Workflow reminder

Create application → attach compute via environment → check capacity →
create service → update source → deploy → poll. Template deploy materializes
and deploys services itself; do not call `create_deployment` afterward.
