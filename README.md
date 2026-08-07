# ContextHub plugins

Plugins that connect AI coding agents to [ContextHub](https://trycontexthub.com), a
permissioned org-context store. Your agent reaches what you are granted and nothing
else, and the authorization filter runs inside the query rather than as an
afterthought.

> **Generated repo.** Published from the ContextHub source repo on release. Do not
> open PRs here, edits are overwritten by the next publish. File issues at
> [trycontexthub.com](https://trycontexthub.com).

## Claude Code

```
/plugin marketplace add ContextHubApps/plugins
/plugin install contexthub@contexthub
/contexthub:connect
```

Auth is OAuth with dynamic client registration. **There is no token to mint or
paste** - `/contexthub:connect` triggers a browser sign-in, and this repo ships no
credential of any kind.

| Command | Answers |
|---------|---------|
| `/contexthub:connect` | Get me connected, and tell me why I see nothing |
| `/contexthub:permissions` | What does *this* credential actually reach? |
| `/contexthub:share` | What am I about to share, and how far does it cascade? |
| `/contexthub:who-can-see` | Who can see this? |

Also included: two skills (`authorization-model`, `connection`) that carry the real
authorization model, and two agents (`context-researcher` for grounded retrieve-and-cite
work, `access-reviewer` for read-only access review, read-only by tool allowlist rather
than by instruction).

**Self-hosted?** The endpoint is a single literal line in
`claude-code/contexthub/.mcp.json` - point it at your own host (keeping the `/mcp`
path) and load the directory with `claude --plugin-dir`. It is deliberately not a
templated value, because the interpolation Claude Code supports is not portable to
other agents and produced a server that silently never connected.

## Other agents

ContextHub is a plain MCP server, so any MCP-capable client can connect to
`https://mcp.trycontexthub.com/mcp` today - OAuth for humans, or a `cht_` bearer
token for headless and CI use. Client-specific packaging lands here as its own
directory (`codex/`, and so on) as each client's extension model supports it.

## The one thing worth knowing

**Denied is byte-identical to not-found.** ContextHub exposes no existence oracle: a
document you cannot read answers exactly like a document that does not exist, and a
folder you do not reach is simply absent. That is deliberate - the alternative leaks
the shape of what you cannot see.

The practical consequence is that **a thin result is usually a permission boundary,
not a bug**. Every command and skill here states that rule, because it is the single
most misread behavior in the product. If a search comes back empty, the honest reading
is "nothing I can reach matches", never "the organization has not written this down".

## License

MIT
