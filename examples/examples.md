# Tre eksempelspørgsmål

Alle tre er kørt i chatten i workflowet *Agent - Kokkeassistenten* med `qwen3:8b`. Screenshots af chat, tool calls og tool-output ligger i `screenshots/`.

---

## Eksempel 1 – Stemning (vektorsøgning)

**Spørgsmål:** *"Jeg vil gerne have noget cremet og hurtigt til en hverdagsaften"*

**Tool calls:** `vector_search` med `query: "creamy"` (k udeladt, så standard på 5 blev brugt)

**Hentet information:** fem opskrifter fra vektorindexet, sorteret efter similarity og beriget med minutter, ingredienser, tags og kategori fra grafen. Fx Cream Cheese Ball (RecipeId 30053, score 0.777, 10 min, kategori Very Low Carbs) og White Chocolate Chambord Cheesecake (RecipeId 444865, score 0.769, 35 min, kategori Dessert).

**Agentens svar (forkortet):**
1. Cream Cheese Ball (10 min) – en dip med grønne løg og ost. *(Source: Food.com RecipeId 30053)*
2. White Chocolate Chambord Cheesecake (35 min) – dessert med hvid chokolade og hindbærsauce. *(Source: Food.com RecipeId 444865)*
3. Raspberry Cream Cheese Swirl (60 min) – sød og cremet med hindbær og flødeost. *(Source: Food.com RecipeId 161893)*

Den spurgte afslutningsvis, om brugeren ville have alternativer.

**Screenshots:** `screenshots/eksempel1-chat.jpg` (chat, tool-forløb i Logs og agentens output), `screenshots/eksempel1-vector_search.jpg` (tool-input og output)

**Vurdering: delvis succes.**
- ✅ Valgte det rigtige tool til et beskrivende ønske og anbefalede kun opskrifter fra databasen, med kilde på hver.
- ⚠️ Opskrifterne er cremede, men ikke "aftensmad": en dip og to desserter. Agenten søgte kun på "creamy" og droppede "hurtigt" og "aftensmad" fra query, så embeddingen ramte cremede retter generelt. Den tjekkede heller ikke, om retterne passede til en hverdagsaften. Udsnittet på 500 tilfældige opskrifter har desuden få cremede hovedretter.
- ⚠️ Svartid ca. 4 minutter, fordi modellen kører på CPU.

---

## Eksempel 2 – Hårde krav (grafsøgning) – fejl/begrænsning

**Spørgsmål:** *"Find vegetariske opskrifter med kikærter under 30 minutter uden nødder"*

**Tool calls:** `graph_search`

**Input til tool:** modellen sendte alle krav som én tekststreng i et felt kaldet `query`:

```
{"include_ingredients": ["chickpeas"], "exclude_ingredients": ["nuts"], "max_minutes": 30, "category": "vegetarian"}
```

Workflowets egne inputfelter (`mustHave`, `exclude`, `tag`, `maxTime`, `category`) blev ikke udfyldt.

**Hentet information:** 10 opskrifter, men uden filtrering. Fordi inputfelterne var tomme, returnerede graph_search blot de 10 bedst ratede opskrifter i databasen, fx Chipotle Corn Chowder (RecipeId 162156, 50 min, indeholder bacon og chicken broth) og Blue Stuffed Mushrooms (RecipeId 93).

**Agentens svar (forkortet):** Der er ingen vegetariske opskrifter med kikærter under 30 minutter i databasen. Brugeren kan overveje at lempe kravene, fx tillade mere tid eller andre ingredienser.

**Screenshots:** `screenshots/eksempel2-chat.jpg` (chat og tool-forløb i Logs), `screenshots/eksempel2-graph_search.png` (tool-input og output)

**Vurdering: fejl i tool-kaldet. Agenten håndterede den dog ærligt.**
- ✅ Valgte det rigtige tool til hårde krav og oversatte kravene korrekt til engelsk (chickpeas, nuts, 30 minutter, vegetarian).
- ✅ Opdagede, at ingen af de returnerede opskrifter opfyldte kravene, opfandt ikke noget og foreslog at lempe kravene, som guardrails beskriver.
- ❌ **Parameter-mapping fejlede.** I *Call n8n Workflow*-tool-noden var workflowets inputfelter ikke sat til at blive udfyldt af modellen ("No parameters are set up to be filled by AI"). Modellen fik derfor kun ét generisk `query`-felt og pakkede alle krav ind i det som JSON. graph_search modtog tomme filtre og kørte en ufiltreret søgning. vector_search ramte ikke problemet, fordi dens inputfelt tilfældigvis også hedder `query`.
- ❌ Agenten konkluderede, at databasen intet indeholdt. Det er ikke nødvendigvis rigtigt, for filteret blev aldrig anvendt.
- **Rettelse:** markér hvert inputfelt i tool-noden som "defined by the model" (✨-knappen ved feltet), så modellen ser de rigtige parametre (`mustHave`, `exclude` osv.). Derudover kræver grafsøgningen datasættets egne termer: Food.com kalder ofte kikærter "garbanzo beans", og "vegetarian" er ikke nødvendigvis et tag. En synonymliste eller et tool, der viser de tilgængelige tags, ville gøre grafsøgningen mere robust.

---

## Eksempel 3 – Stemning + udelukkelse (begge typer)

**Spørgsmål:** *"Noget i stil med en hyggelig karry, men uden kokos"*

**Tool calls:** `vector_search` med `query: "cozy curry"`. **Ikke** `graph_search`.

**Hentet information:** fem karryretter, fx Jamaican Curry Chicken (RecipeId 344890, score 0.825, 60 min, kategori Curries), K C Ming (Beef Mince Curry), Subtropical Egg Curry (Revised) og Spicy Red Curry Beef (indeholder kokosmælk).

**Agentens svar (forkortet):** anbefalede Jamaican Curry Chicken, K C Ming (Beef Mince Curry) og Subtropical Egg Curry, og forklarede, at Spicy Red Curry Beef var udeladt, fordi den indeholder kokosmælk.

**Screenshots:** `screenshots/eksempel3-chat.png` (chat, tool-input og output)

**Vurdering: delvis succes.**
- ✅ Vektorsøgningen fandt relevante, hyggelige karryretter med høj similarity, og agenten sorterede selv opskriften med kokosmælk fra.
- ❌ Den kaldte ikke `graph_search` med `exclude: coconut`, selv om instruktionerne siger, at hårde krav skal tjekkes i grafen. Udelukkelsen skete kun ved, at modellen læste teksten. Det er mindre pålideligt, især ved allergier.
- ❌ Svaret mangler kildeangivelser, og agenten tilføjede egne "tips" (fx alternativer til kokosmælk), som ikke kommer fra databasen. Den lille lokale model følger altså ikke alle instruktioner konsekvent.
- **Mulig forbedring:** hybrid search i ét tool, hvor vektorsøgningen tager en `exclude`-parameter, så udelukkelsen altid sker i grafen. Alternativt en større model eller en evaluator-agent, der tjekker svaret mod instruktionerne.

---

## Fejl under udvikling (Agents builder)

Med n8n's Agents builder kaldte agenten aldrig tools. Dens "thinking" viste, at modellen kun kunne se builderens interne tools (`chat_action`, `delegate_subagent`, `write_todos`, `mark_session_failed`) og ikke de tilknyttede workflows. Tidligere forsøg med mindre modeller gav opfundne svar uden tool calls (fx forslag om levering af mad). Se `screenshots/agents-builder-opsaetning.png` og `screenshots/agents-builder-thinking.jpg`.

## Prompts brugt under udvikling

- "Jeg vil gerne have noget cremet og hurtigt til en hverdagsaften"
- "Use the vector_search tool to find creamy and quick recipes." (test af, om tools var tilgængelige)
- "Find vegetariske opskrifter med kikærter under 30 minutter uden nødder"
- "Noget i stil med en hyggelig karry, men uden kokos"
