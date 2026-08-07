---
name: connection
description: How the ContextHub MCP connection authenticates, and how to diagnose it. Use when ContextHub tools are unavailable, return 401, 404, or auth errors; when the user asks how to connect, sign in, or authenticate to ContextHub; when a connection succeeds but returns no folders; when configuring a self-hosted endpoint; or when setting up headless/CI access. Also use before concluding that ContextHub is broken.
---

# ContextHub connection

## Endpoints

| What | URL |
|---|---|
| MCP server (production) | `https://mcp.trycontexthub.com/mcp` |
| Health check | `https://mcp.trycontexthub.com/healthz` |
| Console (humans only) | `https://trycontexthub.com` |
| Authorization server | `https://clerk.trycontexthub.com` |

Healthy `GET /healthz` returns:

```json
{"ok":true,"vectorIndex":true,"paths":{"/mcp":"all"},"oauth":true}
```

`vectorIndex: true` means semantic search is available. Without it, `how="meaning"` degrades to text search rather than failing.

## The auth flow: OAuth with Dynamic Client Registration

There is **no token to mint, paste, or store**. The user signs in in a browser and the client registers itself.

Discovery chain:

1. Client sends an unauthenticated `POST /mcp`.
2. Server answers **HTTP 401** with `WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource"`.
3. Client fetches that `/.well-known/oauth-protected-resource` document on the MCP host.
4. That document points at Clerk as the authorization server, including its `registration_endpoint`.
5. Client registers dynamically at that endpoint, then runs the browser authorization flow.
6. The resulting access token is sent as a bearer on subsequent `POST /mcp` calls.

### The 401 is not an error

**That first 401 is the normal trigger for browser sign-in.** It is the mechanism, not a fault. Never report it as a bug, never start debugging it, and never conclude the server is broken because of it. You should only be concerned if 401s persist *after* a completed browser sign-in.

## Managing the connection in-session

Use the `/mcp` command in Claude Code to inspect status and authenticate. Select `contexthub` and choose the authenticate option to launch the browser flow. The same command shows whether the server is connected and which tools it exposes.

## Self-hosted deployments

The endpoint is a literal `url` in the plugin's `.mcp.json`. Self-hosted deployments edit that one line, keeping the `/mcp` path. Nothing else needs changing; discovery is relative to whatever host is configured, so a self-hosted deployment advertises its own authorization server through its own `/.well-known/oauth-protected-resource`.

It is deliberately a literal rather than a templated value. Claude Code can interpolate `${user_config.…}` there, but no other agent does: Codex installs the same plugin and reads that string verbatim, producing a server that can never connect and reports no error. A hardcoded URL is correct in every client.

## Prerequisite: organization membership

Signing in is not sufficient. The user must be a **member of a ContextHub organization**, and within it must hold grants. A valid sign-in with no org membership authenticates fine and reaches nothing. This looks like a working connection returning an empty world, which it is.

## Headless and CI fallback

For non-interactive contexts where no browser exists, there is a legacy agent-token path:

| Transport | How |
|---|---|
| HTTP | `Authorization: Bearer cht_…` |
| stdio | `CONTEXTHUB_TOKEN` environment variable |

Use this only when genuinely headless. **The plugin's committed configuration ships no token**, and a `cht_` token must never be written into a committed file. Treat it as a secret supplied by the environment.

## Verifying a connection

Only a successful tool call proves a working connection. Call `list_org_folders` with no arguments. Completing the browser flow is not proof, and neither is a green status in `/mcp`.

## Symptom to cause

| Symptom | Cause | Fix |
|---|---|---|
| Single 401, then a browser opens | Normal DCR trigger | Nothing. Complete the sign-in. |
| 401 loop that never resolves | Browser flow abandoned, cookies blocked, or a stale registration | Re-run `/mcp` and authenticate again in a normal browser window |
| 401 in CI or a headless shell | No browser for the OAuth flow | Use the `cht_` token path above |
| 404 on every call | Pointed at the console host instead of the MCP host | Must be `https://mcp.trycontexthub.com/mcp`. `https://trycontexthub.com` is the human console and serves no MCP. |
| 404 on a self-hosted deployment | the `url` in `.mcp.json` is missing the `/mcp` path | The path matters, not just the host |
| Connected, zero folders | No org membership, or membership with no grants | Confirm membership in the console; ask a folder owner to share via the Share panel |
| Connected, folder looks nearly empty | Content is gated from this principal | Expected. Denied is byte-identical to not-found. |
| `how="meaning"` silently searched by text | Deployment has no vector index | Check `vectorIndex` in `/healthz`. Degradation is intentional. |
| Tools are absent from `/mcp` entirely | Plugin not enabled, or the server is unreachable | Check plugin state, then `/healthz` |
| `registry_*` tools missing | Off unless the deployment sets `REGISTRY_TOOLS=on` | Expected. Never rely on them. |
| A read errored instead of returning fewer rows | Fail-closed: a truncated allow-set throws | Surface it. The system refused to under-report. |

## Never do this

- **Never call a console API endpoint from an agent context.** The console API (`apps/api/src/server.ts`) authenticates with a Clerk browser session JWT only. An OAuth or `cht_` credential gets 401 every time. For anything needing the cross-principal view, deep-link the human to `https://trycontexthub.com`.
- Never diagnose "connected but empty" as a broken connection. It is an authorization outcome, and saying so correctly is the difference between a useful answer and a wasted hour.
- Never conclude the store is empty from an empty view. See the `authorization-model` skill.
