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

## One concrete future improvement

Add a fourth tool, `record_game_fact(game_record: dict)`, that the agent can
call after a confident web search to write a properly-typed game JSON into
`games/` and re-embed it into the main `udaplay` collection. That closes the
learning loop: every web fallback the agent confidently resolves graduates
into a first-class catalogue entry, and over time the proportion of queries
served by the local (free, fast) vector DB rises towards 100%.
