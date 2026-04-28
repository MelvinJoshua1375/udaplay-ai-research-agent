# UdaPlay — AI Research Agent for the Video Game Industry

UdaPlay is a small AI research agent built for a fictional gaming-analytics
company. It answers natural-language questions about video games (titles,
platforms, release dates, genres, publishers) using a **two-tier retrieval
system**: a local ChromaDB vector store of curated game records, with a
Tavily web search as a fallback when the local catalogue is insufficient.

This is the final project of an Udacity course on Agentic AI.

---

## What's inside

```
.
├── Udaplay_01_solution_project.ipynb   # Part 1: ChromaDB RAG over games/
├── Udaplay_02_solution_project.ipynb   # Part 2: stateful agent + 3 tools + demo queries + stand-out features
├── lib/                                # reusable agent infrastructure (Agent, LLM, @tool, state machine, memory, vector_db)
├── games/                              # 20 game records (15 starter + 5 stand-out additions)
├── outputs/                            # rubric evidence: terminal logs + performance chart
├── reflection.md                       # rubric stand-out: strengths / limits / one improvement
├── requirements.txt
├── .env.example
└── .gitignore
```

The `lib/` folder is reused as-is from the course materials; the project's
student-written code lives entirely in the two solution notebooks.

---

## Architecture at a glance

**Part 1 — Offline RAG.** Reads every JSON file under `games/`, embeds the
content with OpenAI's `text-embedding-3-small`, and stores the result in a
persistent ChromaDB collection called `udaplay`. The notebook ends with a
semantic-search demo against the collection.

**Part 2 — Agent.** Re-opens the same persistent collection and wraps it
behind four `@tool` functions:

| Tool | Purpose |
|---|---|
| `retrieve_game(query)` | Top-5 semantic search over the local catalogue. |
| `evaluate_retrieval(question, retrieved_docs)` | LLM judge that returns `{useful, description}` to gate the next step. |
| `game_web_search(question)` | Tavily fallback; returns top-3 web snippets. |
| `record_game_fact(...)` *(v2 extension)* | After a confident web answer about a game not yet in the catalogue, validates the record against `GameRecord` and writes it to `games/NNN.json` plus the live Chroma collection — closing the learning loop. |

The `Agent` class from `lib/agents.py` runs an `LLM ↔ tools` state machine:
on each turn it sends the conversation to the model, executes any tool calls
the model emits, feeds the results back as `ToolMessage`s, and loops until
the model decides no further tool calls are needed. The system prompt
explicitly directs the agent to call `retrieve_game` first, gate on
`evaluate_retrieval`, and only then fall back to `game_web_search`.

The notebook then runs the agent on six demo questions:

  1. Pokémon Gold/Silver release year (vector-DB only).
  2. First 3D Mario platformer (vector-DB only).
  3. Mortal Kombat X on PS5? (vector-DB → judge says insufficient → web).
  4. *"Of the three games we just discussed, which one was released earliest?"*
     — answered from prior turns alone, with **zero tool calls**, proving
     session-level statefulness.
  5. *"Tell me about Astro Bot, the 2024 PlayStation 5 platformer."* —
     vector-DB miss → web search → `record_game_fact` writes Astro Bot into
     `games/021.json` and into the live Chroma collection (v2 closed-loop).
  6. *"When was Astro Bot released, and on which platform?"* — now answered
     **locally** from the just-extended catalogue, with no web search.

…and emits four stand-out artefacts:

1. five extra game records (`games/016.json … 020.json`),
2. a `udaplay_facts` Chroma collection that caches successful web-search
   snippets across sessions (long-term memory),
3. a structured `Answer` Pydantic object per query (`final_text`,
   `citations`, `confidence`, `sources_used`),
4. a `outputs/agent_performance.png` matplotlib bar chart of tokens used and
   tool calls per query.

---

## Setup

```bash
git clone https://github.com/MelvinJoshua1375/udaplay-ai-research-agent.git
cd udaplay-ai-research-agent

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# edit .env and fill in:
#   OPENAI_API_KEY="sk-..."
#   TAVILY_API_KEY="tvly-..."
```

**API keys.** The notebooks load both keys via `python-dotenv`. Get an
OpenAI key at <https://platform.openai.com/api-keys> and a free Tavily key
(1000 requests/month) at <https://app.tavily.com>.

---

## How to run

The notebooks must be run **in order** — Part 2 reuses the persistent
ChromaDB collection that Part 1 builds.

```bash
jupyter lab    # or: jupyter notebook
```

1. Open `Udaplay_01_solution_project.ipynb`. Run every cell from top to
   bottom. The last two cells demonstrate semantic search and verify that
   the persistent client survives a fresh open.
2. Open `Udaplay_02_solution_project.ipynb`. Run every cell from top to
   bottom. The notebook will:
   - re-open the `udaplay` collection,
   - register the three tools,
   - instantiate the agent,
   - invoke it on the three rubric demo queries,
   - emit structured `Answer` objects, populate the `udaplay_facts` cache,
     save the matplotlib chart to `outputs/agent_performance.png`, and
     print a final consolidated report.

Expect Part 2 to take roughly 1–2 minutes against `gpt-4o-mini`.

---

## Rubric coverage

| Rubric line | Where it's satisfied |
|---|---|
| **RAG**: load and process the local game data into a persistent vector DB with embeddings | Notebook 1, cells under "VectorDB Instance" / "Collection" / "Add documents" |
| **RAG**: notebook demonstrates the vector DB can be queried for semantic search | Notebook 1, "Demonstrate semantic search" cell |
| **Agent**: ≥3 tools, each integrated as a function, decorated, and registered | Notebook 2, `retrieve_game`, `evaluate_retrieval`, `game_web_search`, plus `record_game_fact` (v2) |
| **Agent**: first answers with internal knowledge, evaluates, falls back to web | System prompt + agent loop in `Agent` instantiation cell |
| **Agent**: stateful agent class managing conversation state and tool usage | Reuses `lib.agents.Agent`, which uses `lib.state_machine.StateMachine` and `lib.memory.ShortTermMemory` |
| **Agent**: report performance with example queries; output includes reasoning, tool usage, final answer, and citations | Notebook 2 "Invoke" cell + final-report cell + `outputs/output_part2_agent.txt` |
| **Stand-out**: extra games, long-term memory, structured output, visualisation | Stand-out feature cells in Notebook 2 + `games/016–020.json` |
| **Stand-out**: reflection | `reflection.md` |

---

## Evidence files

The `outputs/` folder contains terminal-style logs of the notebooks running.
Use them as rubric evidence without having to re-run the notebooks against
real OpenAI / Tavily APIs:

- `output_part1_rag.txt` — Part 1 ingestion + semantic-search demo.
- `output_part2_agent.txt` — Part 2 agent traces for all three demo queries.
- `output_standout.txt` — long-term memory cache hit + structured `Answer`
  objects.
- `agent_performance.png` — matplotlib chart of tokens / tool-calls.

---

## License

Course materials remain the property of Udacity; the student-authored
notebooks and `reflection.md` are provided as a learning artefact.
