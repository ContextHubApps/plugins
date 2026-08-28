---
name: catch-up
description: Work out what the user has been doing in agentleFS recently, so a fresh session can find the right documents fast instead of guessing. Use at the START of a session as soon as you know roughly what the user wants — especially when they say "catch me up", "what was I working on", "where did I leave off", "get up to speed", "remind me what's going on with X", or when they refer to ongoing work ("the pricing thing", "that migration", "my spec") as though you should already know what they mean. Also use before searching agentleFS when the request clearly concerns their current work but you have no idea which folder holds it, since orienting first is usually cheaper than searching blind.
---

# Catch up on what the user has been working on

A fresh session knows nothing about where this person lives in the store. It can
search, but a store holding several functions' worth of documents will answer
"pricing" from sales, product, finance and engineering at once, and the session
has no way to tell which one this user means.

`list_my_recent_work` closes that gap in one call. It reports which folders they
have been editing and opening, which documents they touched, and when. It does not
answer questions and it does not summarize anything — it tells you **where to
point the tools you already have.**

## The shape of a good catch-up

```
list_my_recent_work        ← once
   ↓ read the folders and filenames
search_org_knowledge       ← aimed, but not fenced in
   ↓
read_org_doc               ← two or three, then stop and talk
```

Four to six calls, then a reply. If you are on your seventh call you have stopped
orienting and started excavating.

## Step 1 — orient, once

Call `list_my_recent_work`. The default 30-day window suits most sessions; widen
it with `days` only if the user is asking about something they describe as older.

One call is the whole budget for this step. The output is small on purpose — if it
did not tell you what you hoped, calling it again with a different window will not
change that, and the user's own words are a better source anyway.

## Step 2 — read the basis line before you trust anything above it

Every response ends with something like `basis: 12 document(s) across 3 folder(s)`.
Read it first. It governs how much weight the rest deserves:

| Basis | What it means | What to do |
|---|---|---|
| Nothing at all | They have not used the store recently, or their grants changed | Say so plainly, then search normally. Do **not** guess at their work. |
| Under ~5 documents | A weak signal — real, but it could be noise | Let it break ties. Never let it steer. |
| A dozen or more across folders | A genuine picture of where they work | Aim your search with it. |

Adoption of agentleFS varies enormously — some people put everything in it, some
put in fragments. A confident-sounding account of someone's work built on four
events is worse than admitting you cannot tell, because the user has no way to see
that it was invented. When the basis is thin, the honest move is one sentence
saying so, and then getting on with the actual question.

## Step 3 — let it aim your search, never fence it

This is the part that goes wrong, so it is worth being precise about.

Orientation tells you where this person's work *tends* to live. That is a strong
hint and a terrible filter. If their activity is all in `product/` and they ask
about a deployment runbook, passing `location: "product"` to `search_org_knowledge`
guarantees you miss it — and you will never find out, because a search that
returns nothing looks exactly like a subject nobody wrote about.

So:

- **Use it to interpret.** "The pricing doc" means the one in *their* folder.
- **Use it to rank.** A hit in a folder they work in daily is likelier to be the
  one they meant than an identically-worded hit somewhere they have never been.
- **Use it to go first.** Search their active area before the rest of the store.
- **Do not use it to exclude.** Run at least one search unscoped by folder before
  concluding the store has nothing.

## Step 4 — open two or three documents, then stop

Orientation hands you filenames, and filenames are tempting. Resist opening them
all. Two or three well-chosen documents is almost always enough to answer or to
ask a good question, and every extra one costs context the user would rather spend
on the actual work.

Pick by what the user asked, not by what is most recent. If nothing looks right,
say what you found and ask — that is faster for them than watching you open six
files that turn out to be wrong.

## What the output is not

`list_my_recent_work` reports what someone **did**. It never says what they
**are**, and neither should you.

Writing eight documents in `engineering/` does not make someone an engineer. They
might be the founder, a designer who writes specs, a support lead documenting
bugs, or someone covering for a colleague on leave. If you decide they are an
engineer and quietly answer everything from there, that assumption shapes every
reply and the user cannot see it happening.

Read the counts as *places*, not as a *person*. And when the user says something
that cuts against them, the user is right — they know their job and this is a
30-day window of file activity.

## Reading what is missing

Two things the output genuinely cannot tell you, worth holding in mind:

**Recent often means finished.** The thing someone touched most recently is
frequently the thing they just wrapped up. Recency marks where they have *been*,
not where they are going. Their message says where they are going.

**Absence is never evidence.** Documents the user cannot currently read are
filtered out, including ones they wrote before their access changed, and folders
they hold no grant on never appear at all. An empty or thin result means "nothing
I can reach", never "nothing exists". Say it that way.

## Reporting back

Lead with the answer to what they asked. Weave the context in rather than
narrating your process — they want to know where things stand, not which tools you
called.

Cite documents by path so they can open them. If you leaned on orientation to pick
a folder, say so in passing ("looks like this lives in your `product/` area") so a
wrong guess is visible and correctable rather than silent.

## Do not

- Do not call `list_my_recent_work` repeatedly with different windows hunting for
  a better answer. One call, then move on.
- Do not pass a folder filter to search based only on orientation.
- Do not describe the user's role, seniority, or team from activity counts.
- Do not report a thin or empty result as though the organization has written
  nothing — see the `authorization-model` skill on why absence is not evidence.
- Do not open every filename the orientation returned.
