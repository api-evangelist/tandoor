---
name: tandoor-safe-cleanup-foods-and-keywords
description: Deduplicate and tidy a Tandoor space's foods, units, keywords and supermarket categories without destroying recipes — using Tandoor's delete-impact preview endpoints and merge instead of delete.
api: Tandoor API
generated: '2026-08-27'
method: generated
source: openapi/tandoor-api-openapi.yml
operations:
  - apiFoodList
  - apiFoodCascadingList
  - apiFoodNullingList
  - apiFoodProtectingList
  - apiFoodMergeUpdate
  - apiFoodMoveUpdate
  - apiFoodDestroy
  - apiKeywordList
  - apiKeywordMergeUpdate
  - apiUnitMergeUpdate
  - apiSupermarketCategoryMergeUpdate
  - apiExportCreate
---

# Clean up foods and keywords without losing recipes

Tandoor has **no undo, no soft delete and no trash**. A `DELETE` is permanent. What it does have — and
this is unusual enough to build the whole flow around — is a documented pre-flight that tells you
exactly what a delete would destroy *before* you issue it.

## The rule

**Never call a `Destroy` operation without first calling all three preview endpoints for that object.**

- `GET /api/{resource}/{id}/cascading/` — objects that will be **deleted with it**
- `GET /api/{resource}/{id}/nulling/` — objects that survive but **lose their reference** to it
- `GET /api/{resource}/{id}/protecting/` — objects that will make the **delete fail**

Each returns a paginated list of `GenericModelReference` `{id, model, name}`. Available on `recipe`,
`food`, `unit`, `keyword`, `recipe-book`, `shopping-list`, `supermarket` and `supermarket-category`.

## Steps

1. **Back up first.** `POST /api/export/` (`apiExportCreate`) exports the space. This is the only
   recovery path that exists — see `https://docs.tandoor.dev/system/backup/`. Do this before any
   bulk cleanup, and tell the user you did.

2. **Find the duplicates.** `GET /api/food/?query=<term>` (`apiFoodList`) or
   `GET /api/keyword/?query=<term>` (`apiKeywordList`). Both are **trees**, so a "duplicate" is often
   really a child that belongs under a parent.

3. **Prefer merge to delete.** `PUT /api/{resource}/{id}/merge/{target}/` repoints every reference
   from the source onto the target and then removes the source — no recipe loses an ingredient.
   Available as `apiFoodMergeUpdate`, `apiUnitMergeUpdate`, `apiKeywordMergeUpdate` and
   `apiSupermarketCategoryMergeUpdate`.
   **The merge itself cannot be undone.** There is no unmerge. Confirm the direction with the user —
   `{id}` is the one that disappears, `{target}` is the one that survives.

4. **Or re-parent instead.** `PUT /api/food/{id}/move/{parent}/` (`apiFoodMoveUpdate`) and the
   keyword equivalent move a node under a different parent without destroying anything. This is the
   safest operation in the whole cleanup surface.

5. **Only then delete.** If the three previews come back empty and the user has confirmed, call
   `DELETE /api/food/{id}/` (`apiFoodDestroy`). If `protecting/` is non-empty the delete will be
   refused by the server anyway — report the protecting objects to the user rather than trying to
   force it.

## Reporting back

Summarise in the user's terms, not the API's: "merging *tomato* into *tomatoes* will update 14
recipes and 2 shopping list entries" reads better than a list of ids, and it is exactly what
`cascading/` and `nulling/` just told you.

## Errors

No error responses are documented. Expect `{"detail": "..."}` on permission failures — most cleanup
operations require more than read-only access to the space, and a shared read-only user will be
refused.
