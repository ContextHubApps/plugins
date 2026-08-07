---
name: authorization-model
description: How ContextHub decides what a principal may read. Use when reasoning about ContextHub access, grants, roles, cascade, or groups; when a search or listing returns less than expected and you need to know whether content is missing or gated; when explaining why an agent cannot see a document; when asked who can see something; or before claiming anything about permissions, visibility, or what exists in the store. Also use to correct the two dead designs (tag-based ABAC, Cerbos on file access).
---

# ContextHub authorization

Retrieval is authorization-filtered. An agent sees exactly what its principal is granted, and the filter is applied inside the query rather than bolted on afterward.

## The decision chain

```
credential (cht_ token | Clerk OAuth access token)
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
| role | `owner`, `writer`, `proposer`, `reader`, `member` |
| scope | `org`, `folder`, or `document` |

Postgres `reach_grants` is the source of truth. OpenFGA tuples are a derived projection, so a grant survives an OpenFGA outage and can be reconciled afterward.

**Nothing is readable by default.** Ungranted is ungranted. There is no ambient read, no public tier, no fallback that opens content up.

### Cascade and nesting

- Grants **cascade down the folder tree**. A grant on a folder reaches every descendant, including files created later.
- A grant is **granted-here** (made directly on this scope) or **inherited** (arriving from an ancestor). The console marks which.
- **Groups nest.** A grant to a group reaches its members, and a group can contain groups, so a principal may reach a file through several hops. Resolution is live, not snapshotted.

### Roles

`reader` / `writer` / `owner` are what the console offers outward as view / edit / manage. `writer` implies `reader`.

`owner` is deliberately **not** a rung of the relation ladder. The OpenFGA model states `writer ⟹ reader` but `owner ⇏ writer/reader`, because ownership is a fact about a thing rather than a level of access to it.

The **authority layer folds it anyway.** `canDo` in `decide.ts` checks the verb, and if that fails, checks `owner` and allows on a hit. So an owner satisfies the writer bar in practice. This fold lives in the decision function precisely so both doors agree: a folder's creator is granted `owner` plus `reader` and never `writer`, and before the fold moved there, that creator could delete the folder through the console yet be refused writing a file into it over MCP. Same principal, two doors, two answers. Distinguish the relation (`owner ⇏ writer`) from the authority (an owner may write).

`proposer` folds to reader for read purposes; it suits a contributor who stages changes rather than committing them. `member` is an internal role.

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

## Cerbos governs console RBAC only

Cerbos (`cerbos/policies/console.yaml`) decides the `admin` / `member` console role on a principal, answering "may you use this console tool". It **never** decides which files you can read. Console role and content access are separate systems; a console admin has no file access they were not granted.

## Labels carry ZERO authority

Labels (tags) are organization and discovery metadata. No authorization decision reads the labels table.

- "Unlabeled" says nothing about access.
- A sensitive-sounding label restricts nothing.
- Filtering by label narrows what you already reach; it never widens it.

Labels inherit from a folder or directory for discovery purposes, so a folder's label surfaces everything inside it in a label-filtered listing. That is a search convenience, not an access rule.

## Common misconceptions

**Wrong: tag-based ABAC, where untagged content is publicly readable.**
This design is REMOVED and repeating it is a correctness failure. There was a version where security tags formed an authorization axis and untagged content was readable by anyone in the tenant. Since the security-tag axis was removed, reach grants plus group membership are the ONLY axis. Untagged content is not public. It is ungranted, therefore invisible.

**Wrong: Cerbos decides file access.**
An older design put Cerbos on file access. That is gone. Cerbos is console RBAC only. Content decisions are OpenFGA ReBAC only. If you find yourself explaining a read denial in terms of a Cerbos policy, you have the wrong model.

**Wrong: an empty result proves nothing exists.**
See above. This is the failure this system is specifically built to prevent you from making.

**Wrong: there is a tool to list grants or read the audit trail.**
There is not, by design. No grants tool, and `audit_tail` was deliberately removed so a token holder cannot audit a whole tenant. Audit is a console surface, admin-gated. For cross-principal questions, deep-link the console.

**Half-wrong: "owner implies nothing" / "owner implies everything".**
Both miss. The *relation* does not ladder (`owner ⇏ writer/reader` in the model), but the *authority layer* folds owner into the writer bar in `canDo`, so an owner may write. See Roles.

## Consequences for how you work

- Never assert a document does not exist. Say you could not reach one.
- Never infer access from a name, path, or label.
- Never name who can see something unless a tool returned it. Route cross-principal questions to the console: the **Share panel**, the **"Shared with"** list (granted-here versus inherited), the **reach lens / "View as"** on the Files tree, the **share summary** panel, **Groups**, and **Access → Roles**.
- Never call a console API endpoint from an agent context. That API takes a Clerk browser session JWT only; an OAuth or `cht_` credential gets 401 every time.
- A write landing is not a write going live. `how="propose"` stages for review, and a commit may land in quarantine for operator promotion in Triage. "It wrote" and "it is live" are different claims.

## Source of truth

| What | Where |
|---|---|
| Decision logic (`canDo`, `readableSet`) | `packages/core/src/authz/decide.ts` |
| Allow-set to SQL, fail-closed | `packages/core/src/authz/read-filter.ts` |
| Grant rows | `reach_grants` in `packages/core/prisma/schema.prisma` |
| Existence-oracle parity tests | `apps/mcp-server/tests/exposure-parity.test.ts` |
| Console RBAC policy | `cerbos/policies/console.yaml` |
| Expected behavior matrix | `docs/permissioning-test-matrix.md` |
| Engine choice and its addendum | `docs/adr/0001-authorization-engine-cerbos-now-openfga-when-hierarchical.md` |
| Why the model is folders all the way down | `docs/adr/0003-drop-the-bundle-abstraction-folders-all-the-way-down.md` |
