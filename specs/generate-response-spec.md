# Spec: `generate_response()`

**File:** `generator.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user query and a list of retrieved rule chunks, generate a response that directly answers the question using only the retrieved text as context. The response must be grounded — it should not draw on the model's general knowledge of board games, only on what was retrieved.

---

## Input / Output Contract

**Inputs:**

| Parameter          | Type         | Description                                                                             |
| ------------------ | ------------ | --------------------------------------------------------------------------------------- |
| `query`            | `str`        | The user's original question                                                            |
| `retrieved_chunks` | `list[dict]` | Ranked list of chunks from `retrieve()`, each with `"text"`, `"game"`, and `"distance"` |

**Output:** `str`

A plain string containing the response to show the user. The response should:

- Answer the question using only the retrieved rule text
- Identify which game the answer comes from
- Acknowledge clearly when the answer is not found in the loaded rules

Returns a fallback string (not an error) when `retrieved_chunks` is empty.

---

## Design Decisions

_Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours._

---

### Context formatting

_How will you format the retrieved chunks before passing them to the LLM? Describe the structure — not the code. Consider: will you label chunks by game? Include distance scores? Separate chunks with delimiters?_

```
Each chunk will be formatted as a labeled block, separated by a blank line:

    [Game: Chess]
    A piece is captured when it is directly attacked by an opposing piece...

    [Game: Chess]
    The en passant rule applies only immediately after the opponent moves a pawn...

    [Game: Checkers]
    A piece must capture if a capture move is available...

Key decisions and why:

1. Label each chunk with [Game: <name>]
   Research consistently shows LLMs distinguish sources more reliably when
   each block is explicitly labeled. Without labels, the model may blend rules
   from different games or fail to cite correctly.

2. No distance scores included
   Distance scores are internal metadata — they'd add noise to the context
   and encourage the model to reason about retrieval quality rather than
   content. Filtering by threshold (handled separately) is a better place
   for that logic.

3. Chunks separated by blank lines, not XML tags or numbered lists
   Blank-line separation is clean and sufficient. XML tags (<source>) work
   well too and some research prefers them, but they add verbosity for this
   use case. Numbered lists imply ranking/priority, which isn't what we want.

4. Raw chunk text, no truncation
   Chunks are already short (from the chunking step), so we pass the full
   text of each one. Truncating here would discard exactly the detail the
   model needs to answer correctly.
```

---

### System prompt — grounding instruction

_Write the exact system prompt instruction you will use to prevent the model from answering beyond the retrieved text. This is the most important design decision in this function._

```
Exact system prompt grounding instruction:

    You are a board game rules assistant. Answer the user's question using
    ONLY the rule text provided below. Do not use your general knowledge of
    board games, and do not infer or extrapolate beyond what is explicitly
    stated in the provided text. If the answer is not contained in the
    provided rules, say so clearly — do not guess.
```

---

### System prompt — citation instruction

_Write the exact instruction you will use to tell the model to identify which game its answer comes from._

```
For the chunks, Every chunk of information has a label of the game it comes
from. This makes it so that the LLM has an idea of what game the instruction
comes from.
```

---

### Fallback behavior

_What should the response say when the answer isn't found in the loaded rule books? Write the exact fallback message._

```
What it should say is that the LLM hasn't been trained on that information.
For example, I asked about the board game "Chutes and Ladders," and it told
me that it doesn't have information on that game, but did inform me that if
I had any questions about Clue and Risk, to let it know.
```

---

### Handling low-relevance chunks

_`retrieved_chunks` may include chunks with high distance scores (weak relevance). Will you filter these out before building context, pass them all in, or handle them another way? What are the tradeoffs?_

```
I don't filter out any chunks at all, but rather return the most relevant.
This would mean that if no chunks are super relevant, there is a chance
that the response from the LLM doesn't match the question that is asked.
```

---

### Message structure

_Describe how you will structure the messages list for the API call — what goes in the system message vs. the user message?_

```
The system message that is inputted will tell the LLM to only use what it
was trained on as a reference for the answer. The user message is sent
as a user_message alongside the chunks.
```

---

## Implementation Notes

_Fill this in after implementing and testing._

**Test query and response:**

```
Query: Can I trade in Catan
Response: According to the official rules summary of the game Catan, yes, you
can trade in Catan. The rules state that "You may trade with other players at
any mutually agreed ratio — this is negotiated freely during your turn."
Additionally, the rules describe harbor trades, which allow you to trade
resources at specific ratios, such as 3:1 or 2:1, depending on the type of
harbor.
Correctly grounded? Yes
Cited the right game? Yes
```

**One thing you changed from your original spec after seeing the actual output:**

```
I did not change anything from my original spec after seeing the output.
It worked as intended.
```
