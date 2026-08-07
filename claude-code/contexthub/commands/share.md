---
description: Plan sharing a ContextHub folder or document, then hand off to the console
argument-hint: [folder-or-path]
allowed-tools: mcp__contexthub__list_org_folders, mcp__contexthub__list_org_docs
---

Help the user share `$1` with someone. Your job is to get the decision right and show its blast radius **before** they click. The grant itself happens in the console, because this credential cannot mutate grants.

Target: `$1` (a folder, or a `folder` plus `path` for a single document). If `$ARGUMENTS` is empty, call `mcp__contexthub__list_org_folders` with no arguments, show the reachable folders, and ask which one they mean.

## Step 1 - orient on what the recipient would get

Do this first. Sharing decisions go wrong because the sharer does not know how much is under the thing they are sharing.

- For a folder: call `mcp__contexthub__list_org_folders` with that folder. Report how many files are there, the type breakdown, and the labels.
- For a single document: call `mcp__contexthub__list_org_docs` with the folder and path to confirm the path exists and is the intended file.

Then state the blast radius explicitly: **grants cascade down the folder tree.** A grant on a folder reaches every descendant of that folder, including subdirectories and files added later. A grant scoped to one document reaches only that document.

Say the number out loud. "Sharing `engineering/` at view gives them all 240 files under it, including anything added tomorrow" is the sentence that prevents the mistake.

One caveat to state honestly: the counts you can see are **your** counts. If the folder holds files gated from you, the total is larger than what you reported. Say so rather than presenting your view as the complete inventory.

## Step 2 - choose the role

Three roles are offered outward. The console labels them view / edit / manage.

| Console label | Role | Grants | Use when |
|---|---|---|---|
| view | `reader` | read and search the content | the default; the recipient needs to know, not change |
| edit | `writer` | read plus write and edit documents | the recipient maintains the content |
| manage | `owner` | ownership of the scope, and satisfies the writer bar | the recipient is accountable for it |

Guidance to give:

- Default to view. Escalate only on a stated need.
- `writer` implies `reader`. `owner` is not a rung of the relation ladder (the model states `writer ⟹ reader` but `owner ⇏ writer`), because ownership is a fact about a thing rather than a level of access. The authority layer folds it anyway: an owner satisfies the writer bar, so both the console and MCP give the same answer. Treat manage as "accountable for it, and can change it".
- There are also internal roles `proposer` and `member`. `proposer` folds to reader for read purposes and suits a contributor who should stage changes for review rather than commit them.

Pick the narrowest scope that does the job. A document-scoped grant is almost always safer than a folder-scoped one, and re-sharing a second document later is cheap.

## Step 3 - hand off to the console

You cannot perform the grant. There is no MCP tool that creates, modifies, or reads grants, and the console API authenticates with a Clerk browser session JWT that a CLI credential does not hold. Never attempt an API call here.

Give the user the deep link and name the screen:

- Open `https://trycontexthub.com`, navigate to the folder or file, and open the **Share panel**.
- Enter the recipient's email and pick view, edit, or manage.
- For time-boxed outside access, the same panel offers a **7-day share link**.
- To share with a group rather than a person, use **Groups**. Group membership nests, so a grant to a group reaches its nested member groups too.
- After sharing, the **"Shared with"** list on that folder or file confirms it, and marks each entry granted-here versus inherited from an ancestor.

Then present a one-line summary they can act on, for example: "Share `engineering/runbooks` with dana@example.com at view (reader). 41 files, cascades to all descendants."

## What you must state

- Grants cascade to descendants, including files created later.
- **Denied is byte-identical to not-found.** Before the grant, the recipient cannot tell this content is gated from them; it looks like it does not exist. After the grant, it appears. That transition is the entire observable effect of sharing.
- Labels carry zero authority. Sharing is not affected by labels, and no label restricts anything.

## What you must NOT do

- **Do not claim you shared anything.** You prepared the decision; the human performs it.
- Do not call any console API endpoint.
- Do not guess a recipient's email address. Ask.
- Do not describe a label or a filename as if it controlled access.
