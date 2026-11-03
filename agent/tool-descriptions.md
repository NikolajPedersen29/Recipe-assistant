# Tool-beskrivelser

Hvert tool er et workflow tool i agenten, der kalder et af retrieval-workflowene. Kopiér **Navn** og **Beskrivelse** ind i tool-opsætningen, og lad agenten udfylde inputfelterne.

---

## vector_search

**Workflow:** `vector_search` (`workflows/tool-vector-search.json`)

**Beskrivelse (til agenten):**
Semantic search over recipe descriptions. Use when the user describes taste, mood, texture or style in free text (e.g. "something creamy and quick", "cozy winter food"). Input `query`: a short English description of what the user wants. Input `k`: number of results (default 5). Returns the most similar recipes with recipeId, name, similarity score, minutes, rating, category, ingredients, tags, the matched text and its source.

**Input:**
| Felt | Type | Eksempel |
|---|---|---|
| query | string | "creamy pasta that is quick to make" |
| k | number | 5 |

**Hvad sker der:** Query embeddes med `nomic-embed-text` i Ollama → `db.index.vector.queryNodes('recipe_chunks', …)` finder de nærmeste `Chunk`-noder → hver chunk følges via `PART_OF` til sin `Recipe`, og opskriftens ingredienser, tags og kategori hentes fra grafen.

---

## graph_search

**Workflow:** `graph_search` (`workflows/tool-graph-search.json`)

**Beskrivelse (til agenten):**
Filter recipes by hard requirements in the recipe graph. Use when the user names ingredients that must be included or excluded, a maximum time, a tag or a category. All values in lowercase English. Inputs: `mustHave` (comma-separated ingredients that must all be in the recipe), `exclude` (comma-separated ingredients/allergens that must not be in the recipe), `tag` (one tag, e.g. "vegetarian", "easy", "< 30 mins"), `maxTime` (max total minutes), `category` (e.g. "dessert", "chicken"). Leave unused inputs empty. Returns up to 10 recipes sorted by rating, with their ingredients, tags and source.

**Input:**
| Felt | Type | Eksempel |
|---|---|---|
| mustHave | string (kommasepareret) | "chickpeas, garlic" |
| exclude | string (kommasepareret) | "nut, coconut" |
| tag | string | "vegetarian" |
| maxTime | number | 30 |
| category | string | "dessert" |

**Hvad sker der:** Parameteriseret Cypher på `(Recipe)-[:HAS_INGREDIENT]->(Ingredient)`, `-[:HAS_TAG]->(Tag)` og `-[:IN_CATEGORY]->(Category)`. Ingredienser matches med `CONTAINS`, så "coconut" også udelukker "coconut milk". Agenten skriver aldrig selv Cypher. Det er en bevidst guardrail.

---

## related_recipes

**Workflow:** `related_recipes` (`workflows/tool-related-recipes.json`)

**Beskrivelse (til agenten):**
Find recipes related to a specific recipe through the graph (shared ingredients and tags). Use when the user wants alternatives to, or something similar to, a recipe that was already found. Input `recipeId`: the id from an earlier result. Returns up to 5 recipes with the number of shared ingredients, which ingredients they share, shared tags and source.

**Input:**
| Felt | Type | Eksempel |
|---|---|---|
| recipeId | string | "38" |

**Hvad sker der:** Graftraversering `(Recipe)-[:HAS_INGREDIENT]->(Ingredient)<-[:HAS_INGREDIENT]-(andre Recipe)`, sorteret efter antal fælles ingredienser og tags.
