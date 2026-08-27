---
name: tandoor-import-recipe-from-url
description: Import a recipe into Tandoor from a web page URL (or pasted page source), review what was scraped, and save it — the single most-used write flow in Tandoor.
api: Tandoor API
generated: '2026-08-27'
method: generated
source: openapi/tandoor-api-openapi.yml
operations:
  - apiRecipeFromSourceCreate
  - apiRecipeImportList
  - apiRecipeImportImportRecipeCreate
  - apiRecipeCreate
  - apiRecipeRetrieve
  - apiKeywordList
---

# Import a recipe from a URL

## Before you start

- **Base URL.** Tandoor is self-hosted. The base is `https://<instance>/api/`. The vendor's hosted
  instance is `https://app.tandoor.dev/api/`. Confirm the instance's capabilities by fetching
  `https://<instance>/openapi/` — it needs no credentials.
- **Auth.** Send `Authorization: <access token>` on every call. Do **not** follow redirects: an
  unauthenticated `/api/` request is 302'd to `/accounts/login/` and a redirect-following client will
  read an HTML login page as a 200.
- **Throttle.** URL import is rate limited, default `60/hour` per user
  (`DRF_THROTTLE_RECIPE_URL_IMPORT`). No rate-limit header is returned; budget for it yourself.
- **Not idempotent.** There is no `Idempotency-Key`. If a call times out, do **not** blindly retry —
  search `apiRecipeList` for the recipe name first, or you will create a duplicate.

## Steps

1. **Scrape the source.**
   `POST /api/recipe-from-source/` (`apiRecipeFromSourceCreate`) with `{"url": "<page url>"}`.
   If the site blocks the server, fetch the page yourself and send `{"data": "<html or text>"}`
   instead. The response is a `RecipeFromSourceResponse` — a parsed recipe plus `duplicates`, the
   recipes already in the space that look like this one.

2. **Check `duplicates` before writing anything.** If it is non-empty, stop and confirm with the user.
   This is the only duplicate protection the API offers.

3. **Reconcile keywords.** The scrape returns keyword names as strings. Call
   `GET /api/keyword/?query=<name>` (`apiKeywordList`) for each and reuse the existing id where one
   matches. Creating near-duplicate keywords is the most common way an agent makes a space messy, and
   keywords are a tree — merging them back later costs a `PUT /api/keyword/{id}/merge/{target}/`.

4. **Create the recipe.**
   `POST /api/recipe/` (`apiRecipeCreate`) with the reconciled body: `name`, `description`,
   `servings`, `working_time`, `waiting_time`, `source_url`, `keywords[]`, and `steps[]` where each
   step carries `instruction` and `ingredients[]` of `{amount, unit, food, note}`.
   Foods and units may be sent as `{"name": "..."}`; Tandoor creates them if they do not exist.

5. **Verify.** `GET /api/recipe/{id}/` (`apiRecipeRetrieve`) and check that every step and ingredient
   round-tripped. Report the recipe id and its URL back to the user.

## The bulk variant

For a batch of URLs from an import job, use the import-list flow instead:
`GET /api/recipe-import/` (`apiRecipeImportList`) lists staged imports;
`POST /api/recipe-import/{id}/import_recipe/` (`apiRecipeImportImportRecipeCreate`) commits one; and
`POST /api/recipe-import/import_all/` (`apiRecipeImportImportAllCreate`) commits every staged item.
`import_all` is a bulk write with no dry run and no undo — ask before calling it.

## Errors

The contract documents **no** error responses. Expect the django-rest-framework shapes:
`{"field": ["message"]}` for validation failures and `{"detail": "message"}` for auth and permission
failures. A `400` on step 1 usually means the page carried no `schema.org/Recipe` markup — fall back
to asking the user to paste the recipe text and send it as `data`.
