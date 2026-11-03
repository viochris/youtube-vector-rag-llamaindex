# 🎬 YouTube Vector RAG Chatbot (LlamaIndex)

A CLI chatbot that lets you chat with a YouTube video through its transcript. It fetches the captions once, splits and embeds them into ChromaDB Cloud, retrieves and reranks the most relevant passages for every question, and is instructed to answer strictly from what the video says, while remembering the conversation across runs in PostgreSQL. This is the plain **Vector RAG** version, where context is found only by similarity search.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat&logo=python&logoColor=white)
![LlamaIndex](https://img.shields.io/badge/Framework-LlamaIndex-6E56CF?style=flat)
![ChromaDB](https://img.shields.io/badge/VectorDB-ChromaDB%20Cloud-FF6F00?style=flat)
![Gemini](https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?style=flat&logo=googlegemini&logoColor=white)
![LiteLLM](https://img.shields.io/badge/Routing-LiteLLM-black?style=flat)
![Reranker](https://img.shields.io/badge/Reranker-Cross--Encoder-yellowgreen?style=flat)
![PostgreSQL](https://img.shields.io/badge/Memory-PostgreSQL%20(AsyncPG)-336791?style=flat&logo=postgresql&logoColor=white)
![Phoenix](https://img.shields.io/badge/Tracing-Arize%20Phoenix-purple?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

## 📌 Overview

This repository contains one CLI script, `Vector-self-make-youtube-reader-cmd.py`, that turns a YouTube video into something you can ask questions. Point it at a list of video URLs and it downloads the transcript, splits it into overlapping chunks, embeds the chunks with Google's Gemini embedding model, stores the vectors in ChromaDB Cloud, and opens an interactive chat loop where answers are built from the retrieved transcript passages.

The transcript loader is written inside the script itself (`load_youtube_transcripts`). It reads the video id from the URL, calls `youtube-transcript-api`, joins the caption snippets into one text, and wraps it in a LlamaIndex `Document` together with the video id and the URL as metadata.

Scraping and embedding only happen when the ChromaDB collection is empty. On every later run, the script rebuilds the index straight from the vector store, so nothing is downloaded or embedded again. The index is then wired into a context-mode chat engine with cross-encoder reranking and PostgreSQL-backed memory.

The answers are meant to come from the retrieved context and not from the model's own memory. The system prompt tells the model to answer strictly from the retrieved transcript context, and to say so when the answer is not there. After each answer, the script prints the source passages, the token usage, and the estimated cost of that turn.

This repository is the Vector version of a three-architecture comparison (Vector, Graph, and Hybrid RAG). See [Three RAG Architectures](#-three-rag-architectures) for how they differ.

Here is what happens when you run the script:

1. Load `.env` and stop with a plain error message if `GOOGLE_API_KEY` or `PHOENIX_API_KEY` is missing. This check runs before the error handling in step 10 starts.
2. Configure the LLM (Gemini through LiteLLM), the embedding model, the chunking, the token counter, and the debug handler.
3. Register Phoenix tracing, so every run is recorded as a trace.
4. Create the cross-encoder reranker.
5. Connect to ChromaDB Cloud and open the `youtube_docs` collection.
6. If the collection is empty, fetch the transcripts, chunk and embed them, and store the vectors. Otherwise, load the index straight from the vector store.
7. Create the PostgreSQL memory and the context chat engine.
8. Run an interactive loop in the terminal until you type an exit word. Empty input is rejected with a warning.
9. For every question, print the answer, the source nodes with their scores, the token usage, and the estimated cost.
10. If anything fails after that, print the traceback and a categorized summary, then exit with code 1.

### ✨ Key Features
* 🎬 **Self-written transcript loader.** Fetches captions with `youtube-transcript-api` and builds the LlamaIndex documents itself, with the video id and URL stored as metadata.
* 🧠 **Semantic retrieval over the transcript.** The text is split into 2048-token chunks with a 200-token overlap, embedded with `models/gemini-embedding-2`, and searched by meaning rather than by exact keyword.
* ♻️ **Ingest once.** If the ChromaDB collection already has data, the script skips the download and the embedding and loads the existing index.
* 🎯 **Retrieve, then rerank.** Pulls the top 5 chunks for each question, then narrows them to the top 3 with a `cross-encoder/ms-marco-MiniLM-L-2-v2` reranker before they reach the LLM.
* 💬 **Persistent conversational memory.** Chat history is stored in PostgreSQL through `Memory` and AsyncPG, so context survives between runs.
* 🧱 **Grounded by instruction.** A strict system prompt tells the model to answer only from the transcript and to admit when it does not know.
* 🔍 **Transparent answers.** The retrieved source nodes are printed with their scores and text after every response.
* 📊 **Token and cost tracking.** Token counts come from LlamaIndex's callback handler, and the cost is estimated per turn through LiteLLM.
* 🛰️ **Phoenix tracing.** Every run sends its traces to the Arize Phoenix project `youtube_vector_project`.
* 🛡️ **Categorized error handling.** Failures print the raw traceback, then a readable summary such as quota, authentication, database, or access error.

---

## 🎯 Context & Problem Statement

### 📖 The Problem

Video is where a lot of knowledge lives today, and it is hard to work with. You cannot press `Ctrl+F` on something that was spoken, and scrubbing through an hour of video to find one explanation is slow. Pasting a transcript into a general chatbot brings its own problems. Four of them show up when you try to build something better.

1. **A video cannot be searched like text.** Finding the part that explains why something works, or how two ideas connect, means watching or skimming. Keyword search would not help much even on the transcript, because a question phrased differently from the speaker's words can match nothing.
2. **A general chatbot answers from memory, not from the video.** Asked about a specific video, a plain LLM may rely on old training data, mix in other sources, or guess, and it has no duty to say "that was not in the video".
3. **Naive pipelines are wasteful and forgetful.** A basic fetch-embed-ask script downloads and embeds the same video on every run, which costs time and embedding calls, and it loses the whole conversation when the process exits.
4. **It is hard to see what an answer was based on and what it cost.** Without printed sources and token counts, you cannot check an answer against the video or know what a question cost.

### 💡 The Solution

Each problem maps to one piece of the pipeline, and each piece has a known gap.

| Problem | How this repo handles it | What is still missing |
| :--- | :--- | :--- |
| 1. Video cannot be searched | The transcript is split into overlapping chunks, embedded, and searched by meaning in ChromaDB | Only the captions are used, so videos without captions cannot be read, and the default transcript language is English |
| 2. Answers from memory | Retrieved chunks are reranked by a cross-encoder, a strict system prompt limits the answer to them, and the source nodes are printed | The limit is an instruction to the model and not a hard check on its output |
| 3. Wasteful and forgetful | The ingestion step is skipped when the collection has data, and chat history lives in PostgreSQL | The check only looks at whether the collection is empty, and the session id is fixed in the script |
| 4. Hidden sources and cost | Source nodes, token counts, and a per-turn cost estimate are printed, and Phoenix records a trace | Token counts use an OpenAI tokenizer, so they are approximate for Gemini |

---

## 🧠 Core Concepts Explained

If RAG is new to you, this table explains the terms used in the rest of the README.

| Concept | Plain-language explanation | Where you see it |
| :--- | :--- | :--- |
| **RAG** | Retrieval-Augmented Generation. First find the passages that are relevant to the question, then give them to the LLM together with the question, so the answer is based on your source | The whole script |
| **Transcript** | The captions of a video as text. Here it is fetched with `youtube-transcript-api` and joined into one text per video | `load_youtube_transcripts` |
| **Document** | A LlamaIndex object that holds text and metadata. Each video becomes one document with its `video_id` and `url` | `Document(...)` |
| **Chunking** | Splitting long text into smaller pieces, because search and prompting work better with short passages. `chunk_size=2048` is counted in tokens, and `chunk_overlap=200` repeats the last 200 tokens at the start of the next chunk, so a sentence cut at the boundary keeps its context | `SentenceSplitter` |
| **Embedding** | A list of numbers that represents the meaning of a text. Texts with similar meaning get similar numbers, which makes search by meaning possible | `models/gemini-embedding-2` |
| **Chroma collection** | A named container inside ChromaDB that holds vectors, much like a table in a database | `youtube_docs` |
| **Vector store and index** | The place where chunks and embeddings are kept, and the LlamaIndex object that searches it | `ChromaVectorStore`, `VectorStoreIndex` |
| **Retrieval and top-k** | Finding the chunks whose embeddings are closest to the question. `similarity_top_k=5` means "fetch the 5 closest" | `as_chat_engine(..., similarity_top_k=5)` |
| **Reranker (cross-encoder)** | A second, more careful scoring step. The first search compares embeddings quickly. The cross-encoder then reads the question and each retrieved chunk together and re-scores them, keeping only the best `top_n` | `SentenceTransformerRerank` (5 fetched, 3 kept) |
| **Context chat engine** | A chat engine that, for every message, retrieves context, adds it to the prompt, and remembers the exchange | `chat_mode="context"` |
| **System prompt** | A standing instruction given to the LLM before the conversation, here "answer only from the transcript" | `system_prompt` |
| **`Memory`** | LlamaIndex's conversation memory. With `async_database_uri`, the history is stored in a database table. `token_limit` caps how much history is kept | `Memory.from_defaults(...)` |
| **LiteLLM** | A library that offers one interface to many LLM providers. Here it calls Gemini and provides the cost calculation | `LiteLLM`, `litellm.completion_cost` |
| **Trace** | A step-by-step record of what happened inside a run. Arize Phoenix collects traces and shows them in a web interface | `register(...)`, `LlamaIndexInstrumentor` |
| **`TokenCountingHandler`** | A LlamaIndex callback that counts the tokens sent to and received from the LLM | `token_counter` |
| **`LlamaDebugHandler`** | A LlamaIndex callback that records LLM inputs and outputs. The script reads its events to calculate the cost | `debug_handler` |
| **`nest_asyncio`** | Allows async code to run in environments where an event loop is already running | `nest_asyncio.apply()` |

---

## 🧩 Tech Stack Highlights

| Layer | Tool | Role |
| :--- | :--- | :--- |
| **Transcript source** | `youtube-transcript-api` | Fetches the captions of each video |
| **Orchestration** | LlamaIndex core (`Settings`, `Document`, `VectorStoreIndex`, `StorageContext`, `SentenceSplitter`) | Global configuration, chunking, and index construction |
| **Text embeddings** | `GoogleGenAIEmbedding` (`models/gemini-embedding-2`) | Embeds the transcript chunks |
| **Vector storage** | ChromaDB Cloud (`chromadb.CloudClient`, `ChromaVectorStore`) | Stores the vectors in the `youtube_docs` collection |
| **Reranking** | `SentenceTransformerRerank` (`cross-encoder/ms-marco-MiniLM-L-2-v2`) | Rescores the retrieved chunks and keeps the top 3 |
| **LLM** | LiteLLM with `gemini/gemini-2.5-flash` (temperature 0.3) | Answer generation |
| **Chat engine** | LlamaIndex `context` chat mode | Retrieves context on every turn and injects it into the prompt |
| **Conversation memory** | `Memory` + AsyncPG (PostgreSQL) | Persistent chat history across runs |
| **Observability** | Arize Phoenix (`phoenix.otel`, OpenInference LlamaIndex instrumentor) | Traces of every run |
| **Metrics** | `TokenCountingHandler` with `tiktoken`, `LlamaDebugHandler`, `litellm.completion_cost` | Token counts and cost estimate |
| **Config & runtime** | `python-dotenv`, `nest-asyncio`, `asyncio` | Loads secrets from `.env` and runs the async loop |

---

## ⚙️ Configuration Reference

Every tunable value lives directly in `Vector-self-make-youtube-reader-cmd.py`.

| Setting | Value | Effect |
| :--- | :--- | :--- |
| **Target videos** | `youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]` | The videos that are fetched and indexed on a fresh run |
| **Transcript language** | The default of `youtube-transcript-api` (English) | The transcript is requested without a language option |
| **Chunking** | `chunk_size=2048`, `chunk_overlap=200` | How the transcript is split before embedding |
| **Embedding model** | `models/gemini-embedding-2`, `embed_batch_size=50` | Embeds the chunks and each question |
| **Retrieval depth** | `similarity_top_k=5` | Number of chunks fetched per question |
| **Reranker** | `cross-encoder/ms-marco-MiniLM-L-2-v2`, `top_n=3` | Number of chunks that reach the LLM |
| **LLM** | `gemini/gemini-2.5-flash`, `temperature=0.3` | The answer model and its randomness |
| **LLM fallback** | A commented-out block in the script | Model fallback and retries are disabled |
| **Chat mode** | `context` | Retrieves context on every message |
| **Chroma collection** | `youtube_docs` | Where the transcript vectors are stored |
| **Chat memory** | `session_id="youtube_session"`, `token_limit=40000`, `table_name="youtube_vector_memory"` | How the chat history is stored and trimmed |
| **Phoenix project** | `youtube_vector_project` | The project that receives the traces |
| **Phoenix endpoint** | `PHOENIX_COLLECTOR_ENDPOINT`, with a hardcoded default | Where traces are sent. The default points to the author's Phoenix space |
| **Token counting** | `tiktoken` with `cl100k_base` | The tokenizer behind the token numbers |
| **Cost format** | Six decimal places | The estimated cost of each turn, for example `$0.002618 USD` |
| **Debug trace** | `LlamaDebugHandler(print_trace_on_end=True)` | Prints a trace in the terminal after each run |
| **Logging** | The root logger and the `httpx`, `google_genai`, and `litellm` loggers are set to `WARNING` | Keeps the terminal readable |
| **Checked at startup** | `GOOGLE_API_KEY`, `PHOENIX_API_KEY` | The script stops with an error if either is missing |
| **Not checked at startup** | `CHROMA_API_KEY`, `ASYNCPG_DATABASE_URL` | They are passed straight to their clients |
| **Exit commands** | `exit`, `close`, `out`, `break`, `keluar`, `x` | Ends the session |

---

## 💡 Questions to Try

The first three questions are the ones used in the [benchmark repository](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid), in increasing order of difficulty. They are generic enough to ask about any video.

| Question | What it shows |
| :--- | :--- |
| `What is this video mainly about?` | A simple question that needs the overall topic |
| `What are the main points or steps explained in this video, and how do they relate to each other?` | An analytical question that needs several parts of the transcript |
| `Based on everything discussed in this video, what conclusion or key takeaway can be drawn, and what supporting details from different parts of the video lead to that conclusion?` | A multi-hop question that connects details from different parts of the video |
| `Explain that in simpler terms`, right after any answer | The memory at work, because the follow-up only makes sense with the previous answer |
| A question the video never covers | The grounding instruction at work, where the model should say it does not know |

---

## 🖥️ Example Console Output

An illustrative first-run session, based on the messages the script prints:

```text
⚙️ [SYSTEM] Configuring LlamaIndex Settings (LLM & Embeddings)...
📊 [SYSTEM] Initializing Phoenix Observability & Tracing...
⚙️ [SYSTEM] Initializing Cross-Encoder Reranker...
⏳ [SYSTEM] Initializing ChromaDB Cloud Client...
🔌 [SYSTEM] Attaching ChromaVectorStore adapter...
⏳ [SYSTEM] ChromaDB is empty. Extracting transcripts from provided YouTube URLs...
📥 [SYSTEM] Fetching YouTube transcripts...
✅ [SYSTEM] Successfully loaded transcripts from 1 video(s).
⚙️ [SYSTEM] Creating Vector Store Index and embedding documents to ChromaDB Cloud...
✅ [SYSTEM] Index successfully generated and stored in ChromaDB Cloud.
🧠 [SYSTEM] Initializing Chat Engine and persistent PostgreSQL memory...
✅ [SYSTEM] Chat Engine successfully integrated and ready for interaction.

======================================================================
💬 [SYSTEM] Interactive YouTube Reader Session Started. Type 'exit' to stop.
======================================================================

🗣️ [USER] You: <your question about the video>
🧠 [AI] Processing your query against the YouTube transcript context...
----------------------------------------------------------------------
🤖 [AI RESPONSE]:
<the answer, based on the retrieved transcript passages>
----------------------------------------------------------------------

🔍 [SOURCE NODES] Retrieved <number> relevant nodes from the vector store:
--- Node 1 (score: <rerank score>) ---
📄 Text Context: <the transcript chunk>

📊 [LLAMAINDEX CALLBACK TOKEN OVERVIEW]:
Model Used        : <model>
Prompt Tokens     : <number>
Completion Tokens : <number>
Total LLM Tokens  : <number>
======================================================================

💰 [COST ANALYSIS]:
Total Estimated Cost: $<amount> USD
======================================================================
```

On later runs, the ingestion lines are replaced by `📂 [SYSTEM] Existing data found in ChromaDB. Bypassing extraction phase...` and `✅ [SYSTEM] Index successfully loaded from cloud storage.`

---

## 📁 Repository Structure

```text
youtube-vector-rag-llamaindex/
├── Vector-self-make-youtube-reader-cmd.py
├── LICENSE
├── README.md
└── requirements.txt
```

| Path | Description |
| :--- | :--- |
| `Vector-self-make-youtube-reader-cmd.py` | The full pipeline: transcript loading, indexing, retrieval, reranking, chat engine, and CLI loop in one script |
| `requirements.txt` | All Python dependencies, with notes inside the file |
| `LICENSE` | MIT License |
| `README.md` | You're reading it |

---

## 🏗️ Architecture & Data Flow

### 🔄 End-to-End System Flowchart

```mermaid
flowchart TD
    A["YouTube URLs from the youtube_urls list"] --> B{"Is the Chroma collection youtube_docs empty?"}

    B -- "Yes" --> C
    B -- "No" --> H["VectorStoreIndex.from_vector_store loads existing data"]

    subgraph Ingestion ["Runs only if the collection is empty"]
        C["load_youtube_transcripts fetches the captions and builds Documents"] --> D["SentenceSplitter chunks the text, 2048 size and 200 overlap"]
        D --> E["GoogleGenAIEmbedding produces vectors"]
        E --> F["ChromaDB Cloud collection youtube_docs"]
    end

    F --> G["VectorStoreIndex"]
    H --> G

    Q["Question in the terminal"] --> I["Retriever returns the top 5 chunks"]
    G --> I
    I --> J["Cross-encoder reranker keeps the top 3"]
    J --> K["Context chat engine"]
    L["PostgreSQL memory through AsyncPG"] --> K
    M["Gemini LLM through LiteLLM"] --> K
    K --> N["Answer based on the transcript"]
    N --> O["Answer, source nodes, tokens, and cost are printed"]
```

> **Note.** On every run, the script first checks whether the Chroma collection already has data. If it does, nothing is downloaded or embedded again, and the stored index is loaded. Phoenix records a trace for each run, and the token counter and debug handler observe every LLM call.

---

## 🆚 Three RAG Architectures

This repository implements **Vector RAG**. The same task can also be done with a knowledge graph, or with both approaches combined. That gives three versions of the same chatbot, plus a repository that measures what each one costs. This section explains how they differ, where each one lives, and what the benchmark found.

### 📍 Where Each Architecture Lives

| Architecture | Approach | Location |
| :--- | :--- | :--- |
| **Vector RAG** | Similarity search over embeddings | This repository (`Vector-self-make-youtube-reader-cmd.py`) |
| **Graph RAG** | Search over a knowledge graph | Coming soon |
| **Hybrid RAG** | Vector search and graph search combined | Coming soon |
| **Token and cost benchmark** | Compares the three on the same video and questions | [rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) |

### 🔍 What Differs

The Graph and Hybrid columns follow the setup used in the benchmark repository.

| Aspect | Vector RAG (this repo) | Graph RAG (coming soon) | Hybrid RAG (coming soon) |
| :--- | :--- | :--- | :--- |
| How context is found | The question is compared with the chunk embeddings, and the closest chunks are used | The text is turned into relationships (subject, relation, object) kept in a graph, and the question is answered from that graph together with the source text | Both retrievers run, and their results are merged with Reciprocal Rank Fusion. The LLM also writes extra variants of the question before the search |
| Storage | ChromaDB Cloud | Neo4j | ChromaDB Cloud and Neo4j |
| Extra LLM work when indexing | None, only embeddings | An LLM extracts the relationships from every chunk | The same as Graph RAG |
| Reranking | Cross-encoder, top 5 to top 3 | The same cross-encoder | The same cross-encoder |
| Moving parts | Fewest | More | Most |

### 📊 About the Benchmark Repository

[rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) builds the three architectures end to end on the same YouTube transcript and asks each of them the same three questions, the simple, analytical, and multi-hop ones listed under [Questions to Try](#-questions-to-try). Every turn reports prompt, completion, and total tokens plus an estimated cost in USD. All runs use `gemini-2.5-flash` with fallback disabled, so the model, the source, and the questions stay the same across the three.

Results from that repository, shown as total tokens and estimated cost:

| Question | Vector RAG | Graph RAG | Hybrid RAG |
| :--- | :---: | :---: | :---: |
| Q1 (Simple) | 5,858 tokens, $0.002618 | 4,959 tokens, $0.004638 | 9,208 tokens, $0.010026 |
| Q2 (Analytical) | 7,318 tokens, $0.005876 | 7,989 tokens, $0.010416 | 10,408 tokens, $0.010901 |
| Q3 (Multi-hop) | 8,754 tokens, $0.005338 | 9,836 tokens, $0.008787 | 12,871 tokens, $0.012668 |

In estimated cost, the order was Vector, then Graph, then Hybrid on every question. In total tokens, the same order held for the analytical and multi-hop questions. On the simple question, Graph used fewer total tokens than Vector (4,959 against 5,858) but still cost more, because its completions were longer.

These numbers come from one video, three questions, and one run per question, with costs estimated through LiteLLM. The benchmark measures tokens and cost only. It does not measure how good the answers are.

### 🧭 Which One Fits Your Case

* If you want the simplest setup and the lowest cost, Vector RAG is the place to start, and this repository shows it.
* Graph and Hybrid RAG add a graph store and an LLM step at indexing time, and the benchmark shows what each architecture costs per question.
* Whether Graph or Hybrid RAG answers better than Vector RAG was not measured, so the choice between them still needs a quality comparison.

---

## 💻 Installation & Reproduction Steps

### 📋 Prerequisites
* **Python 3.10 or newer** is required.
* **Accounts and keys** you will need: a Google AI Studio API key for Gemini, a [Chroma Cloud](https://www.trychroma.com/) API key, an [Arize Phoenix](https://phoenix.arize.com/) API key, and a PostgreSQL database for the conversation memory. A hosted database, for example one on Supabase, works with its connection string.
* **Internet access**, because the script calls YouTube, Gemini, ChromaDB Cloud, Phoenix, and PostgreSQL. On the first run it also downloads the reranker weights from Hugging Face and the tokenizer file that `tiktoken` needs.

### 🛠️ CLI Installation & Execution

#### 1. Clone the Repository
```bash
git clone https://github.com/viochris/youtube-vector-rag-llamaindex.git
cd youtube-vector-rag-llamaindex
```

#### 2. Create and Activate a Virtual Environment
```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

#### 3. Install Dependencies
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
> `torch` is a large download, so the first installation can take a while. The LlamaIndex ecosystem also ships as many small, fast-moving sub-packages. If you hit a resolver conflict, try installing into a clean virtual environment and letting `pip` resolve every package from `requirements.txt` in a single command rather than one at a time, since that gives it the full picture needed to pick mutually compatible versions.

#### 4. Configure Environment Variables
Create a `.env` file in the project root.
```env
GOOGLE_API_KEY=your_google_ai_studio_key
CHROMA_API_KEY=your_chroma_cloud_key
PHOENIX_API_KEY=your_phoenix_key
PHOENIX_COLLECTOR_ENDPOINT=https://app.phoenix.arize.com/s/your-space/v1/traces
ASYNCPG_DATABASE_URL=postgresql+asyncpg://user:password@host:5432/dbname
```

| Variable | Purpose |
| :--- | :--- |
| `GOOGLE_API_KEY` | Gemini key. The embedding model receives it directly, and LiteLLM finds its Gemini key in the environment for the LLM calls |
| `CHROMA_API_KEY` | Connects to ChromaDB Cloud |
| `PHOENIX_API_KEY` | Authenticates the trace upload to Phoenix |
| `PHOENIX_COLLECTOR_ENDPOINT` | Optional. Where traces are sent |
| `ASYNCPG_DATABASE_URL` | Async connection string for the chat memory table |

> The script has a default value for `PHOENIX_COLLECTOR_ENDPOINT` that points to the author's own Phoenix space. Set this variable to your own space, otherwise your traces will not appear in your account.
>
> Only `GOOGLE_API_KEY` and `PHOENIX_API_KEY` are checked at startup. Keep `.env` out of version control.

#### 5. Set the Target Videos
The video URLs are set directly in the script.
```python
youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]
```
Edit this list before the first run. The video needs captions, and the transcript is requested in the default language, which is English.

#### 6. Run the Application
```bash
python Vector-self-make-youtube-reader-cmd.py
```
Type your questions at the `🗣️ [USER] You:` prompt. Type `exit`, `close`, `out`, `break`, `keluar`, or `x` to end the session.

#### 7. Switch to Another Video or Start Over
Scraping only happens when the Chroma collection is empty, so changing `youtube_urls` has no effect while `youtube_docs` still holds data. Delete the collection once, and the next run indexes the new list. The chat history stays in PostgreSQL, so also change `session_id` in the script, or clear the `youtube_vector_memory` table, if you do not want the old conversation to carry over.
```python
import os
import chromadb
from dotenv import load_dotenv

load_dotenv()
client = chromadb.CloudClient(api_key=os.getenv("CHROMA_API_KEY"))
client.delete_collection("youtube_docs")
```

---

## 🛠️ Troubleshooting & Error Messages

### 🚨 Error Categories Printed by the Script

When something fails after the startup key check, the script prints the traceback and then one of these summaries, and exits with code 1. The checks run from top to bottom, and the first match is used. A missing `GOOGLE_API_KEY` or `PHOENIX_API_KEY` is raised before this handler starts, so it appears as a normal Python error with its message and no summary.

| Message prefix | Matches when |
| :--- | :--- |
| `🚨 [API ERROR]` | The error type is `ResourceExhausted`, or the message contains `quota` or `429` |
| `🚨 [DATA ERROR]` | A `ValueError` whose message contains `empty` |
| `🚨 [AUTH ERROR]` | The error type is `InvalidArgument`, or the message contains `api_key` |
| `🚨 [DATABASE ERROR]` (ChromaDB) | The error type is `OperationalError`, `DatabaseError`, or `InvalidCollectionException`, or the message contains `chroma` |
| `🚨 [DATABASE ERROR]` (PostgreSQL) | The message contains `asyncpg` or `postgres` |
| `🚨 [ACCESS ERROR]` | A `PermissionError` |
| `🚨 [UNKNOWN ERROR]` | Anything that does not match the categories above |

### 🧰 Common Problems

| Symptom | Likely cause | What to do |
| :--- | :--- | :--- |
| `GOOGLE_API_KEY is missing` or `PHOENIX_API_KEY is missing` at startup | The variable is missing from `.env` | Add it to `.env` in the project root |
| LiteLLM reports a missing Gemini key | LiteLLM did not find the key in the environment | Also set `GEMINI_API_KEY` in `.env` with the same value as `GOOGLE_API_KEY` |
| `UNKNOWN ERROR` mentioning a transcript | The video has no captions, captions are disabled, or no English transcript exists | Choose a video with English captions |
| `UNKNOWN ERROR` mentioning an invalid YouTube URL | The URL does not contain `youtube` or `youtu.be` | Use a standard `youtube.com` or `youtu.be` link |
| YouTube refuses the request | YouTube can block requests from some cloud provider and VPN IP addresses | Run the script from a regular home or office connection |
| Answers still come from the old video | The collection already has data, so the new URLs are ignored | Delete the collection as shown in step 7 |
| `UNKNOWN ERROR` (`TypeError`) saying `CloudClient()` is missing `tenant` and `database` | An old `chromadb` version is installed, and it requires both arguments | Run `pip install --upgrade "chromadb>=1.1.0"` or reinstall from `requirements.txt` |
| `ImportError` asking for `sentence-transformers` or `torch` | The reranker dependencies are not installed | Run `pip install -r requirements.txt` in the active virtual environment |
| The first run is slow before the chat starts | The reranker weights are downloaded from Hugging Face | Wait for the download. Later runs reuse the files |
| Traces do not show up in your Phoenix account | `PHOENIX_COLLECTOR_ENDPOINT` still points to the default space | Set it to your own Phoenix endpoint |
| Error about the async driver | `asyncpg` is not installed, or `ASYNCPG_DATABASE_URL` does not start with `postgresql+asyncpg://` | Run `pip install -r requirements.txt` and check the URL |
| The chat remembers an old conversation | The memory uses a fixed `session_id` | Change `session_id` in the script, or clear the `youtube_vector_memory` table |
| Long trace output appears after each answer | `LlamaDebugHandler` is created with `print_trace_on_end=True` | Set it to `False` in the script |
| A database error appears for a problem that is not about the database | Any message containing `chroma` or `postgres` is reported as a database error | Read the printed traceback to find the real cause |

---

## 📝 Conclusion

This project set out to solve four problems: a video that cannot be searched like text, a general chatbot that answers from memory instead of from the video, a naive pipeline that downloads, embeds, and forgets everything on every run, and answers whose sources and cost stay hidden. The script addresses them directly. It fetches the transcript with its own loader, chunks and embeds it so questions are matched by meaning, reranks what it retrieves, tells the model to answer only from that context, and keeps both the vectors and the conversation so repeated use stays cheap and continuous. Every answer is printed with its source passages, its token usage, and its estimated cost.

The design choices hold up reasonably well on their own terms. Skipping ingestion when the collection has data avoids paying for the same download and embeddings twice. Reranking with a small cross-encoder before the LLM is a deliberate quality step, and keeping the whole flow in one readable script makes it easy to follow, run, and modify.

It is also not a finished product. The video list is hardcoded, changing it requires clearing the collection by hand, the transcript carries no timestamps, and nothing measures how good the answers are. The sections below go through these gaps and lay out the next steps.

---

## ⚠️ System Limitations

### 🏗️ Architectural Limitations
* **Hardcoded target videos.** The URLs are set by editing the `youtube_urls` list in the script, not passed in as an argument.
* **One-time ingestion gate.** Fetching and indexing only run when the `youtube_docs` collection is empty. Changing the list without clearing the collection does not trigger re-ingestion, so the script keeps answering from the old videos.
* **One shared collection and one fixed conversation.** Every video goes into the same collection, and the memory uses one `session_id` and one table name, so every run continues the same conversation, even after you switch videos.
* **Any error ends the session.** After the startup key check, the whole pipeline, including the chat loop, sits inside one top-level `try` block. A transient failure on a single turn, such as an API rate limit, is reported but still stops the script.
* **Settings are hardcoded.** The collection name, model names, chunk settings, retrieval depth, Phoenix project name, and memory settings are fixed values in the script.
* **Minimal startup validation.** Only `GOOGLE_API_KEY` and `PHOENIX_API_KEY` are checked. A missing Chroma key or database URL only shows up when that client is first used.
* **Model fallback is disabled.** The fallback and retry options are commented out, so a failed model call is not retried on another model.
* **Dependence on external services.** The script needs live access to YouTube, Gemini, ChromaDB Cloud, Phoenix, and PostgreSQL. There is no offline mode.
* **CLI-only interface.** All interaction happens in a terminal. There is no web page and no visual history.

### 🔬 Model & Domain Limitations
* **Captions only.** A video without captions cannot be read, and automatic captions can contain mistakes that the model then repeats. The transcript is requested in the default language, which is English.
* **No timestamps.** The loader joins the caption snippets into plain text, so answers cannot point to a moment in the video.
* **Simple URL parsing.** The video id is taken from the URL with string rules. Unusual link formats may fail.
* **Retrieval on every message.** The context chat mode retrieves passages for every message, including greetings and thanks.
* **Grounding relies on the prompt.** Staying within the transcript is enforced by the system prompt and the retrieval setup, not by a hard check on the output.
* **Fixed retrieval settings.** Chunk size, overlap, `similarity_top_k`, and the reranker's `top_n` are fixed. A broad question that needs many parts of a long video is limited by only three chunks reaching the LLM.
* **Local reranking.** The cross-encoder runs on your machine, so it needs `torch`, a one-time weight download, and some compute on every turn.
* **Token counts are approximate.** `TokenCountingHandler` uses the `cl100k_base` tokenizer from OpenAI, while the model is Gemini.
* **The cost covers LLM calls only.** Embedding calls are not included, and the estimate depends on the response data that LiteLLM receives.
* **No quality evaluation.** Retrieval and answer quality were not measured against a labeled question set.

---

## 🚀 Future Work
* **Take the video URLs as an argument.** Accept one or more URLs on the command line, so the code does not need editing.
* **One collection and one session per video.** Name the collection and the memory session after the video id, so switching videos needs no manual cleanup.
* **Keep timestamps.** Store the start time of each caption in the metadata, so answers can point to the moment in the video.
* **Support more transcript languages.** Let the user choose the language, or fall back to another one when English is missing.
* **Make the chat loop resilient.** Move error handling inside the loop, so a failed turn prints the error and lets you try again.
* **Enable model fallback.** Turn on the retry and fallback options through the LiteLLM configuration.
* **Move configuration out of the code.** Take model names, chunk settings, and retrieval depth from environment variables or arguments.
* **Evaluate retrieval and answers.** Build a small labeled question set for a few videos and measure retrieval and answer quality, instead of relying on fixed defaults.
* **Add a simple interface.** A small Streamlit page would make the answers and their sources easier to read than terminal output.

---

## 📄 License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---
**Author:** [Silvio Christian, Joe](https://github.com/viochris)

*"Ask a video a question, and let its own transcript answer."*
