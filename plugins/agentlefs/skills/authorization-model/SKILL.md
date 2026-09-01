---
name: authorization-model
description: How agentleFS decides what a principal may read. Use when reasoning about agentleFS access, grants, roles, cascade, or groups; when a search or listing returns less than expected and you need to know whether content is missing or gated; when explaining why an agent cannot see a document; when asked who can see something; or before claiming anything about permissions, visibility, or what exists in the store. Also use to correct the three dead designs (tag-based ABAC, Cerbos anywhere, and owner-outside-the-ladder).
---

# agentleFS authorization

Retrieval is authorization-filtered. An agent sees exactly what its principal is granted, and the filter is applied inside the query rather than bolted on afterward.

## The decision chain

```
credential (afs_ token | Clerk OAuth access token)
  → principal (tenant-bound)
    → reach_grants rows  (subject × role × scope)
      → projected into OpenFGA tuples
        → decide.ts: canDo (point check) | readableSet (set query)
          → read-filter.ts compiles the set to a SQL WHERE fragment
            → embedded in every listing and every retrieval
```

The principal's identity is the entire authorization input. Nothing in the request adds authority: not the folder name, not the label, not the query, not a flag. Same principal, same answer.

**Content access is decided ONLY by OpenFGA ReBAC.** One axis, no others.

## Grants

A row in `reach_grants` says "subject holds `role` on scope".

| Field | Values |
|---|---|
| subject | `user` (a principal) or `group` |
| role | `owner`, `writer`, `reader` |
| scope | `tenant`, `folder`, or `document` |

**Breadth comes from WHERE a grant sits, not from a second vocabulary.** A `tenant`-scope owner is what used to be called a console admin and reaches the whole workspace; a folder-scope owner reaches that subtree. There is no `admin` role.

Postgres `reach_grants` is the source of truth. OpenFGA tuples are a derived projection, so a grant survives an OpenFGA outage and can be reconciled afterward.

**Nothing is readable by default.** Ungranted is ungranted. There is no ambient read, no public tier, no fallback that opens content up.

### Cascade and nesting

- Grants **cascade down the folder tree**. A grant on a folder reaches every descendant, including files created later.
- A grant is **granted-here** (made directly on this scope) or **inherited** (arriving from an ancestor). The console marks which.
- **Groups nest.** A grant to a group reaches its members, and a group can contain groups, so a principal may reach a file through several hops. Resolution is live, not snapshotted.

### Roles

**One ladder: `reader` < `writer` < `owner`.** Each rung contains the one below it, so an owner reads and writes everything it owns. The console offers them outward as view / edit / manage.

The containment is stated once, in `openfga/model.fga`, and nowhere else. There is no app-layer fold: asking the engine for `writer` already returns allow for an owner. Two places stating the same rule is how they come to disagree.

`owner` additionally carries what the ladder cannot express — granting, revoking, and reading the audit log — bounded to the subtree the grant sits on.

`proposer` and `member` are RETIRED as roles. Neither was ever granted anywhere, and both rendered as reader.

## Fail-closed

The read path refuses to under-report. If the allow-set is truncated, `read-filter.ts` throws `ReachTruncatedError` rather than filtering on a subset and returning a partial answer that looks complete. An error here is the system declining to silently show you less than your grants allow. Surface it; never paper over it with a partial summary.

## Denied is byte-identical to not-found

**This is the most important idea in the system.** There is no existence oracle.

- A path you cannot read returns not-found, worded identically to a path that does not exist.
- A folder you do not reach is simply absent from `list_org_folders`. No count, no marker, no placeholder.
- An empty search says it cannot settle the question, not that nothing exists.
- The empty-listing text is deliberately unchanged between "nothing here" and "nothing for you".

`apps/mcp-server/tests/exposure-parity.test.ts` asserts this, including that a denied *directory* which genuinely exists is indistinguishable from an absent one.

### Reading a thin result

| What you observe | What it means | What it does NOT mean |
|---|---|---|
| Empty folder list | This principal reaches no folders | The org has no content |
| `not found: path` | Gated from you, or absent. Cannot distinguish. | The file does not exist |
| Search returns nothing | Nothing matched **in what you can read** | The org has written nothing on it |
| `denied: 12` in folder shape | 12 files are gated from you | Anything about their names, paths, or subject matter |

The one place a count leaks through is `list_org_folders` on a folder, which reports readable versus `denied`. That is a count only. It never identifies a gated file.

Never report a thin result as evidence of absence. Say "nothing I can reach matches" and note that gated content is invisible from here.

## One decider, for the console as well as for content

There is no second engine. Console actions resolve through the same OpenFGA ladder: anything above a small read-only floor requires ownership of the **tenant root**, which is the node every folder hangs off. So "may you use this console tool" and "may you read this file" are the same kind of question asked about different objects.

A tenant-root owner therefore does reach every file in the workspace — deliberately, because that is what being an owner of the root means. This is a change: under the old model a console admin had no file access they were not separately granted.

## Labels carry ZERO authority

Labels (tags) are organization and discovery metadata. No authorization decision reads the labels table.

- "Unlabeled" says nothing about access.
- A sensitive-sounding label restricts nothing.
- Filtering by label narrows what you already reach; it never widens it.

Labels inherit from a folder or directory for discovery purposes, so a folder's label surfaces everything inside it in a label-filtered listing. That is a search convenience, not an access rule.

## Common misconceptions

**Wrong: tag-based ABAC, where untagged content is publicly readable.**
This design is REMOVED and repeating it is a correctness failure. There was a version where security tags formed an authorization axis and untagged content was readable by anyone in the tenant. Since the security-tag axis was removed, reach grants plus group membership are the ONLY axis. Untagged content is not public. It is ungranted, therefore invisible.

**Wrong: Cerbos decides anything.**
Cerbos is GONE — the policy files, the sidecar, the service. It decided console RBAC until #219, and an older design put it on file access before that. Every decision is OpenFGA ReBAC now. If you find yourself explaining any denial in terms of a Cerbos policy, you have the wrong model.

**Wrong: an empty result proves nothing exists.**
See above. This is the failure this system is specifically built to prevent you from making.

**Wrong: there is a tool to list grants or read the audit trail.**
There is not, by design. No grants tool, and `audit_tail` was deliberately removed so a token holder cannot audit a whole tenant. Audit is a console surface, admin-gated. For cross-principal questions, deep-link the console.

**Wrong: "owner grants access but cannot itself read".**
That was true, and it was the defect #219 fixed. Ownership is the top of one ladder: an owner reads and writes everything beneath it. Any explanation that treats ownership as an orthogonal badge rather than the highest rung is describing the old model.

## Consequences for how you work

- Never assert a document does not exist. Say you could not reach one.
- Never infer access from a name, path, or label.
- Never name who can see something unless a tool returned it. Route cross-principal questions to the console: the **Share panel**, the **"Shared with"** list (granted-here versus inherited), the **reach lens / "View as"** on the Files tree, the **share summary** panel, **Groups**, and **Access → Roles**.
- Never call a console API endpoint from an agent context. That API takes a Clerk browser session JWT only; an OAuth or `afs_` credential gets 401 every time.
- A write is authorized or it is refused, and there is no third state. This bullet used to describe `how="propose"` staging for review and commits landing in quarantine for operator promotion in Triage; propose, quarantine and Triage are all gone. A successful `write_org_doc` is live immediately. Who can then READ it is decided by the grants on the path, not by anything the writer sets.

## Source of truth

| What | Where |
|---|---|
| Decision logic (`canDo`, `readableSet`) | `packages/core/src/authz/decide.ts` |
| Allow-set to SQL, fail-closed | `packages/core/src/authz/read-filter.ts` |
| Grant rows | `reach_grants` in `packages/core/prisma/schema.prisma` |
| The ladder itself | `openfga/model.fga` |
| Existence-oracle parity tests | `apps/mcp-server/tests/exposure-parity.test.ts` |
| The console ladder and the tenant root | `packages/core/src/authz/console-ladder.ts` |
| Expected behavior matrix | `docs/permissioning-test-matrix.md` |
| Engine choice and its addendum | `docs/adr/0001-authorization-engine-cerbos-now-openfga-when-hierarchical.md` |
| Why the model is folders all the way down | `docs/adr/0003-drop-the-bundle-abstraction-folders-all-the-way-down.md` |
