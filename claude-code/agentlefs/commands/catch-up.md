---
description: Catch up on what you have been working on in agentleFS, and pick the thread back up
argument-hint: [topic]
allowed-tools: mcp__agentlefs__list_my_recent_work, mcp__agentlefs__list_org_folders, mcp__agentlefs__search_org_knowledge, mcp__agentlefs__read_org_doc
---

Work out where the user left off and hand it back to them in a few sentences they can act on.

Focus: `$1` (optional). If `$ARGUMENTS` names a topic, let it steer everything below — it is a far better signal than recency. If it is empty, report on their recent work generally.

## Step 1 - orient, once

Call `mcp__agentlefs__list_my_recent_work`. Default window is 30 days; pass `days` only if they asked about something older.

One call. If it did not tell you what you hoped, calling it again with a different window will not change that.

| Outcome | Do this |
|---|---|
| Documents and folders come back | Continue to Step 2. |
| Empty, and no folders moved | They have not used the store recently, or their grants changed. Say so in one sentence and go to Step 3 anyway — `$1` may still be answerable from the store. |
| Error / tools missing | Stop and diagnose with the `connection` skill. Do not paraphrase a 401 as "agentleFS is down". |

## Step 2 - read the basis line before you trust anything above it

The response ends with something like `basis: 12 document(s) across 3 folder(s)`. It governs how much weight the rest deserves.

Under about five documents is a weak signal — real, but close to noise. Let it break ties and nothing more. A dozen or more across several folders is a genuine picture of where this person works.

This matters because adoption varies enormously: some people put everything in agentleFS, some put in fragments. A confident account of someone's work built on four events is worse than admitting you cannot tell, because they have no way to see it was invented. When the basis is thin, say so in one clause and move on.

## Step 3 - search, aimed but not fenced

Orientation tells you where this person's work *tends* to live. That is a strong hint and a terrible filter.

If their activity is all in `product/` and `$1` is about a deployment runbook, scoping `mcp__agentlefs__search_org_knowledge` to `product` guarantees you miss it — and you will never find out, because a search returning nothing looks exactly like a subject nobody wrote about.

So use it to **interpret** ("the pricing doc" means the one in their folder), to **rank**, and to **go first**. Run at least one search without a folder scope before concluding the store has nothing.

## Step 4 - open two or three documents, then stop

Orientation hands you filenames and they are tempting. Two or three well-chosen ones is almost always enough to answer or to ask a good question, and every extra one costs context they would rather spend on the actual work.

Call `mcp__agentlefs__read_org_doc` on the ones that match what they asked, not the ones that are merely most recent. If nothing looks right, say what you found and ask — that is faster for them than watching you open six files that turn out to be wrong.

## Step 5 - report

Lead with where things stand, not with what you did. Three or four sentences, then stop.

Cover what they were last working on, anything that looks unfinished, and what the obvious next step is. Cite documents by folder and path so they can open them. If you leaned on the orientation to pick a folder, say so in passing ("looks like this lives in your `product/` area") — a wrong guess should be visible and correctable rather than silent.

Two things to keep straight when you write it up:

**Recent often means finished.** The most recently touched document is frequently the one they just wrapped up. Recency marks where they have been, not where they are going. If `$1` is set, it outranks everything the orientation said.

**Absence is never evidence.** Documents they cannot currently read are filtered out, and folders they hold no grant on never appear. Empty means "nothing I can reach", never "nothing exists".

## Do not

- Do not describe their role, seniority, or team from activity counts. Writing eight documents in `engineering/` does not make someone an engineer — they might be the founder, a designer writing specs, or covering for someone on leave. Read the counts as places, not as a person.
- Do not pass a folder filter to search based only on orientation.
- Do not call `mcp__agentlefs__list_my_recent_work` repeatedly hunting for a better answer.
- Do not open every filename the orientation returned.
- Do not report a thin or empty result as though the organization has written nothing.
- Do not narrate your tool calls. They want to know where things stand.
