---
description: Answer who can see a ContextHub folder or document, honestly split by what is knowable here
argument-hint: [folder-or-path]
allowed-tools: mcp__contexthub__list_org_folders, mcp__contexthub__list_org_docs
---

Answer "who can see `$1`". The honest answer has two halves and they are not equally knowable from here. Keep them separate and never let the second half drift into speculation.

Target: `$1`. If `$ARGUMENTS` is empty, call `mcp__contexthub__list_org_folders` with no arguments, list the reachable folders, and ask which one they mean.

## Part 1 - what YOU can see (answerable now)

Call `mcp__contexthub__list_org_folders` with the folder. For a specific document, also call `mcp__contexthub__list_org_docs` with the folder and path.

Report:

- How many files in this folder you can read.
- How many are `denied`, meaning gated from you.
- The type and label breakdown.

This is a fact about **your** principal, established by a real call. Present it as such.

The gated count is the interesting number. It tells you how many files sit in this folder that you cannot see, and nothing else. Not their names, not their paths, not their subject matter. Do not speculate about them.

## Part 2 - who ELSE can see it (requires the console)

State the limitation plainly rather than working around it: **there is no MCP tool that returns grants.** That is deliberate, not an oversight. A token holder should not be able to enumerate a tenant's access, so no such tool exists, and the audit surface is console-only and admin-gated. The console API authenticates with a Clerk browser session JWT that this credential does not hold, so you must not attempt an API call for this either.

Send the human to `https://trycontexthub.com` and name the screen that answers their actual question:

| Question | Console screen |
|---|---|
| Who has a grant on this, and is it granted here or inherited? | the **"Shared with"** list on the folder or file |
| What exactly can a specific person or group see, file by file? | the **reach lens** ("View as") on the Files tree |
| How much is visible versus denied, by type and label, and who can reach it? | the **share summary** panel |
| Who is in a group, and what does the group reach? | **Groups** |
| Who is a console admin versus a member? | **Access → Roles** |

The reach lens is usually the right recommendation, because it shows a chosen principal's effective per-file access **with provenance**, which is what people mean when they ask this question.

## Explain how to read the console answer

Give the user these three ideas, because a grant list is misleading without them:

- **Granted-here versus inherited.** A grant made directly on this scope shows as granted-here. One arriving from an ancestor folder shows as inherited. Removing an inherited grant means finding the ancestor it was made on; there is nothing to remove here.
- **Cascade.** Grants flow down the folder tree. Someone with a grant three levels up reaches this file without ever appearing to have been given it directly. The "Shared with" list marks that as inherited.
- **Group nesting.** A grant to a group reaches its members, and groups nest, so a person can reach a file through a group inside a group. The reach lens resolves this; a raw grant list does not.

Also worth stating: console role and file access are different systems. Being a console `admin` (Cerbos RBAC) decides which console tools a person may use. It does not grant read access to files. Only reach grants do that.

## What you must NOT do

- **Never guess at a name.** Do not say "probably the engineering team" or "likely whoever owns this folder." If you do not have it from a tool result, you do not have it.
- Do not call any console API endpoint. No `/api/reach/grants`, no `/api/share/summary`, no `/api/folders/reach`. Every one would 401.
- Do not present your own reach as the full picture. Other principals may reach far more than you, or far less, and you cannot see which.
- Do not infer access from labels. Labels carry zero authority; an unlabeled file is not public.
- Do not read a thin result as "nothing exists here." **Denied is byte-identical to not-found.** A folder that looks nearly empty to you may be full of content gated from you.

## Shape of the answer

1. What you can see, with real numbers from the calls you made.
2. What you cannot determine from here, and why the tool deliberately does not exist.
3. The console screen that answers it, plus how to read granted-here versus inherited.
