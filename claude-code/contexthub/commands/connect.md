---
description: Connect this session to ContextHub and verify the connection with a real call
allowed-tools: mcp__contexthub__list_org_folders
---

Get the user from zero to a verified ContextHub connection. Work through the phases in order and stop as soon as the connection is proven working.

## Phase 1 - probe, do not lecture

Call `mcp__contexthub__list_org_folders` with no arguments. That single call answers almost everything, so make it first rather than asking the user about their setup.

Interpret the outcome:

| Outcome | Meaning | Go to |
|---|---|---|
| A list of folders comes back | Already connected and granted. Done. | Phase 4 |
| An empty list / "no folders ingested yet" | Connected and authenticated, but reaching nothing | Phase 3 |
| Auth error, 401, or the tool is not available at all | Not signed in yet, or the server is not configured | Phase 2 |

## Phase 2 - sign in through the browser

Tell the user to run `/mcp`, select `contexthub`, and choose the authenticate option. That opens a browser window where they sign in.

Explain these points, briefly:

- Auth is OAuth with Dynamic Client Registration. There is no token to mint, paste, or store. Clerk (`https://clerk.trycontexthub.com`) is the authorization server, and the client discovers it automatically from `/.well-known/oauth-protected-resource` on the MCP host.
- The production endpoint is `https://mcp.trycontexthub.com/mcp`.
- **An initial HTTP 401 is normal, not a failure.** An unauthenticated `POST /mcp` answers 401 with a `WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource"` header. That header is precisely what triggers the browser sign-in. Do not report it as a bug and do not start debugging it.
- Self-hosted deployments point somewhere else. The endpoint is a literal line in the plugin's `.mcp.json`, so if the user is not on the hosted service, that file is the knob. Keep the `/mcp` path.

After they report signing in, re-run Phase 1. Do not claim success on the strength of the browser flow alone; only a successful tool call proves it.

If the server itself looks unreachable, `https://mcp.trycontexthub.com/healthz` returns `{"ok":true,"vectorIndex":true,"paths":{"/mcp":"all"},"oauth":true}` when healthy.

## Phase 3 - signed in, but seeing nothing

This is a different problem from a broken connection, and saying so plainly is the whole value of this step. Authentication succeeded. The empty result means the credential's principal currently reaches no folders. Two likely causes, in order:

1. **No ContextHub organization membership.** Signing in with an account that belongs to no org is a valid sign-in with zero reach. The user needs to be a member of a ContextHub organization.
2. **Member of an org, but holding no grants.** Content access comes only from explicit grants (reach grants projected into OpenFGA), and nothing is readable by default. Someone with manage rights on a folder has to share it with them.

Point them at the console at `https://trycontexthub.com` to confirm org membership, and tell them to ask a folder owner to share via the Share panel.

State clearly: an empty list is never proof that the organization has no content. Denied is byte-identical to not-found, so a thin or empty view may simply mean everything is gated from this principal.

## Phase 4 - confirm and orient

Report concretely:

- Which endpoint is in use.
- How many folders this credential reaches, and name a few.
- Optionally, call `mcp__contexthub__list_org_folders` once more with a folder argument to show that folder's shape, including how many files are visible versus gated (`denied`).

Then point onward, without re-explaining them:

- `/contexthub:permissions` for what this credential reaches.
- `/contexthub:who-can-see` for who else can reach something.
- `search_org_knowledge` to prime a retrieve-then-cite pass.

**If the store is reachable but nearly empty, offer to seed it.** A connected user staring at an empty store is the most common dead end in first-touch, and the fix is one paste. Show them this, verbatim, in a code block so it is copyable:

```
Use ContextHub as our team's long-term memory. Call list_org_folders to see what
I reach. Then take everything you've learned about this project - architecture
decisions, gotchas, conventions, anything a new teammate would need - and write
each as its own doc with write_org_doc. Match the frontmatter (type + tags) of
neighboring files so they're findable. If you've genuinely learned nothing yet,
write one short doc capturing what this repo is and how to run it, so the store
isn't empty. Tell me what you wrote and where.
```

Say plainly that semantic search is already on server-side (`vectorIndex: true`); what a new store lacks is content, not capability. Do not run this for them unprompted - writing to a shared org store is the user's call.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Tools absent from `/mcp` entirely | Plugin installed but Claude Code not restarted since | Restart Claude Code — `/reload-plugins` does not bring up the MCP server |
| Tools vanished after a crash | Prior session died without a clean shutdown, so project plugin state was never written back | Restart. The credential is almost certainly still valid; do not re-auth first |
| 401 loop, sign-in never sticks | Browser flow was abandoned or cookies blocked | Re-run `/mcp`, complete the flow in a normal browser window |
| 404 on every call | Endpoint points at the console host instead of the MCP host | It must be `https://mcp.trycontexthub.com/mcp`, not `https://trycontexthub.com` |
| Connects, zero folders | No org membership or no grants | Phase 3 |
| "startup failed" in a non-Claude client | That client sends only configured headers and never walks the OAuth discovery chain, so it gets the initial 401 | Set `Authorization: Bearer cht_…` as a static header. See the headless note below |
| Works locally, fails in CI | No browser available for OAuth | See the headless note below |

## Headless fallback

For CI and other non-interactive contexts there is a legacy agent-token path: `Authorization: Bearer cht_…` over HTTP, or `CONTEXTHUB_TOKEN` in the environment for stdio. Mention it only when the user is genuinely headless. The plugin's committed configuration ships no token, and you must never write one into a file.

## Do not

- Do not treat the first 401 as an error.
- Do not call any console API endpoint. The console authenticates with a Clerk browser session JWT only, so a CLI credential gets 401 every time. Deep-link the human instead.
- Do not claim the connection works until a tool call has actually returned.
