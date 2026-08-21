# NeuralPilot

An agentic AI chat assistant built with FastAPI, LangGraph, and Google Gemini — streaming
responses, web search, RAG over uploaded documents, long-term memory, and persistent chat history.

## Features

- **Agentic tool use** — a LangGraph `StateGraph` loops between the model and a tool node until
  the answer is complete, so the agent can chain multiple tools in a single turn.
- **Token streaming** — replies stream to the browser over Server-Sent Events. Tool call chunks
  and raw tool output are filtered out, so only assistant prose reaches the UI.
- **Web search** — Tavily (advanced depth, 5 results) for current events, prices, and releases.
- **Document RAG** — upload PDF, DOCX, TXT, MD, PY, or CSV; text is chunked (900 chars, 150
  overlap), embedded with `gemini-embedding-001`, and stored in Chroma. Retrieval is scoped to
  the conversation that uploaded the file.
- **Long-term memory** — the agent can save and recall facts you ask it to remember, per conversation.
- **Calculator** — arithmetic evaluated with builtins stripped and a restricted namespace exposing
  only `math` and a few numeric helpers.
- **Multi-conversation history** — conversations are listed in a sidebar and persist across restarts,
  backed by SQLAlchemy and LangGraph's SQLite checkpointer.
- **Model picker** — choose among five Gemini models from the UI; the selection is validated
  server-side and falls back to the default if unrecognised.
- **Voice input** — optional dictation via the browser Web Speech API, where supported.

## Architecture

| File | Responsibility |
| --- | --- |
| `app.py` | FastAPI routes, SSE streaming, file upload |
| `agent.py` | LangGraph agent, model resolution, checkpointing |
| `tools.py` | Tool definitions (search, RAG, memory, calculator) |
| `rag.py` | Text extraction, chunking, Chroma vector store |
| `database.py` | SQLAlchemy models and persistence helpers |
| `templates/index.html` | Single-page chat UI |

Request flow:

```
Browser ──POST /chat/stream──> FastAPI ──> LangGraph agent ──> Gemini
   ▲                                            │
   └──────── SSE tokens ────────────────────     ├─> Tavily web search
                                                 ├─> Chroma (uploaded docs)
                                                 ├─> SQLite (memory)
                                                 └─> calculator
```

State lives in two SQLite databases: `data/chatbot_memory.db` holds conversations, messages, and
long-term memory, while `data/langgraph_checkpoints.sqlite` holds the agent's own checkpointed
graph state. The vector store is a separate Chroma directory.

### Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Chat UI |
| `GET` | `/conversations` | List conversations, newest first |
| `GET` | `/history/{thread_id}` | Messages for one conversation |
| `POST` | `/upload` | Add a document to the vector store |
| `POST` | `/chat/stream` | Stream a reply as SSE |

## Setup

Requires Python 3.11.

```bash
git clone https://github.com/preranaj815/NeuralPilot.git
cd NeuralPilot

py -3.11 -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS / Linux

pip install -r requirements.txt
```

Create a `.env` file in the project root:

```ini
GOOGLE_API_KEY="your-key"
GOOGLE_MODEL="gemini-3.6-flash"   # optional, this is the default
TAVILY_API_KEY="your-key"

# Optional tracing
LANGSMITH_TRACING="true"
LANGSMITH_API_KEY="your-key"
LANGSMITH_PROJECT="NeuralPilot"
```

Get keys from [Google AI Studio](https://aistudio.google.com/apikey) and
[Tavily](https://app.tavily.com/).

Then run:

```bash
python app.py
```

The app serves on <http://localhost:8080>. `data/`, `uploads/`, and `chroma_db/` are created on
first run and are gitignored — your conversations, files, and embeddings stay local.

## Models

The default is `gemini-3.6-flash`. Selectable models, validated in `agent.py`:

```
gemini-3.6-flash        gemini-3.5-flash        gemini-3.1-pro-preview
gemini-3.7-flash        gemini-3.5-flash-lite
```

Override the default with `GOOGLE_MODEL`. An unrecognised model name from the client falls back to
the default rather than erroring. Note that `.env` changes need a full restart — `load_dotenv()`
will not override a variable already present in the environment, and the uvicorn reloader passes
its existing environment to child processes.

## Deployment

Deploying to a platform like Render:

- **Build command:** `pip install -r requirements.txt`
- **Start command:** `uvicorn app:app --host 0.0.0.0 --port $PORT`

Do not use `python app.py` in production — the `__main__` block hardcodes port 8080 and enables
the reloader.

Before deploying, be aware:

- **Ephemeral storage.** On platforms without a mounted disk, `data/`, `uploads/`, and `chroma_db/`
  are wiped on every deploy and restart, taking chat history and uploaded documents with them.
  Attach a persistent disk, or move to Postgres and a hosted vector store.
- **Environment variables.** `.env` is gitignored and therefore not deployed; set the keys in your
  host's dashboard instead.
- **Pin Python to 3.11**, since the dependency versions are resolved against it.
- **Memory.** Chroma and its `onnxruntime` dependency are heavy; 512 MB instances may run out.

## License

Apache 2.0 — see [LICENSE](LICENSE).
