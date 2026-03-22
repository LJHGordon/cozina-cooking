# Shopping List From Meal Plan Example

User: "Make a shopping list for this week's meals."

Assistant workflow:

1. Call `cozina_get_meal_plan` for the relevant 7-day range.
2. Confirm the date range if needed.
3. Call `cozina_generate_shopping_list` using the meal-plan range.
4. Call `cozina_get_shopping_list` to present the active list.
5. If the user wants changes, check or uncheck items with `cozina_set_shopping_list_item_checked`.

Example generate payload:

```json
{
  "start_date": "2026-03-23",
  "days": 7,
  "mode": "merge"
}
```
