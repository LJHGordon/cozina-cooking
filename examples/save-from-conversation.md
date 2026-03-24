# Example: Save Recipe from Conversation

This example demonstrates structuring a recipe from a casual, conversational description. The user does not provide a URL or formatted text -- they describe the recipe from memory.


## Conversation

**User:**
> My grandma makes this amazing chicken soup whenever someone is sick. She starts by boiling a whole chicken with onion, carrots, and celery for like an hour until it falls apart. Then she pulls out the chicken and shreds it, strains the broth, adds egg noodles and the shredded chicken back in, and cooks it for another 10 minutes. She always puts in fresh dill at the end. Oh and a squeeze of lemon.

**Assistant (internal steps):**

### Step 1: Identify What We Have

From the conversation, the assistant extracts:
- **Title candidate**: "Grandma's Chicken Soup" (inferred from context)
- **Ingredients mentioned**: whole chicken, onion, carrots, celery, egg noodles, fresh dill, lemon
- **Steps identified**: boil chicken with aromatics (1 hour), shred chicken, strain broth, add noodles and chicken (10 min), add dill and lemon
- **Missing**: exact quantities, servings, any seasoning details

### Step 2: Ask Clarifying Questions (Batched)

**Assistant response:**
> That sounds like a proper healing soup. Let me get a few details so I can save it accurately:
>
> 1. How many servings does this usually make -- enough for the family (6-8) or a smaller batch?
> 2. Roughly how many carrots and celery stalks does she use?
> 3. How much dill -- a few sprigs, or a bigger handful?
> 4. Does she season with salt and pepper, or anything else?

**User:**
> It makes a big pot, probably 8 servings. She uses like 3 carrots and 3 celery stalks, one big onion. A good handful of dill, maybe a quarter cup chopped. And yeah salt and pepper obviously. She also puts in 2 bay leaves while it boils.

### Step 3: Structure into Schema

The assistant now has enough information. Structure the recipe:

```json
{
  "title": "Grandma's Chicken Soup",
  "summary": "A restorative whole-chicken soup with egg noodles, fresh dill, and a squeeze of lemon. The kind you make when someone needs healing.",
  "servings": 8,
  "prep_time": "15 min",
  "cook_time": "1 hour 15 min",
  "cuisine": "American",
  "cooking_method": "Boiling",
  "keywords": ["soup", "chicken", "comfort", "family recipe"],
  "sections": [
    {
      "name": "Default",
      "ingredients": [
        { "item": "whole chicken", "quantity": 1, "unit": "", "notes": "about 4 lbs" },
        { "item": "onion", "quantity": 1, "unit": "", "notes": "large, quartered" },
        { "item": "carrots", "quantity": 3, "unit": "", "notes": "peeled and sliced" },
        { "item": "celery stalks", "quantity": 3, "unit": "", "notes": "chopped" },
        { "item": "bay leaves", "quantity": 2, "unit": "" },
        { "item": "egg noodles", "quantity": 8, "unit": "oz" },
        { "item": "fresh dill", "quantity": 0.25, "unit": "cup", "notes": "chopped" },
        { "item": "lemon", "quantity": 1, "unit": "", "notes": "juiced" },
        { "item": "salt", "quantity": 0, "unit": "", "notes": "to taste" },
        { "item": "black pepper", "quantity": 0, "unit": "", "notes": "to taste" }
      ],
      "steps": [
        {
          "instruction": "Place the whole chicken in a large stockpot. Add quartered onion, half of the carrots, half of the celery, and bay leaves. Cover with cold water by 2 inches. Bring to a boil, then reduce to a gentle simmer.",
          "duration": 120
        },
        {
          "instruction": "Simmer uncovered for 1 hour until the chicken is falling-apart tender. Skim any foam that rises to the surface.",
          "duration": 3600
        },
        {
          "instruction": "Remove the chicken from the pot and place on a cutting board. Let cool slightly, then shred the meat, discarding skin and bones.",
          "duration": 600
        },
        {
          "instruction": "Strain the broth through a fine mesh strainer into a clean pot, discarding the spent vegetables and bay leaves.",
          "duration": 120
        },
        {
          "instruction": "Bring the strained broth to a simmer. Add the remaining carrots and celery. Cook until just tender, about 5 minutes.",
          "duration": 300
        },
        {
          "instruction": "Add egg noodles and shredded chicken. Cook for 10 minutes until noodles are tender.",
          "duration": 600
        },
        {
          "instruction": "Remove from heat. Stir in chopped fresh dill and a squeeze of lemon juice. Season with salt and pepper. Serve hot.",
          "duration": 0
        }
      ]
    }
  ]
}
```

### Step 4: Present for Review

**Assistant response:**
> Here is what I put together from your description:
>
> **Grandma's Chicken Soup**
> A restorative whole-chicken soup with egg noodles, fresh dill, and a squeeze of lemon.
>
> - 10 ingredients
> - 7 steps
> - Prep: 15 min | Cook: 1 hour 15 min
> - Serves 8
>
> I made a couple of assumptions: quartered onion for the boil, splitting the carrots and celery between the broth-making stage and the final soup. Does that match how she makes it?
>
> Want me to save this to your Cozina?

**User:**
> That is perfect. Save it.

### Step 5: Save via MCP

Call `cozina_save_recipe` with the structured JSON. No `cookbook_id` specified, so it saves to the default collection.

### Step 6: Report Result

**Assistant response:**
> Saved. "Grandma's Chicken Soup" is in your Cozina now. If you want, I can also create a cookbook for it and organize future imports there.


## Key Patterns in Conversation Extraction

1. **Batch questions.** Do not ask one question at a time. Identify all gaps and ask them in a single message.
2. **Make reasonable assumptions.** The user said "like 3 carrots" -- use 3, do not ask for more precision. Save pedantry for quantities that actually matter (baking ratios, seasoning).
3. **Infer implicit ingredients.** The user said "boiling a whole chicken" -- water is implied. Do not add "12 cups water" as an ingredient; the step instruction "cover with cold water by 2 inches" is more accurate.
4. **Split preparation stages.** The user described one continuous process, but structuring it into discrete steps with durations makes it useful in Cozina's cooking mode with timers.
5. **Preserve voice.** The summary should reflect the recipe's origin story ("the kind you make when someone needs healing") rather than generic description.
