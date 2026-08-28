---
name: sync-conversation
description: Persist the current AI session's conversation into agentleFS as a durable document. Use when the user asks to sync, save, capture, persist, archive, or write up this conversation, session, chat, or thread to agentleFS; when they want a record of what was decided or figured out here; or when they say something like "put this in agentleFS" or "remember this for next time". Always asks where it should land, defaulting to the user's own home folder.
---

# Sync this conversation to agentleFS

A session ends and everything in it is gone. This turns the conversation into a
document the next agent — or the next person — inherits.

Two rules govern the whole thing:

1. **A summary, not a transcript.** Nobody re-reads a chat log. Write what was
   established and why.
2. **Never write without asking where.** You are about to put the user's words into
   a permissioned store. The destination decides who can read them, so it is the
   user's call, every time.

## Step 1 — orient

Call `list_org_folders` with no arguments. This is what the destination choice is
made from, and it is also the connection check.

| Outcome | Do this |
|---|---|
| One or more folders | Continue to Step 2. |
| Empty list | Stop. This credential reaches nothing to write into. Send them to `/agentlefs:connect`. Do **not** report the organization as having no content — see the `authorization-model` skill. |
| Error / tools missing | Stop and diagnose with the `connection` skill. Do not paraphrase a 401 as "agentleFS is down". |

## Step 2 — ask where it goes, defaulting to their home

**Default: the user's own home folder.** Every member of an org is provisioned a
private top-level folder named after them — "Manoj Singireddy", sitting beside
`context` and `engineering` — with a `home/` directory inside holding `notes/`,
`projects/`, `drafts/` and `reference/`. They hold owner and reader on it, and no
other member can read it, list it, or learn it exists.

That is the right default for a conversation, for a reason worth saying out loud:
a session contains half-formed reasoning, dead ends and things said in passing.
Landing it somewhere private and sharing it outward later is a decision the user
can still make. Landing it in a team folder is a decision already made for them.

So the default target is:

```
<Their Name>/home/notes/<descriptive-slug>.md
```

Spot the home folder in the Step 1 listing: it is the top-level folder named after
the person (display name, or the local part of their email — never the full
address). **If no such folder appears, do not invent one and do not write to a
folder you have not seen listed** — say the home is not visible to this credential
and ask where they want it instead.

Now ask, with the default pre-selected. Offer:

- **Their home** — `<Their Name>/home/notes/…`, private to them (recommended).
- **A shared folder** — name the specific reachable folders from Step 1 that plausibly
  fit, and say plainly that anyone granted on that folder will be able to read the
  conversation.
- **Somewhere else** — let them name the full location.

Confirm the exact `location` back to them before writing. A path is cheap to
get right now and expensive to move later: moving a file re-permissions it.

## Step 3 — read the neighbors

Call `list_org_docs` on the chosen folder. Two things come from this:

- The `type` convention already in use. **Match it.**
- Whether a document for this conversation already exists. If the user is re-syncing
  an ongoing thread, write to the **same path** rather than minting `notes-2.md`.

### Tags are not in that listing

The listing renders one line per file — `- path [type] (asset) title` — and **no
tags**. Reading a tag convention off it is not possible; an agent that tries will
either invent unrelated tags or quietly skip the matching it was asked to do, with
nothing signalling that the data was never there.

Tags come from opening a neighbor. Pick one or two of the most representative files
from the listing and call `read_org_doc` on them: it reattaches the document's own
frontmatter, so the tags you see are the ones somebody authored on that file.

Copy those. Do **not** copy the folder's label list from `list_org_folders <folder>` —
that set is *resolved*, meaning it includes labels inherited from the folder and the
directories above it. Restating an inherited label in your own frontmatter converts it
into an authored one, which then survives being removed from the folder. Inherited
labels already apply to your document without being written down.

Tags drive search and carry no authority whatsoever; a mis-tagged file is a file
nobody retrieves, and a well-tagged one grants nobody anything.

### `type` is a fixed list, and a wrong value fails silently

There are exactly seven, and no others exist:

```
meeting-notes | playbook | spec | brand-asset | web-clip | contract | misc
```

Anything else — `conversation`, `session-summary`, whatever reads best — is **not
rejected**. It is silently coerced to `misc`, with nothing said about it in the write
response, so the document lands in a bucket you did not choose and never find out.

Expect to have nothing to copy from. A fresh home holds one file, `home/welcome.md`,
and the folder you are writing into may hold none at all — "match the neighbors" has
no answer there. When that happens, pick from the list above rather than inventing:
`meeting-notes` is the closest bucket for a record of a working session, and `misc`
is the honest choice when it is not.

`tags` are free-form, so put the specifics there.

## Step 4 — compose the document

One conversation, one document. Frontmatter matching the neighbors, then a body
along these lines:

- **What was asked** — the problem in the user's terms, not yours.
- **What was decided or established** — the durable part. Conclusions, and the
  reasoning behind them where the reasoning is not obvious from the outcome.
- **What was tried and rejected**, with why. This is the highest-value section and
  the one most often dropped; it is what stops the next person repeating it.
- **What changed** — files, commands, migrations, config, if any.
- **Open threads** — what is unresolved, and what the next step would be.

Keep it readable in two minutes.

**Strip before writing:**

- Secrets, tokens, API keys, connection strings, credentials of any kind — including
  ones that appeared in tool output rather than in the chat.
- Personal or sensitive information about third parties that has no bearing on the
  conclusion.
- Raw tool dumps, full file contents, and long stack traces. Quote the one line that
  mattered.
- Anything the user asked you not to record. Ask if you are unsure about a passage.

If in doubt about a passage, leave it out and say you did.

## Step 5 — write

Call `write_org_doc` with the confirmed `location`.

It commits if their grant allows it and is refused if it does not — there is no
staging option and no review queue, so a successful write is live immediately. If it
is refused, say so plainly and say where they could write instead.

## Step 6 — report

State the folder and the full path, and **hand over the link the write returned** —
the confirmation ends with `— cite: <url>`, and that is the openable half. A commit
hash is not something anyone can click, and a document nobody opens is a document
that may as well not have been written.

The confirmation also says whether it CREATED the document or OVERWROTE one, and
names any directory that did not exist before. Read both back before you report:
"overwrote" on a document you meant to create, or a directory you did not expect to
be new, means you wrote somewhere other than where you meant to — say so rather than
reporting a clean write.

Name what you deliberately left out. If it went to their home, remind them it is
private to them and that `/agentlefs:share` is how it reaches anyone else.

## Do not

- **Do not pick the destination silently.** Even when the answer is obvious, ask.
- Do not write a verbatim transcript, and do not paste whole files into the document.
- Do not write secrets, even ones the user pasted themselves.
- Do not invent a home folder that did not appear in `list_org_folders`.
- Do not create a second document for a conversation that already has one.
- Do not report a refused write as though it had landed.
- Do not claim you shared it with anyone. Writing is not sharing; grants happen in
  the console.
