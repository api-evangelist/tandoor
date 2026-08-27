---
name: tandoor-plan-meals-and-build-shopping-list
description: Schedule recipes onto a Tandoor meal plan for a date range, push their ingredients onto the shopping list, and keep the list tidy — the flow that turns a recipe collection into a week of groceries.
api: Tandoor API
generated: '2026-08-27'
method: generated
source: openapi/tandoor-api-openapi.yml
operations:
  - apiMealTypeList
  - apiMealPlanList
  - apiMealPlanCreate
  - apiAutoPlanCreate
  - apiRecipeShoppingUpdate
  - apiShoppingListEntryList
  - apiShoppingListEntryBulkCreate
  - apiShoppingListEntryPartialUpdate
  - apiMealPlanIcalRetrieve
---

# Plan meals and build the shopping list

## Before you start

- Auth and base URL as in `tandoor-import-recipe-from-url`. Do not follow redirects.
- Everything here is scoped to the **active space**. If the user has several,
  `GET /api/switch-active-space/{spaceId}/` (`apiSwitchActiveSpaceRetrieve`) first.
- All of these are writes and **none is idempotent**. Read before you write.

## Steps

1. **Read the meal types.** `GET /api/meal-type/` (`apiMealTypeList`). These are per-user and
   user-ordered (Breakfast/Lunch/Dinner or whatever the user defined) — never invent one.

2. **Read what is already planned.** `GET /api/meal-plan/` (`apiMealPlanList`) with the date filters
   for the window you are planning. Overwriting an existing plan entry the user made by hand is the
   most annoying thing an agent can do here.

3. **Schedule each meal.** `POST /api/meal-plan/` (`apiMealPlanCreate`) with `recipe` (an id),
   `meal_type`, `from_date`, `to_date` and `servings`. A plan entry may also be a free-text `title`
   with no recipe — use that for "leftovers" or "eating out".

   *Or* let Tandoor pick: `POST /api/auto-plan/` (`apiAutoPlanCreate`) fills a window from keywords
   and recipe books. It is a bulk write — confirm the parameters with the user before calling it.

4. **Push ingredients onto the shopping list.**
   `PUT /api/recipe/{id}/shopping/` (`apiRecipeShoppingUpdate`) adds one recipe's ingredients, scaled
   to the servings you pass. Tandoor merges the entry into an existing one for the same food and unit
   rather than duplicating it.

   For ad-hoc items, `POST /api/shopping-list-entry/bulk/` (`apiShoppingListEntryBulkCreate`) creates
   several entries in one call.

5. **Read the list back sorted for the shop.** `GET /api/shopping-list-entry/`
   (`apiShoppingListEntryList`). Each entry carries `food`, `unit`, `amount`, `checked`, and
   `list_recipe_data` naming the recipe or meal plan it came from. Group by the food's
   `supermarket_category` and order the categories with
   `GET /api/supermarket-category-relation/` (`apiSupermarketCategoryRelationList`) to match the
   user's supermarket aisle layout.

6. **Tick items off.** `PATCH /api/shopping-list-entry/{id}/`
   (`apiShoppingListEntryPartialUpdate`) with `{"checked": true}`. **Prefer this to deleting.**
   Checking is reversible — set it back to `false`. `DELETE` is not: there is no undo and no trash.

7. **Export the plan if asked.** `GET /api/meal-plan/ical/` (`apiMealPlanIcalRetrieve`) returns
   `text/calendar` (RFC 5545) for any calendar client. It is the only non-JSON response in the API.

## Pagination

Every list here is paginated with `page` / `page_size` and returns
`{count, next, previous, results}`. A shopping list for a week routinely exceeds one page — follow
`next` until it is null before telling the user what is on the list.
