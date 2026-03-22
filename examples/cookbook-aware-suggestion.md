# Example: Cookbook-Aware Suggestion

This example demonstrates querying the user's existing recipe collection and providing personalized cooking suggestions based on their library, rather than suggesting generic recipes from the internet.


## Conversation

**User:**
> What should I cook tonight? I have about 30 minutes.

**Assistant (internal steps):**

### Step 1: Query the Recipe Collection

The user has a time constraint (30 minutes). Call `cozina_list_recipes` to get their collection. The API returns a paginated list of recipes with basic metadata including `prep_time`, `cook_time`, `cuisine`, and `title`.

```
cozina_list_recipes({ per_page: 50 })
```

Response includes 23 recipes. Filter client-side for recipes where the combined prep and cook time is under 30 minutes or is not specified (some recipes may have been saved without time data).

Candidates after filtering:
- "15-Minute Garlic Shrimp Pasta" (prep: 5 min, cook: 10 min, cuisine: Italian)
- "Quick Black Bean Tacos" (prep: 10 min, cook: 5 min, cuisine: Mexican)
- "Miso Glazed Salmon" (prep: 5 min, cook: 12 min, cuisine: Japanese)
- "Caprese Salad" (prep: 10 min, cook: 0 min, cuisine: Italian)
- "Stir-Fried Rice Noodles" (prep: 10 min, cook: 10 min, cuisine: Thai)

### Step 2: Check Cookbooks for Context

Call `cozina_list_cookbooks` to understand how the user organizes their recipes.

```
cozina_list_cookbooks()
```

Response:
- "Weeknight Winners" (12 recipes)
- "Date Night" (5 recipes)
- "Meal Prep Sunday" (6 recipes)

The "Weeknight Winners" cookbook is the most relevant given the context (quick, tonight). Note this for the suggestion framing.

### Step 3: Present Suggestions

**Assistant response:**
> Based on your collection and the 30-minute window, here are three options:
>
> 1. **15-Minute Garlic Shrimp Pasta** -- Italian, 15 min total. Your fastest option.
> 2. **Miso Glazed Salmon** -- Japanese, 17 min total. Simple and different from pasta.
> 3. **Quick Black Bean Tacos** -- Mexican, 15 min total. Minimal cleanup.
>
> All three are in your library. Want the full recipe for any of these?

**User:**
> The salmon sounds good. Show me that one.

### Step 4: Retrieve Full Recipe

Call `cozina_get_recipe` with the salmon recipe's ID to get the complete details including all sections, ingredients, and steps.

```
cozina_get_recipe("a1b2c3d4-...")
```

### Step 5: Present Full Recipe

**Assistant response:**
> **Miso Glazed Salmon**
> Serves 2 | Prep: 5 min | Cook: 12 min
>
> **Ingredients:**
> - 2 salmon fillets (6 oz each, skin-on)
> - 2 tbsp white miso paste
> - 1 tbsp mirin
> - 1 tbsp soy sauce
> - 1 tsp sesame oil
> - 1 tsp rice vinegar
> - Green onions for garnish
>
> **Steps:**
> 1. Preheat broiler to high. Line a baking sheet with foil.
> 2. Whisk together miso, mirin, soy sauce, sesame oil, and rice vinegar.
> 3. Pat salmon dry and place skin-side down on the baking sheet.
> 4. Spread the miso glaze evenly over the top of each fillet.
> 5. Broil 8-10 minutes until the glaze is caramelized and salmon is cooked through.
> 6. Garnish with sliced green onions. Serve with steamed rice.
>
> Ready to start cooking? You can open this in Cozina's cooking mode for step-by-step timers.

**User:**
> Perfect, I will make that. Thanks.


## Key Patterns in Cookbook-Aware Suggestions

1. **Always query the user's collection first.** Do not suggest recipes from the internet when the user has a personal library. Their saved recipes represent their taste preferences, dietary restrictions, and skill level.

2. **Apply implicit filters.** "Tonight" implies quick. "Weeknight" implies family-friendly and under 45 minutes. "Something different" implies avoid recently cooked recipes or frequently chosen cuisines.

3. **Suggest 3-5 options, not 1.** Give the user a choice. Vary by cuisine, protein, or cooking method to offer genuine variety.

4. **Show enough metadata to decide.** Title, cuisine, and total time are the minimum. Do not dump the full recipe unless the user asks for it.

5. **Retrieve full details only on demand.** Call `cozina_get_recipe` only after the user picks one. This keeps the initial suggestion response fast and clean.

6. **Bridge to the app.** After presenting the full recipe, mention Cozina's cooking mode. The user may want the step-by-step timer experience rather than reading instructions from a chat window.

7. **Handle empty collections gracefully.** If the user has no saved recipes or none match the filter, pivot: "Your collection does not have anything under 30 minutes yet. Want me to find a quick recipe and save it for you?"
