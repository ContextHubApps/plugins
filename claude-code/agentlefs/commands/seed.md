---
description: Seed agentleFS with what you already know about this project, so the store isn't empty
argument-hint: [folder]
allowed-tools: mcp__agentlefs__list_org_folders, mcp__agentlefs__list_org_docs, mcp__agentlefs__write_org_doc
---

Fill a reachable-but-empty agentleFS with durable knowledge about the current project. A new store has working semantic search and nothing to search — this closes that gap in one pass.

Target folder: `$1` (optional). If `$ARGUMENTS` is empty, pick the target in Step 1.

## Step 1 - find a folder you can write to

Call `mcp__agentlefs__list_org_folders` with no arguments.

| Outcome | Do this |
|---|---|
| One or more folders | If `$1` names one, use it. Otherwise show the list and ask which to write into. |
| Empty list | Stop. This credential reaches nothing — send them to `/agentlefs:connect`, and do not report the organization as having no content. |

Then call `mcp__agentlefs__list_org_docs` on the chosen folder. Two reasons: it tells you whether the store is actually empty, and it shows you the `type` and label conventions already in use.

## Step 2 - decide what is worth persisting

Take stock of what you have learned in this session and from the repository in front of you. Good candidates, roughly in descending value:

- Architecture decisions and the reasoning behind them, especially where the reasoning is not obvious from the code.
- Gotchas and non-obvious constraints — the things that cost someone an afternoon.
- Conventions a new teammate would otherwise violate.
- Operational knowledge: how to run it, how to deploy it, what breaks.

**Do not persist** what the code already says plainly, anything already in the store (you listed it in Step 1), secrets or credentials of any kind, or transient state like current branch names and in-flight work.

**If you have genuinely learned nothing durable yet, write exactly one document** describing what this repository is and how to run it. An empty store is the failure mode this command exists to fix; one honest orientation doc beats zero documents.

## Step 3 - write each as its own document

One idea per document. A single dumped file is a file nobody finds.

For each, call `mcp__agentlefs__write_org_doc` with the chosen folder, a descriptive path, and frontmatter that **matches the neighboring files you listed in Step 1** — `type` and `tags` especially. Tags drive search; they are not access controls. A mis-tagged file is a file nobody retrieves.

The write commits if your grant allows it and is refused if it does not. There is no staging step to choose and no review queue between you and the store, so a successful write is live and agent-visible immediately.

## Step 4 - report

List what you wrote, with folder and path. Name anything you deliberately skipped and why, including anything a refusal stopped you from writing.

Close by telling them the store is now searchable by meaning, and that `/agentlefs:permissions` shows what this credential reaches.

## Do not

- Do not invent knowledge to fill the store. Everything written must be something you actually established from this session or this repository.
- Do not write secrets, tokens, or credentials, even if you found them in the repo.
- Do not write one giant document instead of several small ones.
- Do not report a refused write as though it had landed.
