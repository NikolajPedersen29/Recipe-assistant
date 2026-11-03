# Datamodel i Neo4j

Graf og vektorer ligger i **samme Neo4j-database**. Hver opskrift er en `Recipe`-node i grafen og har en `Chunk`-node med embedding til vektorsøgning.

## Datakilde

- **Datasæt:** *Food.com Recipes and Reviews* (Kaggle), filen `recipes.csv`
- **Udsnit:** 500 tilfældige opskrifter (`random_state=42`) med udfyldt beskrivelse, ingredienser og fremgangsmåde. Lavet med `data/make_sample.py` og gemt i `data/recipes_sample.csv`.
- **Brugte kolonner:** `RecipeId`, `Name`, `AuthorId`, `AuthorName`, `TotalTime`, `Description`, `RecipeCategory`, `Keywords`, `RecipeIngredientParts`, `AggregatedRating`, `Calories`, `RecipeServings`, `RecipeInstructions`

## Node-typer

| Label | Properties | Kilde |
|---|---|---|
| `Recipe` | `id` (string, unik), `name`, `totalTime` (minutter), `calories`, `rating`, `servings` | én pr. række |
| `Ingredient` | `name` (lowercase, unik) | `RecipeIngredientParts` |
| `Tag` | `name` (lowercase, unik) | `Keywords` |
| `Category` | `name` (unik) | `RecipeCategory` (mangler den: "Other") |
| `Author` | `id` (unik), `name` | `AuthorId`, `AuthorName` |
| `Chunk` | `id` (`<recipeId>-main`), `text`, `embedding` (768 floats), `source`, `type` | bygget af navn, beskrivelse, kategori, tags, ingredienser og fremgangsmåde |

## Relationstyper

| Relation | Betydning |
|---|---|
| `(Recipe)-[:HAS_INGREDIENT]->(Ingredient)` | opskriften bruger ingrediensen |
| `(Recipe)-[:HAS_TAG]->(Tag)` | opskriften er tagget (fx "vegan", "< 30 mins", "easy") |
| `(Recipe)-[:IN_CATEGORY]->(Category)` | opskriftens kategori |
| `(Recipe)-[:AUTHORED_BY]->(Author)` | hvem der har skrevet opskriften |
| `(Chunk)-[:PART_OF]->(Recipe)` | forbinder vektordata med grafen |

## Indexes og constraints

Se [`neo4j/setup.cypher`](../neo4j/setup.cypher):

- Unikke constraints på `Recipe.id`, `Ingredient.name`, `Tag.name`, `Category.name`, `Author.id` og `Chunk.id`. De forhindrer dubletter, når ingest køres flere gange, og gør opslag hurtige.
- Vektorindex `recipe_chunks` på `Chunk.embedding` med 768 dimensioner og cosine similarity.

## Chunking

Opskrifter er korte og har allerede en naturlig struktur, så hver opskrift er **én chunk**. Teksten samler navn, beskrivelse, kategori, tags, ingredienser og fremgangsmåde, så både stemning (beskrivelse) og indhold (ingredienser) indgår i embeddingen. Chunken har `source` med RecipeId, så agenten kan henvise til kilden.

## Normalisering

En AI foreslog node- og relationstyperne ud fra kolonnerne. De blev derefter tilpasset:

- Felterne i R-format (`c("butter", "sugar")`) parses til lister.
- Ingredienser og tags gøres lowercase og trimmes, og dubletter fjernes, så "Butter" og "butter" bliver samme node.
- `TotalTime` i ISO-format (`PT1H30M`) omregnes til minutter, så der kan filtreres på tid.
- Manglende kategori bliver "Other", så `MERGE` aldrig får en null-værdi.

## Eksempel på data

Én opskrift fra databasen, som `vector_search` returnerer den (teksten er forkortet):

```json
{
  "recipeId": "344890",
  "name": "Jamaican Curry Chicken",
  "minutes": 60,
  "rating": null,
  "category": "Curries",
  "ingredients": ["salt", "ground black pepper", "curry powder", "potatoes", "onion", "fresh thyme", "chicken", "garlic cloves"],
  "tags": ["chicken", "poultry", "< 60 mins", "spicy", "meat"],
  "text": "Jamaican Curry Chicken. | ... | Category: Curries | Tags: chicken, poultry, meat, spicy, < 60 mins | Ingredients: ... | Instructions: Wash chicken with cold water ...",
  "source": "Food.com recipes.csv (RecipeId 344890)"
}
```

Hele udsnittet ligger i `data/recipes_sample.csv`.

Udsnittet af grafen i Neo4j Browser (`MATCH (r:Recipe)-[rel]->(n) RETURN r, rel, n LIMIT 50`) ses i `screenshots/neo4j-graf.jpg`.
