# I Taught a Private AI to Read My Own Notes: Building a Local RAG

**The assistant was a confident liar about our own systems: fluent, professional, and wrong. So I gave her an open-book exam, entirely on hardware we control. Across 18 real questions, retrieval went from 16 right to all 18. Whether she then uses that note instead of inventing is the harder half, and I don't have that number yet.**

**By Nick Bradford** · November 3, 2026

This was a deployment problem, not a research problem. I had a real assistant, a real pile of internal notes, one hard constraint (it all stays on hardware we control), and one job: make her answer from the truth. Here is what I shipped, what I measured, what it cost, and where the evidence stops.

## The problem: a confident liar

Bella (a local assistant) is sharp on general questions and blind to our own environment. Ask her "how is the photo app set up on our server?" and she returns a fluent, professional answer that is simply invented. It isn't her fault. A language model answers from training, and it has never seen our setup, so it fills the gap with the statistically likely answer instead of the true one. I didn't want a bigger model. I wanted her to look things up before she spoke.

![Bella answering the same question before and after: a confident, invented answer, then a grounded one that cites your own note](images/anim-bella.gif)

## The constraint that drove everything

Privacy was the starting constraint, not a feature bolted on: nothing Bella reads can leave our hardware. That killed every easy option (a hosted vector database, a cloud embedding API, someone else's reranker) before I wrote a line, and forced a local embedder and a self-hosted store.

The constraint has a bonus: it makes the architecture easy to explain to whoever owns the risk. One line around the system, and nothing a model reads crosses it. Which model, which database, which reranker are details you can swap without reopening the question that matters. The person signing off wants to know where the boundary is, what happens when a piece fails, and how you'd find out.

## What RAG is, and how retrieval differs from search

RAG is an open-book exam. Instead of answering from memory, you retrieve the relevant page from your own notes and hand it to the model with the question, so it answers from the page in front of it.

The retrieve step is not the search you're used to. Classic search matches the words you typed, so miss the term the note used and you get nothing. Retrieval matches what you meant, by turning the question and the notes into vectors (lists of numbers) and returning the closest ones. People rarely phrase a question the way the note is written, so matching on meaning is the point. (Keyword search still wins when the query is an exact token, an error code or a filename; a hybrid of the two is on my list to run against the gate, not done yet.)

## The stack, named, all local

- **Vector database:** Qdrant, self-hosted.
- **Embedder:** `nomic-embed-text`, 137M parameters, 274 MB, 768-dimensional vectors.
- **Chat model (Bella):** a local 27B, quantized to 4-bit.
- **Hardware:** the chat model on a single 24 GB GPU; the embedder on an 8 GB GPU.

Each note is chunked, each chunk is embedded once and stored in Qdrant, and a question is embedded the same way so the nearest chunks come back.

![From a note to an answer: notes are turned into meaning once, your question gets the same treatment, and the database hands back the closest notes, all on local machines](images/anim-pipeline.gif)

Chunking is a fixed sliding window, deliberately dumb:

```python
def chunk(text, size=3200, overlap=200):
    step = size - overlap
    return [text[i:i+size] for i in range(0, len(text), step)]
```

Today that's 640 chunks across 229 notes.

## What I turned down

Knowing the options isn't the job; defending a choice is.

- **The embedder.** `nomic-embed-text` over **BGE-M3** (bigger, 1024-dimensional, slower, and on my 18 it ranked the wrong note first a few times) and **MiniLM** (fast, but trained without the document-versus-query distinction, so questions and notes landed in misaligned spaces and I got close-but-wrong hits). Head-to-head numbers for those two are the obvious next ablation.
- **The store.** Qdrant over pgvector (I didn't want retrieval living inside a database I lean on for other things) and over pure keyword search (it breaks the moment someone rephrases). Qdrant also gives me auth and payload metadata to filter on, which is the hook for keeping blocked notes out of results.
- **Chunking.** A fixed window over heading-aware splitting, because the dumb version already maxed the gate.

## How I keep it cheap

Everything local means zero marginal cost per query: no metered API, so re-indexing the whole corpus as often as I like costs nothing but time. The rest follows from that: a 274 MB embedder and a 4-bit chat model fit on hardware that's already running, and the gate lets me keep the cheap, simple version (no reranker, no bigger model) until a number says the complexity earns its keep.

## Lesson 1: the embedder has a secret handshake

My first version worked, mostly. The right note showed up often, not always. Then I read the model card: this family is trained to see a label in front of the text saying what it is. `search_document:` for a stored chunk, `search_query:` for a question. I was feeding it raw text.

```python
def embed(text, kind):                       # kind: "document" or "query"
    prefix = "search_document: " if kind == "document" else "search_query: "
    return embedder.encode(prefix + text)
```

That one line moved hit@1 from 16 of 18 to 17 of 18. An embedding model has a contract, and the defaults are often wrong for your job.

![The prefix handshake: labeling a stored note and a question so the model knows which is which moves hit@1 from 16 of 18 to 17 of 18](images/anim-handshake.gif)

## Lesson 2: tell the chunk where it lives

A chunk cut from the middle of a long note loses its subject. "Run the reset and wait for it to finish." Reset of what? So before embedding, I glue the note's title on the front, while still storing the original chunk:

```python
def embed_chunk(note_title, chunk):
    return embed(f"{note_title}\n\n{chunk}", "document")
```

That moved hit@1 from 17 of 18 to 18 of 18.

![Giving each chunk its note title back before embedding moves hit@1 from 17 of 18 to 18 of 18](images/anim-title.gif)

## The gate, and what the numbers really say

My test set is 18 real questions, each tied to the note that should come back. It's a regression gate, not a benchmark. A script runs all 18 and reports where the right note ranked:

```python
hits = 0
for question, expected_note in gold:              # 18 pairs
    top = search(embed(question, "query"), k=5)    # nearest chunks
    if top[0].note == expected_note:               # right note ranked #1?
        hits += 1
print(f"hit@1: {hits}/18")                         # 18/18 today
```

Now the honest reading, because the percentages flatter it. hit@1 went 16 to 17 to 18 as I added the two fixes. Each step is **one question** moving from rank 2 or 3 up to rank 1. On 18 questions a single flip is about six points, so "89 to 94 to 100 percent" is a prettier way of saying "two more questions ranked first."

And the part that matters most: **hit@3 and hit@5 were 18 of 18 for every configuration, including the baseline.** If I feed the top three chunks to the model, the right note was already there before either fix. The prefix and title tricks improve the ranking, not whether the note is found. So these are the two tricks every RAG guide recommends, and on this corpus they were closer to hygiene than breakthroughs: one line of code each, worth keeping, but they only reordered a top-3 that was already correct. They'd matter more if I fed the model only the top chunk, or where citation order and a tight context budget make rank expensive.

![The regression gate: 18 real questions, the right-note-first count going 16, 17, 18 as each fix lands, though the right note was already in the top 3 at baseline](images/anim-score.gif)

## What I measured, and what I did not

Retrieval is the half I can put a number on: the right note is reliably in front of her. That is not the same as her using it. Whether the answer she writes is faithful to the note, or whether she ignores it and falls back to training, is a different failure, and I don't have a number for it yet.

The honest reason is the privacy rule. Measuring faithfulness usually means an LLM judge, and nothing leaves the box, so no cloud judge. The next step is a local judge plus a deterministic citation check (every claim in her answer must trace to a retrieved line), a held-out set of 30 to 50 questions I did not write into the retriever (including exact-identifier lookups and deliberately odd phrasings), and a batch of unanswerable questions to measure how often she says "I don't know" instead of inventing. Until those numbers exist, the claim stays narrow: I measured retrieval, not faithfulness.

## Latency

A query comes back in about 90 ms, and the split is the story. Over 100 queries, total p50 was 88 ms, p95 was 100 ms, p99 was 112 ms. Of that, the embedding call is roughly 83 ms and the vector search in Qdrant is about 5 ms. The 83 ms is mostly serving overhead, not compute: the embed is an HTTP round-trip through an OpenAI-compatible API to the model host on a second machine, so a 137M model's few-millisecond forward pass is dwarfed by the hop. The database isn't the bottleneck, and neither really is the model; the round-trip is. If I cared, I'd co-locate or batch.

## The failure that still makes me wince, turned into a test

Before any of this, one of our own tools read a real API key out of a doc and quoted it back in an answer. Harmless that time, because it stayed internal. But an AI repeats what you put in front of it, and the next time that line could be in a log or a screenshot.

RAG makes this worse, not better, because the whole point is to feed it all your notes. So I turned the anecdote into a gate check. I plant canary secrets, a fake of every shape (OpenAI, AWS, GitHub, a private-key header, a bearer token, nine in all), into a test note and assert none survive:

```python
redacted, found = scan_text(note_with_canaries)
assert not any(canary in redacted for canary in planted)   # 0 survive
```

Measured: **9 planted, 9 caught, 0 in the output.** That is the ingest layer only, and it's the one hard security number I can show today. Extending the same assertion to the blocked-note filter and the output mask, and adding a prompt-injection note (a document whose text tells the assistant to ignore its rules), are not done yet. I'm not going to claim a result I haven't run.

## Failure modes, and how I'd catch each

The risk owner's real question is what breaks and how you notice.

- **Stale index:** a note changes but its vector doesn't, so she cites an old answer. The indexer re-embeds on file change in about ten seconds; a check compares note timestamps against the last index to flag drift.
- **Orphaned vectors:** a note is deleted but its chunks linger, so she cites something gone. I've hit this for real (an earlier run left roughly 600 stale points). The fix is to reconcile stored vectors against the current note set; I've found the problem but not yet automated the reconcile.
- **Component down:** if the embedder or Qdrant is unreachable, retrieval should fail closed, so she says "I can't look it up right now" rather than guessing from training. That's the intended behaviour; I haven't fault-injected it to prove it holds.

## The limits, because this is one operator on one internal corpus

I should name the ceiling on this evidence. It's one operator, me, on a single internal corpus. I wrote the 18 gold questions, so there's selection bias: I know the notes, and I may have written questions the system can answer. Eighteen is small, and the gate catches regressions, it doesn't prove quality. A fair test needs questions written by someone who didn't build it, a bigger and blinder set, and the faithfulness number I don't have yet. That work is planned, and the eval script plus a redacted sample of the questions will ship so you can run the gate on your own notes.

## The one thing to steal

Build the scorer before the system. The scorer is what turns "I think this is better" into "this is better, and here's the number," and it's the only thing that let me keep the cheap, simple, private version honestly. I can tell you the prefix and title tricks helped because I turned each on and off and watched the count move; without the gate, those are two things I'd merely believe. Privacy turned out not to be a tax either: a 274 MB local embedder and a self-hosted store got the right note in front of her every time, with nothing leaving our control. And the shape isn't personal to me. A private corpus, a hard boundary, and a gate you run before every change is what a clinic or a law office or any team on sensitive internal docs needs. The design doesn't care whose notes they are.
