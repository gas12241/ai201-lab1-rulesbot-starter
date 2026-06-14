# Spec: `retrieve()`

**File:** `retriever.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user's natural language query, find the most relevant chunks from the vector store using semantic similarity search. Return them ranked by relevance so that `generate_response()` can use them as context.

---

## Input / Output Contract

**Inputs:**

| Parameter   | Type  | Description                                                                |
| ----------- | ----- | -------------------------------------------------------------------------- |
| `query`     | `str` | The user's natural language question                                       |
| `n_results` | `int` | Maximum number of chunks to return (default: `N_RESULTS` from `config.py`) |

**Output:** `list[dict]`

Each dict in the returned list must contain exactly these keys:

| Key          | Type    | Description                                                   |
| ------------ | ------- | ------------------------------------------------------------- |
| `"text"`     | `str`   | The chunk text                                                |
| `"game"`     | `str`   | The game name this chunk came from                            |
| `"distance"` | `float` | Cosine distance score — lower means more similar to the query |

Results should be ordered from most to least relevant (lowest to highest distance). Returns an empty list `[]` if the collection contains no documents.

---

## Design Decisions

_Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours._

---

### Query approach

_Describe how you will use `_collection.query()` to find relevant chunks. What arguments will you pass, and why?_

```
We call _collection.query() with three arguments:

    results = _collection.query(
        query_texts=[query],
        n_results=n_results,
        include=["documents", "metadatas", "distances"]
    )

Why each argument:

- query_texts=[query]
  Wraps the single query string in a list because the API expects a batch.
  ChromaDB handles embedding it internally using the same embedding model
  (all-MiniLM-L6-v2 from config.py) that was used when storing the chunks,
  so the vectors are in the same space and distances are meaningful.

- n_results=n_results
  Caps how many chunks come back. The caller passes this in (defaulting to
  N_RESULTS = 3 from config.py). Keeping it small limits how much context
  gets passed to the LLM downstream in generate_response().

- include=["documents", "metadatas", "distances"]
  Requests exactly the three fields we need to build our return dicts:
    - "documents"  → the chunk text  ("text" key)
    - "metadatas"  → the game name   ("game" key)
    - "distances"  → the similarity  ("distance" key)
  Without this argument ChromaDB omits distances by default, so we must
  ask for it explicitly.
```

---

### Return structure

_Sketch out what one item in your return list looks like as a concrete example. Where does each field come from in the query results?_

```
One item in the return list looks like this:

{
    "text":     "A piece is captured when it is directly attacked by an opposing piece...",
    "game":     "Chess",
    "distance": 0.142
}

Where each field comes from in the raw query results (assuming results = _collection.query(...)):

- "text"     ← results["documents"][0][i]
              The chunk text stored in ChromaDB at position i.

- "game"     ← results["metadatas"][0][i]["game"]
              The "game" key from the metadata dict stored alongside each chunk at position i.

- "distance" ← results["distances"][0][i]
              The cosine distance score ChromaDB computed between the query embedding
              and the chunk embedding at position i.

The [0] index strips the batch wrapper (ChromaDB nests results in an outer list because
query_texts can accept multiple queries at once). The [i] index iterates over each result
in that single query's result list.
```

---

### Handling the nested result structure

_`_collection.query()` returns nested lists. Describe what index you need to access to get the actual list of results for a single query, and why the nesting exists._

```
To get the actual list of results for a single query, access index [0] on each
key in the returned dict:

    results["documents"][0]   # list of chunk texts
    results["metadatas"][0]   # list of metadata dicts
    results["distances"][0]   # list of distance scores

The nesting exists because _collection.query() is designed for batch querying —
query_texts accepts a list of query strings so you can run multiple queries in
one call. The outer list always has one entry per query string passed in.

Since we only ever pass one query string at a time (query_texts=[query]), the
outer list always has exactly one element, and [0] extracts the results for
that single query.

Example of what the full nested structure looks like for one query:

    results["documents"]  →  [["chunk text A", "chunk text B", "chunk text C"]]
                                ^                                               ^
                                outer list (one per query string)
                                  ^                               ^
                                  inner list (one per result)

    results["documents"][0]  →  ["chunk text A", "chunk text B", "chunk text C"]
    results["documents"][0][0]  →  "chunk text A"   ← top result
```

---

### Relevance threshold

_Will you filter out results above a certain distance score, or return all `n_results` regardless of how relevant they are? What are the tradeoffs of each approach?_

```
From my understanding, if you were to keep all results
(regardless of how relevant they are), it would dilute the
information that is relevant to the query. Because of that,
it is usually advisable to filter out results above a certain
distance score. The tradeoff for filtering up to n_results is
that you might cut out some information that might be necessary
for the response, so balancing the amount of results to not dilute
the information for the response while keeping all the pertinent
information available is the key.
```

---

### Edge cases

_How does your implementation behave when: (a) the collection is empty, (b) the query matches no chunks well, (c) the query matches chunks from multiple games?_

```
[your answer here]
```

---

## Implementation Notes

_Fill this in after implementing, before moving to Milestone 3._

**Test query and top result returned:**

```
Query: How do you set up the board in Catan?
Top result game: Catan
Distance score: .581
Does it make sense? The game choice makes sense. I'm not entirely sure how to
view the whole text in my terminal (as it gets cut off), so I'm not entirely
sure if the chunks received can answer the query asked.
```

**One thing about the query results that surprised you:**

```
I had hoped that the distance would be smaller. I learned that a higher number
means that the chunk is less connected to the query, but I was thinking that
the number would be lower than .5.
```
