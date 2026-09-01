# agentleFS plugins

Plugins that connect AI coding agents to [agentleFS](https://agentlefs.com), a
permissioned org-context store. Your agent reaches what you are granted and nothing
else, and the authorization filter runs inside the query rather than as an
afterthought.

> **Generated repo.** Published from the agentleFS source repo on release. Do not
> open PRs here, edits are overwritten by the next publish. File issues at
> [agentlefs.com](https://agentlefs.com).

## Why bother

Your agent already knows how software works. It knows nothing about **how your company
works** - why you picked Postgres over Dynamo, who owns the billing service, what the
last postmortem concluded. Ask it today and you get confident generic advice, or an
apology about not having access to your internal docs.

agentleFS is where that knowledge lives, filtered to what you're actually granted. The
plugin is what makes your agent reach for it unprompted, and - just as importantly - what
stops it from telling you "your org never documented this" when the honest answer is "I
can't see that."

## Claude Code and Claude Desktop (3 steps, about a minute)

**1. Add the marketplace and install.**

```
/plugin marketplace add ContextHubApps/plugins
/plugin install agentlefs@agentlefs
```

**2. Restart Claude Code.** The plugin's MCP server only comes up on a fresh start.
Skipping this is the single most common reason the tools don't appear.

**3. Connect.**

```
/agentlefs:connect
```

A browser window opens. Sign in. That's it.

**There is no token to mint or paste.** Auth is OAuth with dynamic client registration,
so the client registers itself and this repo ships no credential of any kind.

> **You'll see an HTTP 401 first, and that's normal.** The 401 carries the header that
> *triggers* the browser sign-in - it's the mechanism working, not a failure. Don't debug it.

Once you're in, `/agentlefs:connect` verifies with a real call and tells you which state
you're in: connected and granted, connected but reaching nothing, or not signed in. Those
three need completely different fixes.

### Copy-paste: fill the store

New store with nothing in it? Semantic search works, but it has nothing to search. Paste
this to seed it from what your agent already learned about your project:

```
Use agentleFS as our team's long-term memory. Call list_org_folders to see what
I reach. Then take everything you've learned about this project - architecture
decisions, gotchas, conventions, anything a new teammate would need - and write
each as its own doc with write_org_doc. Match the frontmatter (type + tags) of
neighboring files so they're findable. If you've genuinely learned nothing yet,
write one short doc capturing what this repo is and how to run it, so the store
isn't empty. Tell me what you wrote and where.
```

### Copy-paste: make it a habit

```
From now on, check agentleFS before answering anything about how *we* do things,
and write durable conclusions back with write_org_doc so the next session inherits
them. If a search comes back thin, say it may be a permission boundary rather than
telling me nothing exists.
```

| Command | Answers |
|---------|---------|
| `/agentlefs:connect` | Get me connected, and tell me why I see nothing |
| `/agentlefs:permissions` | What does *this* credential actually reach? |
| `/agentlefs:share` | What am I about to share, and how far does it cascade? |
| `/agentlefs:who-can-see` | Who can see this? |
| `/agentlefs:catch-up` | What was I working on, and what's the next step? |
| `/agentlefs:seed` | The store is empty - fill it from what the agent already learned |

Also included, and triggered by what you ask rather than by a slash command:

| Skill | Fires when |
|---|---|
| `catch-up` | You refer to ongoing work as though the agent should already know — the same ground as `/agentlefs:catch-up`, without having to ask for it |
| `sync-conversation` | You want this session written back to the store as a durable document |
| `deleting` | Something is about to be removed, and the blast radius needs saying out loud first |
| `authorization-model` | Anything turns on who can read what, or a result comes back thinner than expected |
| `connection` | The tools are missing, returning 401, or connected but reaching nothing |
| `publishing` | A folder is to be packaged for reuse elsewhere — including the difference between publishing it and simply letting someone read it |

Plus two agents: `context-researcher` for grounded retrieve-and-cite work, and
`access-reviewer` for read-only access review — read-only by tool allowlist rather than
by instruction.

**Self-hosted?** The endpoint is a single literal line in
`plugins/agentlefs/.mcp.json` - point it at your own host (keeping the `/mcp`
path) and load the directory with `claude --plugin-dir`. It is deliberately not a
templated value, because the interpolation Claude Code supports is not portable to
other agents and produced a server that silently never connected.

## Codex

Same package, same marketplace, same browser sign-in. Codex registers a marketplace
from the SHELL and installs from inside the session, which is the one step people
expect to work the other way round.

**1. Add the marketplace** in a terminal:

```
codex plugin marketplace add ContextHubApps/plugins
```

**2. Install it.** In the CLI, start `codex` and run `/plugins` to open the plugin
browser, then install **agentleFS** from the agentleFS marketplace. In Codex in the
ChatGPT desktop app, restart the app first - it reads its marketplace sources at
startup - then install from the Plugins directory.

**3. Sign in** when Codex asks. A browser window opens, and there is no token to paste
here either.

What Codex loads from the package is `skills/` and the MCP server. The `commands/` and
`agents/` directories are Claude Code artifacts and are simply not read - they ship in
the same directory because one package serves both ecosystems, not because Codex has
them. `codex mcp list` will not show the server, and that is not a failed install: a
plugin-provided server is launched by the plugin rather than configured as an
`[mcp_servers.*]` entry in `config.toml`.

**The ChatGPT web and mobile apps are a different path.** They install from the
universal public plugin directory, which is a submission-and-review process rather
than a repository you can add, so a marketplace does not reach them. Until agentleFS is
listed there, use the MCP server URL directly in developer mode.

## Other agents

agentleFS is a plain MCP server, so any MCP-capable client can connect to
`https://mcp.agentlefs.com/mcp` today - OAuth for humans, or a `afs_` bearer
token for headless and CI use. Cursor, Gemini CLI, VS Code and Cline take that URL;
Cursor's own plugin format needs a manual review before listing, and is its own job.

**If your client doesn't do OAuth, you need a bearer token.** Many desktop clients
send only the headers you configure and never walk the discovery chain, so they get
the initial 401 and report "startup failed." Mint a token and set one header:

| Field | Value |
|---|---|
| URL | `https://mcp.agentlefs.com/mcp` |
| Header name | `Authorization` |
| Header value | `Bearer afs_...` |

Scope it by minting against a narrow principal - a token in a desktop app's config
carries that principal's roles, and it's revocable if the config ever leaks.

## The one thing worth knowing

**Denied is byte-identical to not-found.** agentleFS exposes no existence oracle: a
document you cannot read answers exactly like a document that does not exist, and a
folder you do not reach is simply absent. That is deliberate - the alternative leaks
the shape of what you cannot see.

The practical consequence is that **a thin result is usually a permission boundary,
not a bug**. Every command and skill here states that rule, because it is the single
most misread behavior in the product. If a search comes back empty, the honest reading
is "nothing I can reach matches", never "the organization has not written this down".

## License

MIT
