# Opskrift-RAG – Kokkeassistenten

En retrieval-agent, der anbefaler opskrifter ved at kombinere **semantisk vektorsøgning** og **graftraversering** i Neo4j. Data indlæses med n8n-workflows, og en AI-agent i n8n bruger begge typer søgning som tools.

## Case og agentens rolle

**Domæne:** opskrifter fra datasættet *Food.com Recipes and Reviews* (Kaggle).

Opskrifter er et godt case for RAG med både graf og vektor:

- **Graf:** opskrifter har eksplicit struktur. De *har* ingredienser, *tilhører* en kategori og *er tagget* (fx "vegan", "< 30 mins"). Det er hårde krav, der skal overholdes præcist ("uden nødder", "under 30 minutter").
- **Vektor:** brugere beskriver ofte, hvad de har lyst til, med stemning og tekstur ("noget cremet og hurtigt", "en hyggelig karry"). Her betyder mening mere end nøgleord.

**Rolle:** *Kokkeassistenten* er en venlig og praktisk opskriftsanbefaler. Den finder op til tre opskrifter, der passer til brugerens lyst, tid og begrænsninger, og angiver altid kilden (Food.com RecipeId) på hver.

## Arkitektur

```
                 ┌──────────────────── n8n ────────────────────┐
recipes_sample   │  Ingest-workflow                             │
   .csv ────────►│  Læs CSV → Parse → ┬─► Graf-Cypher ─────────┼──► Neo4j
                 │                    └─► Ollama embed → Chunk ─┼──►  (graf + vektorindex)
                 │                                              │        ▲
  Bruger ◄──────►│  Chat → AI Agent (qwen3:8b via Ollama)       │        │
                 │           ├─ tool: vector_search ────────────┼────────┤
                 │           ├─ tool: graph_search ─────────────┼────────┤
                 │           └─ tool: related_recipes ──────────┼────────┘
                 └──────────────────────────────────────────────┘
```

| Komponent | Teknologi |
|---|---|
| Graf- og vektordatabase | Neo4j 5 (samme database til begge) |
| Embeddings | `nomic-embed-text` i Ollama (768 dimensioner, cosine) |
| Chatmodel | `qwen3:8b` i Ollama |
| Workflows og agent | n8n (selvhostet i Docker) |

Vektor og graf hænger sammen, fordi hver `Chunk` (med embedding) er koblet til sin `Recipe` via `PART_OF`. Vektorsøgningen beriges derfor med fakta fra grafen (ingredienser, tags, kategori) i samme query.

## Hvor finder du hvad

| Krav | Placering |
|---|---|
| Opsætning (Docker) | [`docker-compose.yml`](docker-compose.yml) |
| Datakilde, sample og script | [`data/`](data/) |
| Neo4j-skema (noder, relationer, properties) | [`docs/schema.md`](docs/schema.md) |
| Constraints og vektorindex | [`neo4j/setup.cypher`](neo4j/setup.cypher) |
| Workflow JSON | [`workflows/`](workflows/) |
| Workflow-screenshots | [`screenshots/`](screenshots/), se oversigten nedenfor |
| Agentens model og konfiguration | [`agent/model.md`](agent/model.md) |
| Agentens instruktioner | [`agent/instructions.md`](agent/instructions.md) |
| Tool-beskrivelser | [`agent/tool-descriptions.md`](agent/tool-descriptions.md) |
| Tre eksempelspørgsmål med vurdering | [`examples/examples.md`](examples/examples.md) |

### Workflows

| Fil | Formål |
|---|---|
| `workflows/ingest-opskrifter.json` | **Dataindlæsning:** læser CSV, parser og normaliserer, skriver grafen og embeddings til Neo4j |
| `workflows/tool-vector-search.json` | **Vektorsøgning:** embedder brugerens beskrivelse og finder de mest lignende opskrifter |
| `workflows/tool-graph-search.json` | **Grafsøgning:** filtrerer opskrifter på ingredienser, udelukkelser, tag, tid og kategori |
| `workflows/tool-related-recipes.json` | **Grafsøgning:** finder opskrifter, der deler flest ingredienser og tags med en given opskrift |
| `workflows/agent-kokkeassistenten.json` | **Agent:** chat + AI Agent med model, hukommelse og de tre tools |

### Screenshots

| Fil | Viser |
|---|---|
| `screenshots/workflow-ingest.jpg` | Dataindlæsning: hele ingest-workflowet efter en vellykket kørsel |
| `screenshots/workflow-vector_search.png` | Vektorsøgning: retrieval-workflowet |
| `screenshots/workflow-graph_search.png` | Grafsøgning: retrieval-workflowet |
| `screenshots/workflow-related_recipes.png` | Grafsøgning: relaterede opskrifter |
| `screenshots/eksempel1-chat.jpg` | Agent-workflowet med tool-forbindelser, chat og tool-forløb |
| `screenshots/neo4j-graf.jpg` | Udsnit af grafen og antal opskrifter og embeddings |
| `screenshots/agents-builder-opsaetning.png` | Agentens konfiguration i Agents builder |
| `screenshots/agents-builder-thinking.jpg` | Agents builder: modellen kan ikke se workflow-tools |
| `screenshots/eksempel*.{jpg,png}` | De tre eksempler, se [`examples/examples.md`](examples/examples.md) |

## Kør projektet selv

**Krav:** Docker Desktop (mindst 8 GB RAM til Docker) og Python 3.

1. **Start tjenesterne**
   ```
   docker compose up -d
   docker compose exec ollama ollama pull nomic-embed-text
   docker compose exec ollama ollama pull qwen3:8b
   ```
   n8n kører på http://localhost:5679, Neo4j på http://localhost:7474 (`neo4j` / `password123`).

2. **Lav datasættet.** Download *Food.com Recipes and Reviews* fra Kaggle, læg `recipes.csv` i `data/`, og kør:
   ```
   python3 -m venv .venv && source .venv/bin/activate
   pip install pandas
   python data/make_sample.py
   ```
   Et færdigt udsnit (`data/recipes_sample.csv`, 500 opskrifter) ligger allerede i repoet.

3. **Opret constraints og vektorindex.** Kør kommandoerne i `neo4j/setup.cypher` i Neo4j Browser.

4. **Importér workflows i n8n.** Importér alle filer i `workflows/`. Publicér de tre tool-workflows.

5. **Indlæs data.** Kør *Ingest - Opskrifter*. Tjek i Neo4j:
   ```
   MATCH (r:Recipe) RETURN count(r);                               // 500
   MATCH (c:Chunk) WHERE c.embedding IS NOT NULL RETURN count(c);  // 500
   ```
   Workflowet bruger `MERGE`, så det kan køres flere gange uden dubletter. Nulstil med `MATCH (n) DETACH DELETE n`.

6. **Opsæt agenten.** I *Agent - Kokkeassistenten*: vælg en Ollama-credential (Base URL `http://ollama:11434`) på chatmodellen, og vælg det tilsvarende workflow i hver af de tre tool-noder. Åbn chatten og stil et spørgsmål.

## Brug af AI i processen

- **Grafmodel:** node- og relationstyper blev foreslået af en AI ud fra datasættets kolonner og derefter normaliseret (ingredienser og tags i lowercase, dubletter fjernet, tid omregnet til minutter). Se [`docs/schema.md`](docs/schema.md).
- **Cypher:** import-Cypher og retrieval-queries blev genereret med AI (Claude) og indbygget i n8n-workflowene, så importen er automatiseret.

## Kendte begrænsninger

- **Agents builder:** agenten blev først bygget i n8n's Agents builder (Preview). Workflow-tools blev ikke eksponeret for modellen, selv om både workflows og agent var publiceret (modellen så kun builderens interne tools). Den endelige agent er derfor bygget med n8n's AI Agent-node, som bruger de samme tre retrieval-workflows som tools.
- **Lokal model på CPU:** `qwen3:8b` kører uden GPU i Docker, så et svar tager 2–5 minutter. Mindre modeller (`qwen2.5:7b`, `llama3.2:3b`) kaldte ikke tools pålideligt og opfandt svar.
- **Parameter-mapping i tool-noderne:** workflowets inputfelter var ikke markeret til at blive udfyldt af modellen. Modellen sendte derfor alle krav som én `query`-tekst, og graph_search kørte uden filtre (se eksempel 2).
- **Eksakte termer i grafen:** grafsøgning kræver de termer, datasættet bruger. Fx hedder kikærter ofte "garbanzo beans" i Food.com.
- **Lille udsnit:** 500 tilfældige opskrifter giver få resultater, når flere krav kombineres.

Se [`examples/examples.md`](examples/examples.md) for konkrete eksempler.
