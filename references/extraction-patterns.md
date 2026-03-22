# Extraction Patterns Reference

This document details every extraction method for converting external recipe sources into the Cozina recipe schema. Follow the priority chain: JSON-LD > Microdata > HTML parsing > text parsing > image OCR > conversation structuring. Stop at the first method that produces a complete recipe (title + at least one section with ingredients and steps).


## 1. JSON-LD Extraction

### Detection

Search the HTML `<head>` and `<body>` for `<script type="application/ld+json">` tags. A page may contain multiple JSON-LD blocks (breadcrumbs, organization, article, etc.). Filter for the block where the top-level `@type` is `"Recipe"`, or where `@graph` contains an object with `"@type": "Recipe"`.

### Field Mapping: schema.org to Cozina

| schema.org Field | Cozina Field | Transform |
|------------------|--------------|-----------|
| `name` | `title` | Direct string copy. |
| `description` | `summary` | Direct string copy. Truncate to 500 chars if extremely long. |
| `recipeIngredient[]` | `sections[0].ingredients[]` | Parse each string into `{ item, quantity, unit, notes }`. See Ingredient Parsing below. |
| `recipeInstructions[]` | `sections[0].steps[]` | See Instruction Parsing below. May be strings, `HowToStep` objects, or `HowToSection` objects. |
| `prepTime` | `prep_time` | ISO 8601 duration (e.g., `"PT15M"`). Convert: `"PT15M"` -> `"15 min"`, `"PT1H30M"` -> `"1 hour 30 min"`. |
| `cookTime` | `cook_time` | Same ISO 8601 conversion as `prepTime`. |
| `totalTime` | (informational) | Not directly stored but useful for validation: `prep + cook` should approximate `total`. |
| `recipeYield` | `servings` | Extract integer. `"4 servings"` -> `4`. `"Makes 12 cookies"` -> `12`. If ambiguous, use the first number. |
| `recipeCategory` | (store in `keywords`) | String or array. Add to `keywords`. |
| `recipeCuisine` | `cuisine` | String or array. If array, use the first value. |
| `image` | `image_url` | May be a string URL, an `ImageObject`, or an array. Extract the highest-resolution URL. |
| `nutrition` | `nutrition` | Map `calories` (strip " calories"), `proteinContent` (strip "g"), `carbohydrateContent`, `fatContent` to integer values. |
| `author` | (informational) | `author.name` or direct string. Not stored in the MCP schema but useful for `summary` attribution. |

### Instruction Parsing (JSON-LD)

The `recipeInstructions` field has three possible shapes:

**Shape 1: Array of strings**
```json
["Preheat oven to 375F.", "Mix dry ingredients.", "..."]
```
Map each string directly to `{ instruction: string, duration: <parsed>, rank: index }`.

**Shape 2: Array of HowToStep objects**
```json
[{ "@type": "HowToStep", "text": "Preheat oven to 375F." }]
```
Extract `text` field. Some HowToStep objects also have `name` (use as step title) and `url` (ignore).

**Shape 3: Array of HowToSection objects**
```json
[{
  "@type": "HowToSection",
  "name": "Dough",
  "itemListElement": [
    { "@type": "HowToStep", "text": "Mix flour and water." }
  ]
}]
```
Map each `HowToSection` to a Cozina `Section`. The section's `name` comes from the `HowToSection.name`. Steps come from `itemListElement`. This is the ideal case -- it preserves the recipe's multi-section structure.

### Ingredient Parsing (from strings)

Recipe websites store ingredients as flat strings in `recipeIngredient[]`. Apply this regex pattern to extract structured data:

```
^([\d½⅓⅔¼¾⅕⅖⅗⅘⅙⅚⅛⅜⅝⅞/.\s-]+)?\s*(cups?|tbsp|tablespoons?|tsp|teaspoons?|oz|ounces?|lbs?|pounds?|g|grams?|kg|kilograms?|ml|milliliters?|L|liters?|pinch|dash|cloves?|cans?|stalks?|heads?|bunch|bunches|pieces?|slices?|medium|large|small|whole|package|pkg|sticks?|sprigs?|handfuls?|sheets?)?\s*(?:of\s+)?(.+)$
```

Groups: `$1` = quantity (convert unicode fractions: 1/2 = 0.5), `$2` = unit, `$3` = item + notes.

Separate notes from item by splitting on parentheses, commas after the item name, or "or" clauses:
- `"1 cup butter, softened"` -> `item: "butter"`, `notes: "softened"`
- `"2 eggs (large)"` -> `item: "eggs"`, `notes: "large"`
- `"1 cup milk or almond milk"` -> `item: "milk"`, `notes: "or almond milk"`


## 2. Microdata Extraction

### Detection

Search for HTML elements with `itemscope` and `itemtype="https://schema.org/Recipe"`. This is the container element.

### Parsing

Within the container, find elements with `itemprop` attributes:

| itemprop | Cozina Field | Notes |
|----------|--------------|-------|
| `name` | `title` | `textContent` of the element. |
| `description` | `summary` | `textContent`. |
| `recipeIngredient` or `ingredients` | `sections[0].ingredients[]` | Each matching element is one ingredient string. Parse same as JSON-LD ingredients. |
| `recipeInstructions` | `sections[0].steps[]` | May be a single element with `<li>` children, or multiple elements each containing one step. |
| `prepTime` | `prep_time` | Check `content` attribute for ISO 8601 duration, otherwise parse `textContent`. |
| `cookTime` | `cook_time` | Same as `prepTime`. |
| `recipeYield` | `servings` | Parse integer from `textContent` or `content` attribute. |
| `image` | `image_url` | Check `src`, `content`, or `href` attribute. |
| `nutrition` | `nutrition` | Container element. Look for child elements with `itemprop` of `calories`, `proteinContent`, `fatContent`, `carbohydrateContent`. |

Microdata is less reliable than JSON-LD because websites often implement it inconsistently. Validate extracted data: if title is empty or ingredients array is empty, fall through to HTML parsing.


## 3. Plain HTML Parsing

Use this when no structured data is present. This is a heuristic approach -- accuracy varies by site.

### Title Extraction

1. Look for the largest heading (`h1`) within the main content area.
2. If multiple `h1` elements exist, prefer the one closest to the recipe content (near ingredients/steps).
3. Fall back to `<title>` tag, stripping the site name suffix (split on ` | ` or ` - `).

### Ingredient Extraction

1. Find a heading containing "ingredient" (case-insensitive).
2. Collect all `<li>` elements in the next `<ul>` or `<ol>` after that heading.
3. If no list is found, collect all `<p>` elements between the "ingredient" heading and the next heading.
4. Parse each text line using the ingredient regex from Section 1.

### Step Extraction

1. Find a heading containing "direction", "instruction", "method", or "step" (case-insensitive).
2. Collect all `<li>` elements in the next `<ol>` (or `<ul>`) after that heading.
3. If no list is found, collect `<p>` elements between this heading and the next heading or end of content.
4. Filter out empty strings, ad copy, and editorial notes (lines under 10 characters or lines containing "advertisement", "sponsored", "pin this").

### Time Extraction

Search for text patterns near the recipe content:
- `prep(?:\s*time)?[:\s]+(\d+)\s*(min|hour|hr)` -> `prep_time`
- `cook(?:\s*time)?[:\s]+(\d+)\s*(min|hour|hr)` -> `cook_time`
- `total(?:\s*time)?[:\s]+(\d+)\s*(min|hour|hr)` -> (validation only)


## 4. Text Parsing

For pasted text with no HTML structure.

### Section Detection

Split the text on lines matching these patterns (case-insensitive):
- `^#{1,3}\s+` (markdown headings)
- `^(ingredients|directions|instructions|steps|method|preparation|for the .+)[:\s]*$`
- `^[A-Z ]{4,}:?\s*$` (ALL-CAPS lines, likely section headers)

### Ingredient Lines

After finding the ingredient header, parse each subsequent non-empty line until the next header. Apply the ingredient regex. Lines that do not match the pattern (no quantity or no recognizable item) are likely sub-headers within the ingredient list (e.g., "For the sauce:") -- create a new section from them.

### Instruction Lines

After finding the instruction header, parse each subsequent non-empty line. Strip leading numbers (`1.`, `2)`, `Step 3:`). Each line becomes one step. Extract duration from time references within the instruction text.


## 5. Image OCR

When the user shares an image:

1. Use vision capabilities to extract all visible text from the image.
2. Apply the text parsing pipeline from Section 4 to the extracted text.
3. Image OCR commonly introduces errors in:
   - Fractions: `1⁄2` may OCR as `1/2`, `112`, or `½`
   - Temperature symbols: `350°F` may lose the degree symbol
   - Ingredient names: uncommon ingredients may be misspelled
4. After structuring, present the extracted recipe to the user and flag low-confidence fields. Ask them to confirm ingredient quantities and any unusual text.


## 6. Conversation Structuring

When the user describes a recipe conversationally:

1. Extract the recipe name from context. If ambiguous, ask: "What do you call this recipe?"
2. Identify ingredients mentioned inline. Convert casual language:
   - "a couple of eggs" -> `{ quantity: 2, unit: "", item: "eggs" }`
   - "some olive oil" -> `{ quantity: 0, unit: "", item: "olive oil", notes: "to taste" }`
   - "a handful of basil" -> `{ quantity: 1, unit: "handful", item: "basil" }`
3. Identify cooking actions and sequence them into steps.
4. Ask for missing information in a single follow-up (batch questions):
   - "A few things I want to confirm: How many servings does this make? About how long does the [specific step] take? Any other ingredients I missed?"
5. Structure the complete recipe and present for review before saving.
