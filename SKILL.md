---
name: cozina-cooking
description: >
  This skill should be used when the user asks to "save a recipe to Cozina",
  "import a recipe", "add this to my cookbook", "what should I cook",
  "plan my meals", "what recipes do I have", or mentions cooking, recipes,
  meal planning, or grocery shopping in the context of their Cozina account.
  Transforms the AI into a Cozina-aware cooking companion that extracts recipes
  from any source (URLs, text, images, conversation) and saves them to the
  user's Cozina app via the cozina-mcp MCP server.
version: 1.0.0
---

# Cozina Cooking Skill

## 1. Purpose

Cozina is a recipe management and cooking app available on iOS, iPadOS, and the web. It stores recipes in a structured, section-based format that powers features like smart cooking timers, collaborative cooking sessions, shopping list generation, and AI-driven meal suggestions.

This skill enables four core capabilities:

1. **Recipe Extraction & Save** -- Extract structured recipe data from any source (URL, pasted text, image, or freeform conversation) and save it to the user's Cozina account.
2. **Cookbook-Aware Suggestions** -- Query the user's existing recipe collection to provide personalized cooking suggestions, filtered by cuisine, ingredient, keyword, or cookbook.
3. **Meal Planning & Shopping Lists** -- Build a 7-day plan from the user's saved recipes and persist the resulting meal plan and shopping list into Cozina.
4. **Account & Usage Awareness** -- Check the user's credit balance, subscription tier, and collection organization to provide context-appropriate guidance.

All operations use the `cozina-mcp` MCP server. Saving a recipe consumes zero AI credits -- always offer to save.


## 2. Prerequisites

### Verify MCP Server Availability

Check that the `cozina-mcp` MCP tools are available in the current session. The following tools must be callable:

- `cozina_save_recipe`
- `cozina_list_recipes`
- `cozina_get_recipe`
- `cozina_list_cookbooks`
- `cozina_get_usage`
- `cozina_get_meal_plan`
- `cozina_save_meal_plan`
- `cozina_clear_meal_plan`
- `cozina_get_shopping_list`
- `cozina_generate_shopping_list`
- `cozina_set_shopping_list_item_checked`

If any tool is missing, the MCP server is not configured. Guide the user through setup:

1. Open Cozina on iOS or the web.
2. Navigate to **Settings > API Access**.
3. Tap or click **Generate Token**. Leave both `read` and `write` enabled unless the user explicitly wants a browse-only token, then copy the token immediately. It is only shown once.
4. Add the following to the Claude MCP configuration file (`.claude/mcp.json` or equivalent):

```json
{
  "mcpServers": {
    "cozina": {
      "command": "npx",
      "args": ["-y", "cozina-mcp"],
      "env": {
        "COZINA_API_TOKEN": "coz_xxxx"
      }
    }
  }
}
```

5. Restart the Claude session so the MCP server initializes.

Do not attempt recipe operations until the token is confirmed working. Call `cozina_get_usage` as a connectivity test -- if it returns data, the server is live.


## 3. Core Workflows

### 3.1 Recipe Detection & Save

When the user shares a recipe source, follow this extraction priority chain. Stop at the first method that yields complete data:

**Priority 1: JSON-LD (structured data)**
Fetch the URL content. Search for `<script type="application/ld+json">` blocks containing `"@type": "Recipe"`. Parse the JSON-LD object and map schema.org fields to the Cozina schema. This is the most reliable source -- websites like AllRecipes, NYT Cooking, Bon Appetit, and most food blogs embed JSON-LD.

**Priority 2: Microdata (HTML attributes)**
If no JSON-LD is found, look for HTML elements with `itemtype="https://schema.org/Recipe"`. Parse `itemprop` attributes to extract title, ingredients, instructions, times, and nutrition. Less common than JSON-LD but still structured.

**Priority 3: Plain HTML Parsing**
If no structured data exists, parse the page content heuristically. Look for headings that indicate recipe sections (`h1`/`h2` for title), unordered/ordered lists (ingredients and steps), and time-related metadata. See `references/extraction-patterns.md` for detailed heuristics.

**Priority 4: Text Parsing**
When the user pastes raw text (no URL), split on common section headers: "Ingredients", "Directions", "Instructions", "Steps", "Method". Parse ingredient lines into `{ item, quantity, unit }` triples. Parse instruction lines into ordered steps. See `references/extraction-patterns.md` for the parsing grammar.

**Priority 5: Image OCR**
When the user shares an image (screenshot, photo of a cookbook page, handwritten recipe card), use vision capabilities to OCR the text content. Then apply the text parsing pipeline from Priority 4. Flag any low-confidence extractions and ask the user to confirm.

**Priority 6: Conversation Structuring**
When the user describes a recipe conversationally ("my grandma makes this pasta where she..."), actively structure the description into the Cozina schema. Ask targeted follow-up questions for missing required fields:
- Title (if not obvious from context)
- Ingredients with quantities and units
- Step-by-step instructions with approximate durations
- Servings count

After extraction, always:

1. Present the structured recipe to the user for review. Show title, ingredient count, step count, and any metadata (cuisine, times, nutrition).
2. Ask if they want to save it. Mention that saving is free (no credits consumed).
3. If the user confirms, call `cozina_save_recipe` with the structured data.
4. Report the result: recipe title, ID, and a deep link if available.

### 3.2 Multi-Section Recipe Handling

Many recipes have distinct components: a cake with frosting, a taco with salsa and slaw, a stew with dumplings. Detect multi-section structure from the source and preserve it.

**Signals that a recipe has multiple sections:**
- JSON-LD `recipeInstructions` contains `HowToSection` objects (strongest signal).
- Ingredient list has sub-headers like "For the dough:", "For the filling:".
- Instructions have phase markers like "Make the sauce:", "Assemble:".
- The user says "there are two parts" or describes separate components.

Map each component to a separate Cozina `Section` with its own `name`, `ingredients[]`, and `steps[]`. Do not collapse them into a single section -- Cozina's cooking mode tracks progress per section and allows users to work on multiple components in parallel.

For single-section recipes, use `"name": "Default"`.

### 3.3 Cookbook-Aware Suggestions

When the user asks "what should I cook" or similar:

1. Call `cozina_list_recipes` to retrieve their collection. Apply filters if the user mentioned a cuisine, ingredient, or keyword.
2. If the user has cookbooks, call `cozina_list_cookbooks` to understand their organization (e.g., "Weeknight Dinners", "Holiday Baking").
3. Suggest 3-5 recipes from their existing collection, prioritized by:
   - Recency (recently added or cooked)
   - Match quality (if filters were applied)
   - Variety (avoid suggesting similar recipes)
4. If the collection is empty or no matches are found, offer to help find and save new recipes.
5. When the user picks a recipe, call `cozina_get_recipe` to retrieve full details including all sections, ingredients, and steps.
6. Present the full recipe in a clean, readable format: ingredients first, then steps grouped by section.

### 3.4 Meal Planning

When the user asks for a weekly plan, a next-7-days plan, or help filling their planner:

1. Call `cozina_list_recipes` and `cozina_list_cookbooks` to understand what they already have.
2. Ask targeted follow-up questions if needed: serving count, cuisine variety, leftovers, dietary constraints, and start date.
3. Draft a 7-day plan as dated recipe assignments using `breakfast`, `lunch`, `dinner`, and `snack`.
4. Present the plan clearly and ask for confirmation before saving.
5. Save the confirmed entries with `cozina_save_meal_plan`.
6. If the user wants to start over, clear the date range with `cozina_clear_meal_plan`.

Use `cozina_get_meal_plan` whenever the user asks what is already planned or wants adjustments to an existing week.

### 3.5 Shopping Lists

When the user asks for a shopping list:

1. Determine the source:
   - explicit recipes
   - or the saved meal plan for the next 7 days (or a provided range)
2. Call `cozina_generate_shopping_list`.
3. Call `cozina_get_shopping_list` and present the active list grouped by aisle/category when possible.
4. If the user asks to check or uncheck items, call `cozina_set_shopping_list_item_checked`.

The shopping list is persisted into Cozina. Do not describe it as a text-only workaround.

### 3.6 Error Handling & Edge Cases

**URL fails to load:** Some recipe websites block automated fetching (paywalled content, aggressive bot detection). If the page content cannot be retrieved, ask the user to paste the recipe text or share a screenshot instead. Do not retry the URL repeatedly.

**Incomplete extraction:** If extraction produces a title but no ingredients, or ingredients but no steps, present what was found and ask the user to fill in the gaps. A partial save is better than no save -- the user can edit the recipe in the Cozina app later.

**Duplicate detection:** Before saving, check if a recipe with the same title and source URL already exists by calling `cozina_list_recipes` with a query matching the title. If a likely duplicate is found, inform the user: "You already have a recipe called 'Chicken Parmesan' from this URL. Want me to save it as a new copy, or skip?"

**Non-recipe URLs:** If the user shares a URL that does not contain a recipe (a blog post, a restaurant menu, a video without recipe data), say so clearly. Do not fabricate recipe data from non-recipe content.

### 3.7 Usage & Profile

When the user asks about their account, credits, or subscription:

1. Call `cozina_get_usage` to retrieve current usage data.
2. Report their subscription tier, monthly credit usage (used/limit), remaining credits, and top-up balance.
3. Break down usage by action type if the `by_action` map contains data.
4. If credits are low, note that saving recipes is always free -- only AI-powered features (remix, chat, generation) consume credits.

### 3.8 Save Offer Policy

Always offer to save a recipe when:
- The user shares a URL that contains a recipe
- The user pastes recipe text
- The user describes a recipe in conversation
- A recipe is generated or remixed during an AI interaction

Frame the offer naturally: "Want me to save this to your Cozina?" rather than a generic prompt. Saving costs zero credits, so there is no reason not to offer.


## 4. Recipe Schema Quick Reference

There are two related recipe schemas in Cozina:

1. The **full extraction schema** used when extracting and normalizing recipe data from URLs, videos, OCR, or conversation.
2. The **MCP save payload** accepted by `cozina_save_recipe`.

Do not treat them as the same object. Extract into the richer schema first, then translate into the save payload before calling `cozina_save_recipe`.

### Full Recipe Extraction Schema

This is the richer extraction shape defined in `server/main.py`.

```json
{
  "name": "Recipe Name",
  "author_name": "Original Creator Name",
  "slug": "recipe-name-slug",
  "description": "A brief, appetizing summary of the dish.",
  "image_url": "https://example.com/recipe-image.jpg",
  "video_timestamp": "2:45",
  "servings": 4,
  "cuisine": "Italian",
  "cooking_method": "Roasting",
  "language": "<TARGET_LANGUAGE>",
  "total_time": "45 minutes",
  "prep_time": 900,
  "prep_time_text": "15 minutes",
  "cook_time": 1800,
  "cook_time_text": "30 minutes",
  "recipe_category": ["Dinner", "Italian"],
  "keywords": ["Healthy", "Vegetarian"],
  "tool": ["Oven", "Whisk"],
  "sections": [
    {
      "name": "Default",
      "ingredients": [
        {
          "name": "Ingredient Name",
          "quantity": 1.5,
          "unit": "cup",
          "notes": "optional",
          "metadata": {
            "substitutes": ["Alternative 1", "Alternative 2"],
            "usda_tag": "Vegetables"
          }
        }
      ],
      "steps": [
        {
          "rank": 1,
          "title": "Step Title",
          "instruction": "Step instruction...",
          "duration": 300,
          "timestamp": null,
          "is_passive": false,
          "is_prep": false,
          "equipment": ["oven"],
          "metadata": {
            "temp_mode": "400°F",
            "tips": "Don't open the oven door!",
            "warnings": "Hot steam.",
            "critical": true
          }
        }
      ]
    }
  ],
  "metadata": {
    "success_points": ["Use cold butter", "Don't overmix"],
    "safety_warnings": ["Raw eggs", "Hot oil"],
    "chef_tips": ["Rest the dough for 1 hour for best results."],
    "troubleshooting": [
      { "problem": "Dough is crumbly", "solution": "Add 1 tbsp water" }
    ],
    "equipment_with_alternatives": [
      { "name": "Stand Mixer", "alternative": "Hand mixer or strong arm" }
    ],
    "dietary_tags": ["Gluten-Free", "Keto-Friendly"],
    "allergen_tags": ["Dairy", "Eggs"],
    "nutrition_label": {
      "calories": 350,
      "total_fat": "15g",
      "saturated_fat": "5g",
      "trans_fat": "0g",
      "cholesterol": "45mg",
      "sodium": "400mg",
      "total_carbohydrates": "40g",
      "dietary_fiber": "3g",
      "sugars": "5g",
      "protein": "12g"
    }
  }
}
```

### Extraction To Save Payload Bridge

Translate the extraction schema into the public save payload before calling `cozina_save_recipe`.

| Extraction schema | Save payload | Notes |
|---|---|---|
| `name` | `title` | Top-level recipe title field changes names. |
| `description` | `summary` | Use the extracted description as the recipe summary when saving. |
| `ingredients[].name` | `ingredients[].item` | Ingredient field changes names. |
| `prep_time` / `cook_time` | `prep_time` / `cook_time` | Extraction uses numeric seconds; save payload uses human-readable text strings. Prefer `prep_time_text` and `cook_time_text` when available. |
| `metadata.nutrition_label` | `nutrition` | Save payload only supports `calories`, `protein`, `carbs`, and `fat`. |

Fields such as `author_name`, `slug`, `video_timestamp`, `language`, `total_time`, `recipe_category`, `tool`, step metadata, and recipe metadata like `chef_tips` or `success_points` are useful during extraction and reasoning, but they are not part of the current `cozina_save_recipe` contract unless you explicitly map them into supported fields.

### MCP Save Payload

The `cozina_save_recipe` tool accepts this structure. See `references/recipe-schema.md` for the complete field-by-field documentation of the save payload.

```json
{
  "title": "string (required)",
  "sections": [
    {
      "name": "string",
      "ingredients": [
        {
          "item": "string (required)",
          "quantity": "number (optional)",
          "unit": "string (optional)",
          "notes": "string (optional)"
        }
      ],
      "steps": [
        {
          "instruction": "string (required)",
          "duration": "number in seconds (optional)",
          "rank": "number (optional, auto-assigned if omitted)"
        }
      ]
    }
  ],
  "summary": "string (optional)",
  "servings": "number (optional, default 4)",
  "prep_time": "string (optional, e.g. '15 min')",
  "cook_time": "string (optional, e.g. '30 min')",
  "cuisine": "string (optional, e.g. 'Italian')",
  "cooking_method": "string (optional, e.g. 'Baking')",
  "source_url": "string (optional, original URL)",
  "image_url": "string (optional, hero image URL)",
  "nutrition": {
    "calories": "number (optional)",
    "protein": "number in grams (optional)",
    "carbs": "number in grams (optional)",
    "fat": "number in grams (optional)"
  },
  "keywords": ["string array (optional)"],
  "cookbook_id": "UUID string (optional, target cookbook)"
}
```

**Key rules:**
- `title` and `sections` are the only required fields. Everything else is optional.
- Every recipe must have at least one section. Use `"name": "Default"` for single-section recipes.
- Ingredient `quantity` is a number (use `0` for "to taste" items). Unit is a free-form string.
- Step `duration` is in **seconds**. Convert from minutes: multiply by 60. A 5-minute simmer = `300`.
- `cookbook_id` must be a valid UUID from `cozina_list_cookbooks`. If omitted, the recipe goes to the default collection.


## 5. Extraction Patterns Quick Reference

Extraction follows a strict priority chain: JSON-LD > Microdata > HTML parsing > text parsing > image OCR > conversation structuring. Each level is a fallback -- stop at the first one that produces a complete recipe.

**JSON-LD** is the gold standard. Over 80% of major recipe websites embed it. The `@type: "Recipe"` object contains `name`, `recipeIngredient[]`, `recipeInstructions[]`, `prepTime` (ISO 8601 duration), `cookTime`, `recipeYield`, `nutrition`, and `image`.

**Text parsing** splits on known headers and applies regex patterns for ingredient lines: `^(\d+[\d/.\s]*)\s*(cups?|tbsp|tsp|oz|lbs?|g|kg|ml|L|pinch|dash|cloves?|cans?|stalks?|heads?|bunch|pieces?|slices?|medium|large|small)?\s+(.+)$`.

See `references/extraction-patterns.md` for the complete pattern library, field mapping tables, and edge case handling.


## 6. Additional Resources

### Reference Documentation

- **[Recipe Schema Reference](references/recipe-schema.md)** -- Complete field-by-field documentation of the Cozina recipe JSON schema. Includes every field, its type, constraints, default values, and example data. Use this when you need to verify whether a field is required, what format a value should take, or how nested objects are structured.

- **[Extraction Patterns Reference](references/extraction-patterns.md)** -- Detailed extraction patterns for every source type. Includes JSON-LD field mapping from schema.org, Microdata parsing rules, HTML heuristics, text parsing regex patterns, and image OCR post-processing steps. Use this when an extraction fails or produces incomplete data.

- **[Cooking Workflows Reference](references/cooking-workflows.md)** -- Current meal-planning and shopping-list workflows supported by `cozina-mcp`, including how to save a week plan and generate the active shopping list.

### Example Workflows

- **[Save from URL](examples/save-from-url.md)** -- End-to-end example of extracting a recipe from a web URL using JSON-LD, structuring it, and saving via MCP.

- **[Save from Conversation](examples/save-from-conversation.md)** -- End-to-end example of structuring a recipe described conversationally by the user, asking clarifying questions, and saving.

- **[Cookbook-Aware Suggestion](examples/cookbook-aware-suggestion.md)** -- End-to-end example of querying the user's existing collection and suggesting recipes based on filters.

- **[Weekly Meal Plan](examples/weekly-meal-plan.md)** -- Example of building and saving a 7-day meal plan into Cozina.

- **[Shopping List From Plan](examples/shopping-list-from-plan.md)** -- Example of generating the active shopping list from a saved meal-plan range.
