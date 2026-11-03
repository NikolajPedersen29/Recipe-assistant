# Agent-konfiguration

| Indstilling | Værdi |
|---|---|
| Agent-navn | Kokkeassistenten |
| Rolle | Opskriftsanbefaler / kokkeassistent |
| Workflow | `workflows/agent-kokkeassistenten.json` (Chat Trigger → AI Agent) |
| Chatmodel | `qwen3:8b` via Ollama Chat Model-noden (`http://ollama:11434`) |
| Hukommelse | Simple Memory (window buffer), så agenten husker samtalen |
| Embedding-model | `nomic-embed-text` via Ollama (768 dimensioner) |
| Tools | `vector_search`, `graph_search`, `related_recipes` (Call n8n Workflow Tool) |
| Instruktioner | `agent/instructions.md` (indsat som System Message, med `/no_think` som første linje) |
| Tool-beskrivelser | `agent/tool-descriptions.md` |

## Valg af model

| Model | Resultat |
|---|---|
| `qwen2.5:7b` | Kaldte ingen tools, svarede ud fra egen viden og foreslog fx levering af mad |
| `llama3.2:3b` | For svag til tool calling og dansk |
| `qwen3:8b` | Kalder tools korrekt (testet direkte mod Ollamas `/v1/chat/completions` med en tool-definition). Valgt |

`/no_think` slår qwen3's tænketrin fra og gør svarene hurtigere. Modellen kører på CPU i Docker, så et svar tager 2–5 minutter.

## Agents builder

Opgaven beder om n8n's Agents builder. Agenten blev først bygget der:

- Model: `qwen3:8b` via en OpenAI-credential med Base URL `http://ollama:11434/v1`, fordi builderen ikke har en Ollama-udbyder.
- Samme instruktioner og samme tre workflows tilknyttet som workflow tools, alle publiceret.

Modellen fik dog aldrig workflow-tools at se, kun builderens interne tools. Derfor er den endelige agent bygget med AI Agent-noden i et almindeligt workflow, som bruger de samme retrieval-workflows. Screenshots af Agents builder-opsætningen ligger i `screenshots/`.

Hent modellerne:

```
docker compose exec ollama ollama pull nomic-embed-text
docker compose exec ollama ollama pull qwen3:8b
```
