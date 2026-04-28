# UdaPlay — Reflection

## Strengths

- **Two-tier retrieval that respects cost.** The agent always tries the local
  vector DB first and only escalates to the Tavily web API after an LLM judge
  flags the local hits as insufficient. This keeps the average query cheap
  while still answering questions that fall outside the catalogue.
- **Reusable, well-typed primitives.** The `lib/` package provides an `Agent`,
  `LLM` wrapper, `@tool` decorator, state machine, and short-term memory.
  The notebook code is therefore short and focused on the project-specific
  pieces (system prompt, three tools, three demo queries, stand-out
  features). New tools can be added by writing a single decorated function.
- **Structured answers in addition to prose.** Every final answer is
  re-emitted as an `Answer` Pydantic model with citations, confidence, and
  source provenance — usable by downstream consumers (dashboards, APIs)
  without further parsing.
- **Long-term memory cache.** Successful web-search snippets are persisted
  into a separate `udaplay_facts` Chroma collection so a repeated question
  in a later session can be answered from cache without paying Tavily again.

## Limitations

- **Judge LLM is the same model family as the worker.** `evaluate_retrieval`
  uses `gpt-4o-mini`, the same model the agent itself uses. A truly
  independent judge (e.g. a different provider) would catch more failure
  modes, especially overconfident hallucinations.
- **No cross-validation between vector DB and web.** When both sources return
  facts, the agent picks the vector DB and stops. A more rigorous design
  would compare them and surface conflicts.
- **Static catalogue.** New games have to be added by dropping JSON files
  into `games/` and re-running Part 1. The agent has no path to grow its
  internal catalogue from learnings yet (the `udaplay_facts` cache is web
  snippets, not curated game records).

## v2 extension — closed-loop learning (implemented)

The original "one concrete future improvement" was a fourth tool,
`record_game_fact`, that closes the learning loop by turning confident web
findings into permanent catalogue entries. **This is now implemented and
demonstrated.**

The v2 agent is registered with four tools instead of three:
`retrieve_game`, `evaluate_retrieval`, `game_web_search`, **`record_game_fact`**.
The system prompt instructs the agent to call `record_game_fact` exactly once
after a confident `game_web_search` answer about a specific game that is not
yet in the catalogue. The tool validates the record against the
`GameRecord` Pydantic schema, writes a new `games/NNN.json` file with the
next sequential id, and adds the record to the live `udaplay` Chroma
collection.

Demo evidence (Notebook 2, cells 19, 23, 25): a fifth query
("Tell me about Astro Bot, the 2024 PlayStation 5 platformer") triggers the
full closed-loop sequence — retrieve → evaluate (useful=false) → web search
→ `record_game_fact` (status=ok, id=021, collection_count=21) — and a sixth
query ("When was Astro Bot released, and on which platform?") is answered
**locally** out of the just-extended catalogue, with no web search.

## One new improvement to pursue next

Add a periodic **catalogue-deduplication / re-embedding pass**. Right now
`record_game_fact` only refuses ids that already exist on disk, but it does
not check whether a near-duplicate game (different platform, slightly
different name) is already in the collection. A scheduled job that:

  1. Clusters the catalogue's embeddings,
  2. Flags pairs whose cosine similarity is above a learnt threshold,
  3. Asks an LLM judge to merge or distinguish the candidates,

would keep the catalogue coherent as the agent grows it autonomously.
