---
layout: default
title: "RAG Doesn't Fail at the AI Layer. It Fails at the Data Layer."
date: 2026-03-21 11:46:00 +0530
categories: engineering ai llm machine-learning rag
---

Most teams I've talked to approach RAG the same way: pick an embedding model, stand up a vector database, wire it to an LLM, and ship a demo that impresses. Two months later, users have stopped trusting it.

The retrieval was bad. Not because the AI was wrong — because the underlying data was a mess they never had to reckon with before.

That's the uncomfortable truth about RAG: it doesn't expose your AI maturity. It exposes your data maturity. And most organizations aren't ready for that mirror.

This post isn't another explainer on how RAG works. It's about what actually breaks in production — and what separates systems people trust from systems people route around.

---

## First, The Mental Model Worth Keeping

Before the hard lessons, a quick frame for anyone newer to the pattern.

**Retrieval Augmented Generation** solves a real limitation: LLMs are trained on general knowledge up to a cutoff date, with no access to your internal data, and a tendency to confabulate when they hit the edges of what they know. Fine-tuning patches some of this, but it's expensive, slow to update, and overkill for most use cases.

RAG offers a cleaner path — give the model an open-book exam instead of testing its memory:

1. A user asks a question
2. Your system retrieves relevant chunks from a knowledge base
3. Those chunks are injected into the prompt as context
4. The model generates an answer grounded in what you retrieved

Simple in principle. The pipeline looks like this:

```
[User Query] → [Embed Query] → [Vector Search] → [Retrieve Chunks] → [Augmented Prompt] → [LLM] → [Answer]
```

And before you can retrieve, you need to index:

```
[Raw Documents] → [Chunking] → [Embedding] → [Vector Store]
```

The tech stack is mature. Pinecone, Weaviate, pgvector for storage. OpenAI or Cohere embeddings. LangChain or LlamaIndex for orchestration. You can have a working prototype in an afternoon.

That's also the problem.

---

## Lesson 1: Garbage In, Garbage Out — But Worse Than You Think

Classic data quality problems get *amplified* in RAG systems.

When a user searches traditional enterprise software and gets a stale result, they see a document and can judge its age. When a RAG system retrieves that same stale document and synthesizes it into a confident, fluent answer — they believe it. The LLM is too good at sounding authoritative.

The failure modes I've seen most often:

- **Outdated policy documents** still indexed because no one set expiration logic
- **Duplicate content** from migrations causing contradictory retrievals — the LLM picks one and sounds certain
- **Low-signal content** like meeting notes or draft docs indexed alongside authoritative sources, poisoning retrieval quality
- **Untitled or poorly named files** that embed as noise — semantic search can only work with semantic content

The fix isn't more sophisticated AI. It's data governance: source tagging, freshness metadata, tiered content quality, and deprecation pipelines. Boring infrastructure work that teams skip because it doesn't feel like "building AI."

**The question to ask before indexing anything**: *Would I be comfortable if the model cited this document in a customer-facing response?*

---

## Lesson 2: Chunking Is a Design Decision, Not a Default

Most tutorials say "split into 512-token chunks with 10% overlap" and move on. This is fine for demos. In production, it's often the root cause of poor retrieval.

The problem: chunks are retrieved in isolation, but meaning lives in context. A policy document that says "exceptions apply in cases outlined in Section 4.2" means nothing if Section 4.2 is in a different chunk — or not retrieved at all.

Chunking strategies worth considering:

- **Semantic chunking**: Split on paragraph and section boundaries, not arbitrary token counts. Respects the author's intent.
- **Hierarchical indexing**: Index both summaries and full chunks. Retrieve summaries first to find relevance, then fetch the full chunk. Better precision, lower token cost.
- **Document-aware chunking**: Treat structured documents (PDFs, wikis, code) differently. A 500-token chunk of a Python file might contain three unrelated functions. A 500-token chunk of a legal contract might split a clause mid-sentence.

The right chunking strategy depends on your content type. The wrong chunking strategy is one size fits all.

---

## Lesson 3: Vector Search Is Powerful — And Not Enough

Semantic similarity search is what makes RAG feel magical. But pure vector search has a blind spot: it's bad at exact matches.

Ask a system "what is the reimbursement limit for business travel?" and vector search will find semantically similar content about travel policy. But if the answer is a specific dollar figure buried in a table — a thing that doesn't "mean" much out of context — it might miss entirely.

Hybrid search — combining vector similarity with BM25 keyword matching — closes most of this gap. It's not a premature optimization; it's table stakes for production systems.

Beyond that:

- **Metadata filtering** before retrieval (by recency, department, document type) cuts noise dramatically
- **Reranking** with a cross-encoder model as a second pass reorders results by actual relevance, not just vector proximity
- **Multi-query expansion** generates 3-5 reformulations of the original question before retrieval — catches synonyms, phrasing variants, and angle differences the user didn't think to try

The retrieval step is where most teams underinvest. It's less exciting than prompt engineering, but it has more leverage on answer quality.

---

## Lesson 4: Evaluate Retrieval Separately From Generation

This is the one that bites teams hardest at scale.

It's easy to eyeball whether an LLM answer sounds good. It's harder to audit whether the *retrieval* was good — especially when the LLM can write a coherent answer from mediocre context, masking the problem.

You need to evaluate two things independently:

**Retrieval quality**: For a given query, were the right documents retrieved? You want metrics like recall@k (did the correct document appear in the top-k results?) and MRR (how highly was it ranked?).

**Answer quality**: Given retrieved context, did the model answer correctly? RAGAS and TruLens both provide frameworks for this — measuring faithfulness (does the answer stay grounded in context?) and answer relevance separately.

Without this split, you'll optimize the wrong layer. Teams chase prompt engineering improvements when the real problem is a retrieval precision issue — or vice versa.

---

## Lesson 5: The Hard Part Is Change, Not Construction

RAG systems built on living data have a maintenance burden most teams underestimate.

Your knowledge base changes. Policies update. Products change. Old documents get superseded. Each of those events needs to propagate into your vector store — which means re-chunking and re-embedding affected documents, updating metadata, and potentially invalidating cached queries.

Teams that treat the index as "built once" end up with systems that degrade slowly and invisibly. Users stop trusting answers. They go back to Ctrl+F.

Design for change from the start:

- Treat your vector index as a derived artifact, not a source of truth — maintain the pipeline that recreates it
- Use document IDs and version hashes to detect and process updates
- Build deletion and expiration logic before you need it
- Monitor retrieval quality over time — drift is real

---

## When RAG Is the Wrong Tool

RAG gets oversold. It's not the answer to every knowledge problem.

If your knowledge base is small and stable, just put it in the system prompt. Modern context windows are large enough that a 50-page policy manual fits comfortably — no retrieval pipeline, no maintenance overhead, no retrieval failures.

If your queries are highly structured — "show me all invoices over $10K from Q3" — a SQL query over a relational database will outperform RAG on accuracy, speed, and auditability. LLM-to-SQL has its own challenges, but it's the right abstraction for structured data.

If you need real-time operational data, retrieval latency may be unacceptable. Direct API calls to the source of truth are often a better fit.

The right question isn't "can we use RAG for this?" It's "does retrieval over a document corpus match the shape of this problem?"

---

## The Actual Stack

For teams building this seriously, here's what a production setup typically involves:

| Layer | Options |
|---|---|
| **Embedding** | OpenAI text-embedding-3-large, Cohere embed-v3, Sentence Transformers |
| **Vector DB** | Pinecone, Weaviate, Qdrant, pgvector |
| **Keyword Search** | Elasticsearch, OpenSearch, or BM25 via Weaviate hybrid mode |
| **Reranking** | Cohere Rerank, cross-encoder models via Sentence Transformers |
| **Orchestration** | LangChain, LlamaIndex, Haystack |
| **Evaluation** | RAGAS, TruLens |
| **LLM** | GPT-4o, Claude Sonnet, Llama 3, Mistral |

---

## The Bottom Line

RAG is the right pattern for most LLM applications that need to work with private, evolving, or domain-specific knowledge. The core idea — retrieve, augment, generate — is simple and powerful.

But the teams that ship trustworthy systems are the ones that treat it as a data engineering problem first and an AI problem second. They invest in content quality, chunking strategy, hybrid retrieval, and ongoing evaluation. They design for change, not just for launch.

The teams that struggle are the ones that got the demo working in an afternoon and thought they were done.

---

*Building RAG in production? I'm especially curious what's broken for you at the retrieval layer — that's usually where the interesting problems live. Reach out to the CS360 engineering team or drop a comment below.*