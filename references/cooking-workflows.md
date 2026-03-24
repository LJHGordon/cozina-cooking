# Cooking Workflows Reference

This document describes the Cozina workflows that are currently available through `cozina-mcp`.

## Current MCP Support

| Tool | Purpose |
|------|---------|
| `cozina_save_recipe` | Save a structured recipe to the user's account. |
| `cozina_list_recipes` | Search and browse recipes with filters. |
| `cozina_get_recipe` | Retrieve full recipe detail. |
| `cozina_list_cookbooks` | List the user's cookbooks with recipe counts. |
| `cozina_create_cookbook` | Create a normal personal cookbook, or return the existing one if the name already exists. |
| `cozina_rename_cookbook` | Rename a normal personal cookbook. |
| `cozina_delete_cookbook` | Delete a normal personal cookbook without deleting its recipes. |
| `cozina_get_usage` | Check subscription tier and usage. |
| `cozina_get_meal_plan` | Read the user's next 7 days of planned meals. |
| `cozina_save_meal_plan` | Save meal-plan entries for a date range. |
| `cozina_clear_meal_plan` | Clear meal-plan entries for a date range. |
| `cozina_get_shopping_list` | Read the active shopping list. |
| `cozina_generate_shopping_list` | Generate or update the active shopping list from recipes or meal-plan entries. |
| `cozina_set_shopping_list_item_checked` | Check or uncheck a shopping-list item. |

## 1. Weekly Meal Planning

### What the assistant should do

When the user says "plan my meals for the week" or similar:

1. Call `cozina_list_recipes` and `cozina_list_cookbooks` to understand the user's collection.
2. Ask for constraints if needed: servings, dietary preferences, time budget, cuisine variety, leftovers, or target start date.
3. Draft a 7-day plan beginning today or the requested `start_date`.
4. Present the plan for confirmation before saving.
5. Save the confirmed entries with `cozina_save_meal_plan`.

### Data model

- Meal plans are stored as dated recipe assignments in `meal_plan_items`.
- Valid `meal_type` values are `breakfast`, `lunch`, `dinner`, and `snack`.
- v1 uses a rolling next-7-days range by default.

### Related tools

- `cozina_get_meal_plan`
- `cozina_save_meal_plan`
- `cozina_clear_meal_plan`

## 2. Shopping Lists

### What the assistant should do

When the user asks for a shopping list:

1. Decide the source:
   - explicit `recipe_ids`
   - or the saved meal-plan range for the next 7 days
2. Call `cozina_generate_shopping_list`.
3. Call `cozina_get_shopping_list` to present the current active list.
4. If the user asks to mark items complete, call `cozina_set_shopping_list_item_checked`.

### Data model

- v1 uses a single active shopping list.
- The default list name is `My Shopping List`.
- Ingredients are deduplicated by lowercase `name + unit`.
- Duplicate quantities are summed.
- Categories are assigned using Cozina's aisle-mapper logic.

### Related tools

- `cozina_get_shopping_list`
- `cozina_generate_shopping_list`
- `cozina_set_shopping_list_item_checked`

## 3. Cookbook Management

### What the assistant should do

When the user asks to create, rename, or delete a cookbook:

1. Call `cozina_list_cookbooks` first.
2. Create missing normal personal cookbooks with `cozina_create_cookbook`.
3. Rename normal personal cookbooks with `cozina_rename_cookbook`.
4. Delete normal personal cookbooks with `cozina_delete_cookbook`.
5. Do not rename or delete protected cookbooks:
   - `Favorites`
   - `Liked Recipes`
   - `Breakfast`
   - `Lunch`
   - `Dinner`

### Data model

- v1 cookbook creation is name-only.
- Duplicate cookbook creates return the existing cookbook.
- Deleting a cookbook removes the cookbook and its cookbook memberships, but leaves the recipes in All Recipes.
- Family/shared cookbook management is not exposed through MCP in this version.

### Related tools

- `cozina_list_cookbooks`
- `cozina_create_cookbook`
- `cozina_rename_cookbook`
- `cozina_delete_cookbook`

## 4. Cookbook-Aware Recommendations

Recipe suggestions still begin with the user's existing collection.

1. Call `cozina_list_recipes` to understand what they already have.
2. Use `cozina_list_cookbooks` when cookbook organization matters.
3. Suggest recipes before creating new imports.
4. If the user wants the week saved, convert the accepted picks into `cozina_save_meal_plan` entries.

## 5. Recipe Import And Save

Recipe extraction behavior is unchanged:

1. Extract the recipe from URL, text, image, or conversation.
2. Present the structured result for confirmation.
3. If the user named a cookbook, call `cozina_list_cookbooks` to resolve it first.
4. If the cookbook does not exist yet, ask for confirmation and create it with `cozina_create_cookbook`.
5. Save with `cozina_save_recipe`.
6. Offer to add the new recipe to the user's upcoming meal plan or shopping list.

## 6. Usage Guidance

When the user asks about their account or credits:

1. Call `cozina_get_usage`.
2. Report tier, monthly usage, remaining credits, and top-up balance.
3. Remind the user that recipe saving is free; AI-powered generation is what consumes credits.
