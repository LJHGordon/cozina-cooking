# Cozina Recipe Schema Reference

This document provides the complete field-by-field specification of the JSON object accepted by the `cozina_save_recipe` MCP tool. The schema mirrors the Cozina app's internal `Recipe`, `RecipeSection`, `Ingredient`, and `RecipeStep` data models.

## Top-Level Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `title` | `string` | **Yes** | -- | Recipe name. Keep concise: "Spaghetti Carbonara", not "The Best Ever Spaghetti Carbonara You Will Ever Make". |
| `sections` | `Section[]` | **Yes** | -- | At least one section. Each section groups related ingredients and steps (e.g., "Dough", "Filling", "Glaze"). |
| `summary` | `string` | No | `null` | Brief description (1-2 sentences). Displayed below the title in the app. |
| `servings` | `number` | No | `4` | Integer, number of servings the recipe yields. |
| `prep_time` | `string` | No | `null` | Human-readable prep time string. Examples: `"15 min"`, `"1 hour"`, `"30 minutes"`. |
| `cook_time` | `string` | No | `null` | Human-readable cook time string. Same format as `prep_time`. |
| `cuisine` | `string` | No | `null` | Cuisine classification. Examples: `"Italian"`, `"Japanese"`, `"Mexican"`, `"Indian"`, `"American"`. |
| `cooking_method` | `string` | No | `null` | Primary cooking method. Examples: `"Baking"`, `"Grilling"`, `"Sauteing"`, `"Slow Cooking"`, `"No Cook"`. |
| `source_url` | `string` | No | `null` | Original URL the recipe was extracted from. Stored for attribution and re-fetching. |
| `image_url` | `string` | No | `null` | URL to a hero image. Must be publicly accessible (no auth-gated URLs). The app downloads and caches images locally. |
| `nutrition` | `Nutrition` | No | `null` | Basic macronutrient data per serving. See Nutrition Object below. |
| `keywords` | `string[]` | No | `[]` | Searchable tags. Examples: `["quick", "one-pot", "gluten-free", "date-night"]`. |
| `cookbook_id` | `string (UUID)` | No | `null` | Target cookbook UUID. Retrieve valid IDs from `cozina_list_cookbooks`. If omitted, recipe is saved to the user's default (uncategorized) collection. |


## Section Object

Each section represents a logical grouping within a recipe. Simple recipes have one section named `"Default"`. Complex recipes (layered cakes, multi-component dishes) have multiple sections.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | `string` | **Yes** | -- | Section name. Use `"Default"` for single-section recipes. Other examples: `"Pasta"`, `"Sauce"`, `"Frosting"`, `"Marinade"`. |
| `ingredients` | `Ingredient[]` | **Yes** | -- | Array of ingredients for this section. May be empty if section is instructions-only (rare). |
| `steps` | `Step[]` | **Yes** | -- | Array of cooking steps for this section. Must contain at least one step. |

### Design Rule

Do not flatten multi-section recipes into a single section. The Cozina app's cooking mode, shopping list, and timer features all operate at the section level. A "Cake" recipe with "Batter" and "Frosting" sections enables the user to work on both components with independent progress tracking.


## Ingredient Object

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `item` | `string` | **Yes** | -- | Ingredient name. Lowercase, no quantity or unit embedded. Examples: `"all-purpose flour"`, `"garlic cloves"`, `"unsalted butter"`. |
| `quantity` | `number` | No | `0` | Numeric quantity. Use `0` for "to taste" or "as needed" items. Fractions must be converted to decimals: 1/2 = `0.5`, 1/3 = `0.333`, 3/4 = `0.75`. |
| `unit` | `string` | No | `""` | Unit of measurement. Free-form string. Common values: `"cup"`, `"cups"`, `"tbsp"`, `"tsp"`, `"oz"`, `"lb"`, `"g"`, `"kg"`, `"ml"`, `"L"`, `"pinch"`, `"clove"`, `"can"`, `"piece"`, `"slice"`, `"bunch"`, `"head"`, `"stalk"`. Keep singular or plural consistent within a recipe. |
| `notes` | `string` | No | `null` | Preparation notes. Examples: `"finely diced"`, `"room temperature"`, `"divided"`, `"or substitute almond milk"`. |

### Ingredient Parsing Rules

- Separate the quantity, unit, and item cleanly. `"2 cups all-purpose flour"` becomes `{ quantity: 2, unit: "cups", item: "all-purpose flour" }`.
- Parenthetical notes go into `notes`: `"2 eggs (large, room temp)"` becomes `{ quantity: 2, unit: "", item: "eggs", notes: "large, room temp" }`.
- Range quantities: use the midpoint. `"2-3 cloves garlic"` becomes `{ quantity: 2.5, unit: "cloves", item: "garlic" }`.
- "To taste" items: `{ quantity: 0, unit: "", item: "salt", notes: "to taste" }`.


## Step Object

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `instruction` | `string` | **Yes** | -- | Full instruction text. One logical action per step. Keep imperative voice: "Preheat oven to 375F", not "The oven should be preheated to 375F". |
| `duration` | `number` | No | `0` | Duration in **seconds**. This powers the app's cooking timers. Convert all time references: 5 minutes = `300`, 1 hour = `3600`, 45 seconds = `45`. Use `0` for instant actions (add salt, stir once). |
| `rank` | `number` | No | auto | Sort order within the section. 0-indexed. If omitted, the API assigns ranks in array order. Provide explicit ranks only when reordering is necessary. |

### Duration Extraction Rules

- Parse time references from the instruction text. "Simmer for 20 minutes" = `duration: 1200`.
- "Until golden brown" with no time reference = `duration: 0` (the app will not start a timer for this step).
- Passive wait times still get durations: "Let dough rise for 1 hour" = `duration: 3600`.
- Multiple time references in one step: use the longest. "Saute for 3-5 minutes" = `duration: 300` (use the upper bound).


## Nutrition Object

Per-serving macronutrient data. All fields are optional integers.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `calories` | `number` | No | `null` | Kilocalories per serving. |
| `protein` | `number` | No | `null` | Grams of protein per serving. |
| `carbs` | `number` | No | `null` | Grams of carbohydrates per serving. |
| `fat` | `number` | No | `null` | Grams of fat per serving. |

Omit the entire `nutrition` object if no data is available. Do not guess nutritional values -- only include them when the source explicitly provides them (JSON-LD `nutrition` field, recipe page sidebar, etc.).


## Complete Example

A full recipe JSON ready to pass to `cozina_save_recipe`:

```json
{
  "title": "Spaghetti Carbonara",
  "summary": "Classic Roman pasta with eggs, pecorino, guanciale, and black pepper. No cream.",
  "servings": 4,
  "prep_time": "10 min",
  "cook_time": "20 min",
  "cuisine": "Italian",
  "cooking_method": "Sauteing",
  "source_url": "https://example.com/carbonara",
  "image_url": "https://example.com/images/carbonara.jpg",
  "keywords": ["pasta", "quick", "Italian", "classic"],
  "nutrition": {
    "calories": 520,
    "protein": 22,
    "carbs": 58,
    "fat": 24
  },
  "sections": [
    {
      "name": "Default",
      "ingredients": [
        { "item": "spaghetti", "quantity": 400, "unit": "g" },
        { "item": "guanciale", "quantity": 200, "unit": "g", "notes": "cut into small strips" },
        { "item": "egg yolks", "quantity": 6, "unit": "" },
        { "item": "whole eggs", "quantity": 2, "unit": "" },
        { "item": "pecorino romano", "quantity": 100, "unit": "g", "notes": "finely grated, plus extra for serving" },
        { "item": "black pepper", "quantity": 0, "unit": "", "notes": "freshly cracked, generous amount" },
        { "item": "salt", "quantity": 0, "unit": "", "notes": "for pasta water" }
      ],
      "steps": [
        {
          "instruction": "Bring a large pot of well-salted water to a rolling boil.",
          "duration": 600
        },
        {
          "instruction": "While water heats, combine egg yolks, whole eggs, and most of the pecorino in a bowl. Whisk until smooth. Add generous black pepper.",
          "duration": 120
        },
        {
          "instruction": "Cut guanciale into small strips or lardons. Place in a cold skillet and render over medium heat until fat is translucent and edges are crisp.",
          "duration": 480
        },
        {
          "instruction": "Cook spaghetti in the boiling water until 1 minute short of al dente. Reserve 2 cups of pasta water before draining.",
          "duration": 540
        },
        {
          "instruction": "Remove skillet from heat. Add drained pasta to the guanciale pan. Toss to coat in rendered fat.",
          "duration": 30
        },
        {
          "instruction": "Pour the egg and cheese mixture over the pasta, tossing vigorously. Add splashes of pasta water to create a creamy, emulsified sauce. Work quickly off heat to avoid scrambling the eggs.",
          "duration": 60
        },
        {
          "instruction": "Serve immediately with extra pecorino and cracked black pepper.",
          "duration": 0
        }
      ]
    }
  ]
}
```

## Notes on Data Integrity

- **Do not invent data.** If the source does not provide nutrition info, omit the `nutrition` object entirely. If the source does not specify servings, either omit `servings` (defaults to 4) or note the uncertainty in `summary`.
- **Preserve attribution.** Always populate `source_url` when extracting from a URL. Always populate `image_url` when the source provides a hero image.
- **Duration accuracy matters.** Cozina's cooking mode uses durations to power countdown timers. An incorrect duration (e.g., 30 seconds instead of 30 minutes) will cause timers to fire incorrectly. Double-check unit conversions: minutes to seconds, hours to seconds.
- **One action per step.** Do not merge multiple actions into a single step. "Preheat the oven and dice the onions" should be two separate steps. This enables accurate timer assignment and progress tracking in the cooking UI.
