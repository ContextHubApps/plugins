---
name: publishing
description: How to publish folders and files as reusable packages in the agentleFS registry, and how to install them. Use when the user asks to publish, package, share publicly, distribute, or "make available to other teams" any folder or set of documents; when they ask how to reuse content across organizations; when a publish is refused with HANDLE_UNCLAIMED, HANDLE_NOT_YOURS, PACKAGE_NOT_YOURS, TOO_LARGE or a visibility error; when they ask about @publisher handles, package versions, or the public catalog; or when deciding whether a request means "publish" or merely "let someone read this".
---

# Publishing to the agentleFS registry

The registry is the abstraction for **publishing a directory or a set of files so other people can reuse them**. A package is an immutable, versioned snapshot of files taken from one folder, addressed as `@publisher/package@version`.

## First: is this actually a publish?

Most requests that sound like publishing are not. Get this wrong and you copy content out of an organization permanently when the user wanted a link.

| The user wants | The right move | Not this |
|---|---|---|
| A colleague to read a document | A grant on the file or folder | A package |
| To save something durable | `mcp__agentlefs__write_org_doc` | A package |
| One team to see a folder | A grant on that folder | A package |
| Content **reused** in other workspaces — a handbook, a prompt library, a policy set, docs | `mcp__agentlefs__registry_publish` | A grant |
| The same content installed in many places, tracking updates | Publish, then install per site | Copying files |

The test: **does someone need their own copy of this, somewhere else?** If they only need to *read* it where it already lives, that is a grant, and the `authorization-model` skill covers it.

## The three gates

A publish passes three independent checks. Knowing which one refused you is the difference between a fix and a retry loop.

1. **Organization ownership.** Publishing requires the same authority as the console's publish button — ownership of the organization, not merely membership. A member with write access to the files still cannot publish them.
2. **Owner reach on every file.** The engine checks you own and can read *every* file under the root. All-or-nothing, counting files you cannot see.
3. **The acknowledgement**, for anything that leaves the organization. See below.

## Visibility, and the one irreversible choice

Publishing copies **file bodies**, not paths, out of the organization's isolation boundary. That copy is permanent: versions are immutable and there is no unpublish tool on this surface.

| Visibility | Who can see it | Leaves the org? |
|---|---|---|
| `private` (default) | This organization only | No |
| `unlisted` | Anyone with the exact package name. Never listed in the catalog. | **Yes** |
| `public` | Listed in the public catalog, readable by anyone, signed in or not | **Yes** |

`unlisted` is **not** "not published". It serves bodies to anyone holding the name.

So `unlisted` and `public` both require `acknowledge_public: true`, and that argument means one specific thing:

> **The user told you this content may leave the organization.**

Never set it on your own initiative, never set it because a refusal asked for it, and never infer it from "publish this" alone — that sentence is satisfied by `private`. Ask, in terms of the consequence: *"this makes the file contents readable by anyone on the internet, permanently, and it cannot be undone — confirm?"*

Omitting `visibility` publishes a NEW package `private`, and leaves an EXISTING package's audience exactly as it was. Omission never *changes* who can see a package — but on a package that is already public or unlisted, the version you are adding is published to that audience, so **the acknowledgement is still required**. Omitting is not a way around it: `registry_publish` resolves the audience first and refuses if it is not private.

**Visibility belongs to the package, not the version.** One value is read for every version, so changing it on a republish changes what people can see of versions already out there — which is why omission means "leave it" rather than "make it private". Naming it, though, still applies to the whole package:

- `visibility: "public"` on a package that was private makes **every earlier version** public at once, and those bodies leave the organization permanently. The acknowledgement covers this, which is why it is required.
- `visibility: "private"` on a package that was public **hides every earlier version** — readers who had `1.0.0` start getting a 404. Legitimate as a deliberate act, and not something to do in passing while adding a version.

So on a republish: omit `visibility` unless the user asked to CHANGE the audience — and expect to be asked for the acknowledgement anyway if the package is public or unlisted. If they did ask to change it, say which direction it moves and that it applies to every version, not just the new one.

## The workflow

### 1. Find out which handle you can publish under

```
mcp__agentlefs__registry_handles
```

A package name is `@publisher/package` and **both halves are required**. Handles are owned globally, first-come — `@anthropic` is not free to take — so publishing under one you do not own is refused rather than silently renamed.

If the list is empty, claim one with `mcp__agentlefs__registry_claim_handle`. Confirm the exact spelling with the user first: the claim is global, permanent, and there is no release.

### 2. Choose the root and list its files

```
mcp__agentlefs__list_org_folders          # what folders exist
mcp__agentlefs__list_org_docs             # what is in the one you picked
```

### 3. Publish

```
mcp__agentlefs__registry_publish
  name: "@acme/handbook"
  version: "1.0.0"
  root: "handbook"
  include: ["onboarding/day-one.md", "policies/travel.md"]
  description: "Acme's onboarding handbook"
```

**`include` paths are relative to `root`.** This is the one argument on the whole agentleFS surface that is not a path from the workspace root, and it is the most common way to get a publish wrong. With `root: "handbook"`, the entry is `onboarding/day-one.md` — not `handbook/onboarding/day-one.md`, which is what `list_org_docs` handed you. A full path under the root is accepted and trimmed for you, but do not rely on that: build the list deliberately.

Why it matters more than a normal typo: an unreadable path is kept in the package with an **empty body** rather than dropped, so a wrong prefix publishes a real version in which every file is blank — at a version number that can never be reused. The tool reports how many bodies actually landed; read that number rather than stopping at "published".

### 4. Report what happened

Quote the package name, version and visibility back to the user. If the result carries an empty-file warning, say so plainly and publish a corrected version at a new version number — the bad one cannot be replaced.

## Installing a package

```
mcp__agentlefs__registry_browse                     # the catalog
mcp__agentlefs__registry_install                    # into a site you own
mcp__agentlefs__registry_retrieve                   # which version a site serves
```

**The install site is the ACL.** Installing does not copy files into the organization; it links the package at a folder, and that folder's permissions decide who can read it. So install into a folder whose audience already matches who should see the package, and check that before installing rather than after.

Pin a `version` for reproducibility, or follow a `channel` to track the publisher's head. Pin unless the user asked to track updates.

`registry_browse` shows public packages plus your own organization's at any visibility. A `[private]` or `[unlisted]` marker on a row means **do not recommend that package onward** — the person you are talking to may see it and the next person may not.

## Decoding a refusal

| Reason | What to do |
|---|---|
| `NAME_INVALID` | The name needs both halves: `@publisher/package`. |
| `HANDLE_UNCLAIMED` | Nobody owns that handle. Claim it, or use one `registry_handles` lists. |
| `HANDLE_NOT_YOURS` | Another organization owns it. Pick one you own. Do not retry. |
| `PACKAGE_NOT_YOURS` | Another organization published this package; only they can version it. |
| `TOO_MANY_FILES` | Over 500 files. Narrow `include`, or split into several packages. |
| `TOO_LARGE` | Bodies total over 8 MiB. Narrow `include`, or split. |
| `denied: publishing requires ownership` | Gate 1. The user needs an organization owner to do this; you cannot work around it. |
| Refusal naming `acknowledge_public` | Gate 3. Go and ask the user. Do not re-send with the flag on your own judgement. |

A refusal about ownership or a handle will produce the identical result on retry. Report it and ask for what is missing.

## When the registry tools are absent

The registry tools ship **off by default** — they are a separate subsystem, and six extra names on every agent's surface is a cost every organization pays for the few that publish. If `registry_publish` is not in your toolset, the deployment has not enabled it (`REGISTRY_TOOLS=on`); that is configuration, not a fault, and not something to debug. Tell the user which tools you have and that publishing needs enabling by whoever runs the deployment.

## Gotchas

- **"Publish" in casual speech usually means "share".** Check before you snapshot anything out of the organization.
- **Omitting `visibility` never changes the audience — it does not avoid the ack.** A new package defaults to `private`; an existing one keeps what it has, and if that is public or unlisted the publish is refused until acknowledged. Guessing `public` is what is never safe.
- **A version is immutable.** Republishing `1.0.0` after a mistake is a no-op, not a correction. Go to `1.0.1`.
- **There is no unpublish, and hiding is not undoing.** No tool deletes a package or a version. Republishing at `private` does hide the whole package from the catalog — including versions that were public — but it is not a recall: the bodies were already served to anyone who asked, and hiding them cannot retrieve those copies. Treat a mistaken public publish as disclosed, tell the user so, and rotate anything secret that was in it.
- **The catalog is global; installs are per organization.** Seeing a package does not mean anyone has installed it, and installing does not tell the publisher who you are beyond the row they can see.
- **Labels are not visibility.** A file labelled `public` is not public, and packaging is not what a label does. See the `authorization-model` skill.
