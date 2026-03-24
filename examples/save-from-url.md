# Example: Save Recipe from URL

This example demonstrates the complete workflow for extracting a recipe from a web URL, structuring it into the Cozina schema, and saving it via the MCP tool.


## Conversation

**User:**
> Save this recipe to my Cozina: https://example.com/recipes/classic-beef-stew

**Assistant (internal steps):**

### Step 1: Fetch the Page Content

Fetch the URL using the web fetch tool. Retrieve the full HTML content.

### Step 2: Search for JSON-LD

Scan the HTML for `<script type="application/ld+json">` blocks. Found one:

```json
{
  "@context": "https://schema.org",
  "@type": "Recipe",
  "name": "Classic Beef Stew",
  "description": "A hearty, slow-simmered beef stew with tender vegetables and a rich gravy.",
  "image": "https://example.com/images/beef-stew-hero.jpg",
  "recipeYield": "6 servings",
  "prepTime": "PT20M",
  "cookTime": "PT2H30M",
  "recipeCuisine": "American",
  "recipeCategory": "Main Course",
  "nutrition": {
    "@type": "NutritionInformation",
    "calories": "380 calories",
    "proteinContent": "32g",
    "carbohydrateContent": "28g",
    "fatContent": "14g"
  },
  "recipeIngredient": [
    "2 lbs beef chuck, cut into 1-inch cubes",
    "3 tablespoons all-purpose flour",
    "2 tablespoons olive oil",
    "1 large onion, diced",
    "4 cloves garlic, minced",
    "4 cups beef broth",
    "1 cup red wine",
    "3 medium potatoes, cubed",
    "3 carrots, sliced into rounds",
    "2 stalks celery, chopped",
    "1 can (14 oz) diced tomatoes",
    "2 bay leaves",
    "1 teaspoon dried thyme",
    "Salt and pepper to taste"
  ],
  "recipeInstructions": [
    {
      "@type": "HowToStep",
      "text": "Toss beef cubes with flour, salt, and pepper until evenly coated."
    },
    {
      "@type": "HowToStep",
      "text": "Heat olive oil in a large Dutch oven over medium-high heat. Sear beef in batches until browned on all sides, about 3 minutes per batch. Remove and set aside."
    },
    {
      "@type": "HowToStep",
      "text": "Add onion and celery to the pot. Cook until softened, about 5 minutes. Add garlic and cook 1 minute more."
    },
    {
      "@type": "HowToStep",
      "text": "Pour in red wine, scraping up browned bits from the bottom. Simmer for 2 minutes."
    },
    {
      "@type": "HowToStep",
      "text": "Add beef broth, diced tomatoes, bay leaves, and thyme. Return the beef to the pot. Bring to a boil, then reduce heat to low. Cover and simmer for 1 hour 30 minutes."
    },
    {
      "@type": "HowToStep",
      "text": "Add potatoes and carrots. Continue simmering covered for 45 minutes until vegetables are tender and beef is fork-tender."
    },
    {
      "@type": "HowToStep",
      "text": "Remove bay leaves. Taste and adjust seasoning with salt and pepper. Serve hot."
    }
  ]
}
```

### Step 3: Map to Cozina Schema

Transform the JSON-LD into the Cozina recipe structure:

- `name` -> `title`: "Classic Beef Stew"
- `description` -> `summary`: "A hearty, slow-simmered beef stew..."
- `recipeYield` -> `servings`: parse "6 servings" -> `6`
- `prepTime` "PT20M" -> `prep_time`: "20 min"
- `cookTime` "PT2H30M" -> `cook_time`: "2 hours 30 min"
- `recipeCuisine` -> `cuisine`: "American"
- `recipeCategory` -> `keywords`: ["Main Course"]
- `image` -> `image_url`: direct copy
- `nutrition` -> parse integers: `{ calories: 380, protein: 32, carbs: 28, fat: 14 }`
- `recipeIngredient[]` -> parse each string into structured ingredients
- `recipeInstructions[]` (HowToStep) -> single section, steps with durations parsed from text

No `HowToSection` objects found, so all instructions go into one section named "Default".

### Step 4: Present to User

**Assistant response:**
> I extracted the recipe from that page. Here is what I found:
>
> **Classic Beef Stew**
> A hearty, slow-simmered beef stew with tender vegetables and a rich gravy.
>
> - 14 ingredients
> - 7 steps
> - Prep: 20 min | Cook: 2 hours 30 min
> - Serves 6 | American cuisine
> - Nutrition: 380 cal, 32g protein, 28g carbs, 14g fat per serving
>
> Want me to save this to your Cozina? It is free -- no credits used.

**User:**
> Yes, save it to my "Comfort Food" cookbook.

### Step 5: Resolve Or Create Cookbook

Call `cozina_list_cookbooks`. If "Comfort Food" already exists, use its UUID.

If it does not exist yet, ask for confirmation, then call `cozina_create_cookbook({ "name": "Comfort Food" })` and use the returned UUID.

### Step 6: Save via MCP

Call `cozina_save_recipe` with the structured JSON, including `cookbook_id` from the previous step.

### Step 7: Report Result

**Assistant response:**
> Done. "Classic Beef Stew" has been saved to your Comfort Food cookbook. You can find it in the Cozina app ready to cook.
