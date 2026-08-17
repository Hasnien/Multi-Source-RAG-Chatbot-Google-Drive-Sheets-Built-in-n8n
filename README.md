# RAG Chatbot — Multi-Source Knowledge Base (n8n)

A retrieval-augmented generation (RAG) chatbot built entirely in n8n, using
a live demo dataset on **Dhaka city road accidents and traffic congestion**.
The bot ingests content from two different source types — a Google Drive
document and a Google Sheet — into a shared vector store, then answers
visitor questions grounded in that data via an AI Agent with conversational
memory.

Built as a portfolio piece to demonstrate a full no-code RAG pipeline:
multi-source ingestion → chunking → embeddings → vector retrieval →
grounded generation with memory.

## Why this project

Most RAG demos ingest a single document type. This one deliberately mixes
**two different content shapes** — a long-form markdown document and a
structured FAQ spreadsheet — to prove the pipeline handles both retrieval
patterns correctly: full-text chunk retrieval vs. row-based Q&A retrieval,
both landing in the same searchable knowledge base.

## Architecture

**Ingestion (two branches, one shared vector store):**

```
Google Drive Trigger → Download → Extract Text ──┐
                                                    ├──▶ Shared Vector Store
Google Sheets Trigger → Format Row (Q + A) ───────┘      (Simple Vector Store,
                                                            same memory key)
```

- The Drive branch pulls a markdown document, extracts its text, and inserts
  it into the vector store (chunked via a text splitter for longer content).
- The Sheets branch reads new FAQ rows, combines each question/answer pair
  into one text block, and inserts it directly — no chunking needed since
  each row is already a self-contained unit.
- Both branches embed with the same model (Google Gemini embeddings) and
  write to the same vector store memory key, so retrieval later searches
  across both sources at once.

**Chat / retrieval:**

```
Chat Trigger → AI Agent ──┬── Vector Store (retrieve-as-tool)
                            ├── Chat Model (Google Gemini)
                            └── Conversation Memory (buffer window)
```

The AI Agent decides when to call the vector store as a tool based on the
user's question, retrieves the most relevant chunks (from either source),
and answers grounded in that context — with memory so it handles multi-turn
conversations.

## Key design decisions

- **Multi-source, single retrieval surface.** Two very different ingestion
  paths converge on one vector store, so the chat layer doesn't need to know
  or care where an answer's source content came from.
- **`Clear Store` set to off on both insert nodes.** Since both branches
  share one memory key, leaving the default "clear on insert" behavior on
  would let one branch's ingestion run wipe out the other's data.
- **Per-source chunking strategy.** The long-form document is split into
  overlapping chunks (better for prose); FAQ rows are inserted whole
  (splitting would separate a question from its answer, hurting retrieval).
- **Grounded, not generative.** The agent's tool description and system
  behavior are scoped to answer from retrieved context, not to freely
  generate facts about Dhaka traffic — important for a data-accuracy-
  sensitive demo topic like accident statistics.

## Setup

1. Import `rag-chatbot-workflow.json` into your n8n instance.
2. Add credentials for: Google Drive OAuth2, Google Sheets OAuth2, and
   Google Gemini (PaLM) API — used for both embeddings and chat generation.
3. Replace the placeholder folder ID (Google Drive Trigger) and spreadsheet
   ID (Google Sheets Trigger) with your own.
4. On both `Simple Vector Store` insert nodes, confirm they share the same
   Memory Key, and that **Clear Store is turned off** on both.
5. Run each ingestion branch once (manually, or by dropping a file / adding
   a row) to populate the vector store.
6. Open the Chat Trigger's test chat panel and start asking questions.

## Demo dataset

The demo content — a markdown document and a 14-row FAQ sheet covering
Dhaka's road accident statistics and traffic congestion data — is included
in [`/demo-data`](./demo-data) so you can reproduce the exact setup.

## Limitations (by design, for a demo)

- Uses n8n's **Simple Vector Store**, which is in-memory only and resets on
  restart — intentional for a lightweight local demo. A production version
  would swap this for Pinecone, Supabase, MongoDB Atlas, or pgvector without
  changing anything else in the workflow (the vector store node is the only
  thing that would need to change).
- No re-embedding on document *edits*, only on new file/row creation —
  a "File Updated" trigger branch could be added for that.

## Stack

n8n · Google Drive API · Google Sheets API · Google Gemini (embeddings +
chat model) · n8n Simple Vector Store (LangChain in-memory)
