---
title: Session search by meaning, routed so an exact identifier still wins
status: draft
author: dma9527
created: 2026-10-26
last-audited: 2026-10-26
audited-at: f172b404
doc-pr: null
implementation-prs: []
tracking-issues: [24476]
supersedes: []
superseded-by: []
---

# RFC: Session search by meaning

- Status: draft. Nothing built. Acceptance by a maintainer of session search comes
  before the first implementation pull request (section 10).
- Tracking issue: [#24476](https://github.com/kirodotdev/KiroCrew/issues/24476).
- Depends on [#23603](https://github.com/kirodotdev/KiroCrew/pull/23603) (configurable
  scan window and the `SearchRows` result type, for [#22233](https://github.com/kirodotdev/KiroCrew/issues/22233)).
- Code read at `f172b404`. Code is cited by symbol name; every symbol is as of that commit.
- Numbers come from one operator's host: 614 to 638 session files, 363 distinct
  conversations after twinned copies are folded, mixed English and Chinese. They were
  taken between 2026-10-13 and 2026-10-26 with the shipped `qwen3-embedding-0.6b`
  model through the shipped `llama_cpp` provider.

## 1. Summary

Session search finds a conversation only when the query repeats characters the
conversation contains. This document proposes a second retrieval leg over embedded
transcript chunks, and a routing rule that decides per query which leg answers:

- When the whole query occurs literally in the keyword leg's top conversation, the
  query is an identifier. The result is reciprocal rank fusion of both legs with that
  conversation held first.
- Otherwise the result is the vector ranking.
- With the leg off, unavailable or empty, the result is today's keyword ranking, row
  for row.

On two known-answer query sets this takes first-row recall from 0.139 to 0.806 and
from 0.154 to 0.718, with literal queries no worse than today. Every search surface
gets the change through the one shared ranking, and each surface marks a row found by
meaning so a reader knows why a row whose text lacks the query surfaced.

## 2. Motivation

`_search_sessions` in `history_search.py` is the one ranking every transcript search
shares: the sidebar's Older Sessions search and the command palette (both through
`GET /api/sessions/search`), federated search (`api_instances_search_sessions`),
`search_chat_history`, and the channel session picker in `messaging/session_resume.py`.
It gates on AND over needles from `parse_search_query` and scores occurrence counts
from `count_needle`. A query that shares no term with the conversation it describes
returns nothing, or returns conversations that happen to contain every word.

Measured with the shipped `search_sessions` on 2026-10-26 (section 6 gives the sets):

| Query kind | Queries | Target first | Target in top 10 | Empty result |
|---|---|---|---|---|
| Paraphrase, English and Chinese | 57 | 0 | 5 | 9 |
| Literal identifier | 12 | 9 | 11 | 0 |
| Identifier plus description | 6 | 2 | 2 | 2 |

The figures are at a scan window of 2000, so they measure ranking, not the window
defect #22233 tracks. A person who remembers what a conversation was about, and not
which identifier it used, cannot find it, and an agent calling `search_chat_history`
reads the empty result as "never discussed".

## 3. Current behavior on main

### 3.1 Ranking

`_search_sessions` parses the query, shortlists candidates through the FTS5 index
(`SessionSearchIndex`, a candidate filter and text source, never a ranker), then
scores per needle per field with a title boost (`_TITLE_BOOST`), a phrase bonus
(`_PHRASE_BOOST`) and a recency multiplier bounded by `_RECENCY_MAX_BOOST` and
`_RECENCY_HALF_WEIGHT_DAYS`. Ties break by recency rank.

### 3.2 Snippets

`_extract_history_snippet` in `mcp_core_parts/history.py` returns a window of
`_SNIPPET_RADIUS` characters around the first hit of the best tier from
`snippet_needle_tiers`, which falls back to single words and lone characters, and
marks the hit with `<<<...>>>`. For a paraphrase that anchor is incidental: over the
57 paraphrase queries of section 6.1, run on their own target conversations, 56 got a
snippet and its anchor was most often `the` (7), `a` (6), `in` (3), `，` (3) or
`的` (2). The dashboard rows carry `snippet` only
when one was found. The palette (`sessionsProvider.ts`) shows the snippet as the
subtitle with `substringIndices` highlights, and falls back to the agent name.

### 3.3 Embedding stack

The local embedding model is already loaded for memory. `_try_embed` takes a
priority, and `embeddings_settings.py` defines `PRIORITY_INTERACTIVE` (a human is
blocked: "prompt build, search box"), `PRIORITY_NORMAL` and `PRIORITY_BULK`.
`HybridRetriever` in `knowledge/retrieval.py` fuses legs with `_rrf_fuse`, pins its
vector leg to an `embedder_signature` (an unpinned leg "scores every space against
every query and nothing goes red"), and breaks ties by `updated_at`. Every caller
embeds query text and stored text identically, with no instruction.

## 4. Goals and non-goals

Goals:

1. A conversation is found by describing it, in English or Chinese.
2. A query that repeats an exact identifier puts the conversation naming it first,
   as today.
3. With the leg off or the model absent, every surface returns today's rows in
   today's order.
4. A row found by meaning says so on every surface.

Non-goals:

- Changing `_search_sessions` scoring for a fixed candidate set. A test pins it.
- Indexing the `meta` blob of transcript records (tool input and output), which
  dominates bytes on disk.
- A remote embedding service, or any change to memory recall ranking or its floors.
- A search mode switch in the UI (section 8.1 says why).

## 5. Who searches, and how

The design is checked against the two callers that actually search:

- **A person in the dashboard** types into Older Sessions or the palette. The query
  is debounced 250 ms (`useDebouncedSessionSearch`) and usually describes a subject.
  They open one row and judge it by title and subtitle.
- **An agent** calls `search_chat_history` to recover the exact words of a past
  thread, after memory recall did not answer. Its tool text tells it to "try a query,
  read the snippets, then re-search with different keywords". It reads the snippet
  to decide whether to call `get_chat_session`.

Both read the snippet as the reason a row surfaced, and both read an empty result as
an answer. Section 8 follows from those two facts.

## 6. Measurements

### 6.1 Query sets

Two known-answer sets over 363 conversations. Targets were drawn at random from
conversations with at least 4 chunks.

- Set A, 36 queries: 30 paraphrases (24 English, 6 Chinese) and 6 literal
  identifiers. Written by the author, and the set the routing rule was chosen on.
- Set B, 39 queries: 27 paraphrases (20 English, 7 Chinese), 6 literal, 6 identifier
  plus description. Written by an independent author who saw only the target
  passages, from targets drawn with a different seed excluding set A's. The routing
  rule was fixed before set B was scored.

The sets are built from private transcripts and cannot ship. Section 10 ships the
harness and a synthetic fixture instead.

### 6.2 Recall

Vector leg: the first 8 chunks per conversation, 2279 chunks.

| Method | A R@1 | A MRR | B R@1 | B R@5 | B MRR |
|---|---|---|---|---|---|
| Keyword (today) | 0.139 | 0.161 | 0.154 | 0.282 | 0.204 |
| Vector, bare query | 0.556 | 0.628 | 0.667 | | 0.734 |
| Vector, instructed query | 0.722 | 0.765 | 0.692 | 0.872 | 0.772 |
| RRF of both, keyword top hit held first | 0.333 | 0.489 | 0.231 | 0.718 | 0.448 |
| Routed (section 7.3) | 0.806 | 0.853 | 0.718 | 0.949 | 0.804 |

On set B the router equals the vector leg on all 27 paraphrases and lifts the 6
literal queries from 0.500 to 0.667 at R@1 and from 0.500 to 1.000 at R@5.

### 6.3 Precision

Top-5 pools from three methods over set A's paraphrases were judged blind by two
judges who never saw which method produced a row. The known target was judged
relevant in 27 of 27 pools holding it. P@1: keyword 0/22 (8 queries returned
nothing), vector instructed 27/30, fused with protection 8/30.

### 6.4 Why fusing every query loses

For a natural language query the keyword leg admits many wrong conversations, up to
33 on set A with the target absent from all, and RRF gives them equal standing.
Holding the keyword top hit first then puts a wrong conversation in row one.

### 6.5 Cost

| Quantity | Value |
|---|---|
| Distinct chunks, 300 tokens, whole corpus | 51,005 (87,041 occurrences; 41% repeats from twinned files) |
| Tokens per 1200 characters | 297 Latin, 563 CJK (median) |
| Resident matrix, float32 by 1024 | 209 MB at 51,005 chunks |
| Exact scan, one dot product | 2 ms at 28,753 vectors |
| Query embedding, short query | 200 ms median, 337 ms max of 5 |
| Wait behind one in-flight bulk batch | about 1.3 s |
| First full index at duty 0.2 | 75 to 161 hours wall clock |

## 7. Design

### 7.1 Chunk store

`sessions/.index/session_vectors.db`, beside the FTS index and on its lifecycle:

| Table | Key | Columns |
|---|---|---|
| `chunk` | chunk id | text hash (unique), token count, vector (nullable), embedder signature, vector written at |
| `chunk_ref` | session key, ordinal | message index, char start, char end, chunk id |

- A chunk never spans two messages, so every hit addresses one message.
- Chunks are cut at 300 tokens of the model's tokenizer, which caps every chunk at
  one model cost whatever the script.
- Identical text is stored once and referenced from every session holding it.
  Twinned files can diverge (measured: 35 of 227 pairs hold text only in the copy),
  so dedup is per chunk, and result rows collapse per conversation.
- Rows are written with a null vector and filled at `PRIORITY_BULK`, so the pending
  count is real and the build resumes after a restart.
- A session is indexed once quiet for `_INDEX_QUIET_WINDOW_SECS`, the FTS rule.
- Incognito and temporary sessions are never indexed.

### 7.2 Vector leg

- One resident float32 matrix of the live signature's vectors, scored by one dot
  product. No ANN index at this size; the store stays the source of truth if one is
  added later.
- Rows whose signature differs from the live embedder are excluded, never scored.
- The query is embedded at `PRIORITY_INTERACTIVE` with an instruction prefix, through
  an explicit query role on the embed call. Stored text stays bare.
- Ties break by session modification time, newer first, as `HybridRetriever` does.
  The keyword leg's recency multiplier is not reused: on bounded cosines it turns the
  ranking into a freshness sort (a fresh unrelated session at p90 cosine 0.412 scores
  1.030, above a 30-day-old target at 0.510 scoring 0.893).

The instruction stays on this leg. It lowers every query-to-answer cosine, and memory
recall admits by fixed floors (`SHORT_COSINE_FLOOR` 0.62 in `memory_v2`). On the
shipped lesson bench the median gold cosine fell from 0.639 to 0.556 with it and the
share of gold pairs at or above 0.62 from 0.53 to 0.33.

### 7.3 Routing

Run the keyword leg first. If the case-folded whole query occurs in its top
conversation's folded content, return `_rrf_fuse` of both legs with that conversation
held first. Otherwise return the vector ranking. If the vector leg is off, absent or
empty, return the keyword ranking unchanged.

Pinned by tests:

1. Leg disabled or empty: rows identical to today's, same order.
2. Literal query: the keyword top conversation is row one against a higher-scoring
   vector row.
3. Non-literal query: the vector ranking is returned as it stands.

### 7.4 Result rows

Each row gains `match`, one of `"text"` or `"meaning"`. A `"meaning"` row carries as
`snippet` the matched chunk, cut to `_SNIPPET_RADIUS` around its centre, redacted
like any snippet and without `<<<...>>>` markers. The keyword snippet extractor is
not called for it, since it would centre the row on a stopword (section 3.2). On the
literal route a row in both legs is `"text"`.

The vector leg contributes at most 10 conversations per query. The result rows pass
the same gates as keyword rows: app ownership (`_AppOwnedKeys`), redaction, and the
`keys` restriction, applied before rows collapse.

## 8. User experience

### 8.1 No mode switch

The router decides per query, so a person never chooses between "text" and
"meaning", and a literal query keeps today's first row. A switch would ask the
reader to predict which leg can answer, which is the decision the router already
makes better. The operator control is a config key, `session.semantic_search`.

### 8.2 Per surface

| Surface | Change | Checked against |
|---|---|---|
| Older Sessions (sidebar) | A `"meaning"` row shows the chunk as subtitle and a "Related" label; no highlight, since `substringIndices` finds no query run in it | `HistoryPane.tsx`, backend order kept during search |
| Command palette | Same subtitle and label | `sessionsProvider.ts` sorts title fuzzy matches above backend order. Measured: 0 of 57 paraphrase queries subsequence-match any of 638 titles, so the bias does not reorder meaning rows |
| `search_chat_history` | Each line names its match kind; the snippet is the chunk. Tool text: describe the thread in your own words, or give its identifier; a "meaning" snippet need not contain your words | `mcp_tools/sessions.py`; today a meaning hit would print a snippet marked around a stopword (3.2) |
| Channel picker | Same rows; a text marker on meaning rows | `messaging/session_resume.py` |
| Federated search | Local meaning rows interleave by rank with peer rows; a peer row without `match` reads as `"text"` | `api_instances_search_sessions` interleaves by rank, so no scores are compared across sources |

### 8.3 Partial index

During the first build the leg answers from what is indexed. The result carries the
coverage through the `SearchRows` type #23603 adds, as `semantic: {indexed,
eligible}`, and every surface that shows #23603's scan note adds "Meaning search
covers N of M sessions so far" until they are equal. A reader then knows a missing
hit can still arrive.

### 8.4 Latency

The query embedding adds about 200 ms after the 250 ms debounce. During a bulk build
an interactive embed jumps the queue but waits for the batch in flight, about 1.3 s.
Query embeddings are memoised by text and signature, so refining a query re-embeds
only the new text, and the sidebar's debounce already drops intermediate keystrokes.

### 8.5 Known weak spots

- Identifier plus description: the keyword leg needs every word and often finds
  nothing, and the vector leg can miss the identifier. 1 of 6 such queries on set B
  failed both (target 22nd).
- A query whose answer is not in the corpus still gets up to 10 meaning rows. They
  are labelled, and no floor is set because no measurement supports one (section 11).

## 9. Risks and rollback

| Risk | Mitigation |
|---|---|
| A second path reaches the ownership filter | Rows pass the same gates before collapse; a test returns a foreign app's session from the vector leg and asserts it is dropped |
| Foreign vectors after a model change | Signature pin; the leg answers from the re-embedded share and the coverage note says so |
| 209 MB resident | Matrix loads on first meaning query and is dropped after idle; off by default until phase 4 |
| Week-long first build | Pending rows are durable; coverage note; bulk priority never blocks a search |

Rollback: `session.semantic_search` off restores today's rows row for row (test 1 in
7.3). Deleting `session_vectors.db` removes the data. The query role is used only by
this leg, so it leaves with it.

## 10. Migration plan

| Phase | Ships | Default |
|---|---|---|
| 0 | #23603 (scan window key, `SearchRows`) | merged first |
| 1 | Query role on the embed call; unused by existing callers; recall harness under `src/kiro_crew/eval/bench/` with a synthetic fixture | no behaviour change |
| 2 | Chunk store, indexer, pending and coverage metrics | off |
| 3 | Vector leg, routing, `match`, snippets, labels, tool text, coverage note | off |
| 4 | Default on, after the harness and an operator-run private set show routed R@1 at or above vector R@1 and literal R@1 at or above keyword | on |

Phase 4 changes a default, which is the decision this document asks to record.

## 11. Alternatives considered

| Alternative | Why not |
|---|---|
| Fuse every query | Measured worse than the vector leg alone (6.4) |
| Store chunks as episodic memory | Different retention; on the measured host 51,005 chunks are five times the configured `episodic_max_count` of 10,000, so retirement would evict memory and sessions against each other |
| BM25 in FTS5 | `SessionSearchIndex` rejects it as a ranker: it cannot express per-field substring counts, so every result reorders |
| Remote embeddings | Transcripts leave the host |
| LLM query expansion | A model call per search, and still no match for a paraphrase sharing no term |
| A FAISS file | No row-level staleness, and `faiss` is optional and absent on some installs |
| A Crew app page | Cannot change the search box people already use |

## 12. Open questions

1. Floor for meaning rows. Needs a set of queries with no answer in the corpus;
   until then the cap of 10 and the label stand.
2. Identifier plus description. Route on an identifier token rather than the whole
   query? Untested.
3. Chunks beyond the first 8 per conversation were not measured; the design indexes
   all of them.
4. Should the bulk build run at full duty while the gateway is idle, shortening the
   75 to 161 hours?
