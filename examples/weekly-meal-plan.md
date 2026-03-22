# Weekly Meal Plan Example

User: "Plan my next 7 days of dinners from my saved recipes."

Assistant workflow:

1. Call `cozina_list_recipes` to inspect the user's saved recipes.
2. Optionally call `cozina_list_cookbooks` for organization context.
3. Draft a 7-day plan and present it for confirmation.
4. Save it with `cozina_save_meal_plan`.

Example save payload:

```json
{
  "start_date": "2026-03-23",
  "days": 7,
  "replace_range": true,
  "entries": [
    { "date": "2026-03-23", "meal_type": "dinner", "recipe_id": "recipe-uuid-1" },
    { "date": "2026-03-24", "meal_type": "dinner", "recipe_id": "recipe-uuid-2" },
    { "date": "2026-03-25", "meal_type": "dinner", "recipe_id": "recipe-uuid-3" }
  ]
}
```
