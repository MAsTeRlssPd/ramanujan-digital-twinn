# Ramanujan Digital Twin

An educational conversational experience inspired by the life and mathematical work of Srinivasa Ramanujan (1887–1920). The application combines a historical persona prompt, retrieval from a curated Markdown corpus, and session memory to answer questions about Ramanujan and his mathematics.

> **Scope:** This is an AI-generated educational simulation, not a representation or endorsement by Ramanujan or his estate. Responses can be inaccurate; consult primary sources for research.

## Features

- Streaming conversation over WebSockets, with session history and session management.
- Retrieval-augmented answers grounded in a local corpus of 27 Markdown documents, with source excerpts available in the interface.
- Short-term conversation history and SQLite memory extraction for personalization.
- Mathematical notation rendered with KaTeX, plus optional 2D and 3D plots and a handwriting-style canvas for supported responses.
- Optional image input, browser speech recognition, and generated speech playback.
- A life timeline and a dashboard for memories associated with a session.

Voice input depends on browser support for the Web Speech API. Speech playback uses the `edge-tts` service. The frontend loads several visualization and math-rendering libraries from public CDNs.

## How the application works

### Chat request lifecycle

1. The browser opens a WebSocket at `/ws/chat/{session_id}` and sends a JSON message containing the text and, optionally, an image, MIME type, API key, and user ID.
2. The backend creates or loads the session and saves the user message in SQLite.
3. The RAG retriever embeds the question and queries the ChromaDB collection. Results beyond the configured distance threshold are discarded.
4. Recent turns, retrieved passages, and saved long-term memories are assembled with the persona prompt.
5. Gemini generates the answer. Text chunks and source metadata are streamed back to the browser.
6. The browser renders Markdown-like content, equations, source cards, and supported plot or handwriting blocks.
7. The completed exchange is saved, and a background Gemini call may extract up to three facts for the session’s long-term memory.

RAG is best-effort: if retrieval cannot initialize, chat orchestration logs a warning and continues without retrieved context. Gemini generation still requires a valid API key.

### Knowledge-base lifecycle

`rag.ingest` reads each `corpus/*.md` file, derives a document type and topic from its filename, splits text into overlapping chunks, embeds the chunks with Gemini, and stores them in the persistent ChromaDB collection. The chunker protects inline and display LaTeX while splitting. Re-running ingestion clears and recreates the collection before adding the current corpus.

`build_corpus.py` can optionally refresh sources defined in `rag/web_scraper.py` before ingestion. Scraping updates files in `corpus/`; inspect and review those changes before committing them.

## Knowledge-base contents

The repository currently includes 27 Markdown documents. They cover biographical material, books and notebooks, correspondence, mathematical topics, papers, web references, and lecture material.

| Area | Representative files |
| --- | --- |
| Life and biography | `biography_early_life.md`, `biography_cambridge.md`, `biography_illness_return.md`, `book_srinivasa_ramanujan_biography.md` |
| Books and notebooks | `book_lost_notebook.md`, `book_kerala_rare_28155.md`, `book_kerala_rare_37877.md` |
| Letters | `letter_to_hardy_1913.md`, `letter_final_mock_theta.md` |
| Mathematics | `math_partition_function.md`, `math_mock_theta.md`, `math_modular_forms.md`, `math_continued_fractions.md`, `math_pi_series.md`, `math_rogers_ramanujan.md`, `math_highly_composite.md`, `math_taxicab_1729.md` |
| Papers and digital references | `paper_lost_notebook_wiki.md`, `paper_ramanujan_sum_wiki.md`, `paper_ramanujan_computing_technology.md`, `web_imsc_ramanujan.md`, `web_wikipedia_ramanujan.md`, `web_ramanujan_explained.md`, `web_ramanujan_conjecture_wiki.md` |
| Media and lectures | `doc_ramanujan_legacy_documentary.md`, `lecture_youtube_NVVhGtFVpEU.md`, `web_math4raghuram_videos.md` |

The corpus is a curated application dataset, not a complete or authoritative digital edition of Ramanujan’s writings. Consult the documents’ original publishers and repositories for provenance and reuse terms.

## Browser experience

- **Conversation sidebar:** create, revisit, rename, and delete sessions.
- **Chat composer:** send text or an image; use microphone input when supported by the browser.
- **Memory panel:** inspect facts extracted from exchanges associated with the selected session.
- **Timeline:** browse key events from 1887 to 1920.
- **Sources:** expand retrieved excerpts shown with responses.
- **Math display:** render LaTeX and supported 2D/3D graph configurations produced in responses.
- **Voice playback:** request speech from the backend, or enable automatic reading in the interface.

The interface uses CDN-hosted libraries and Google Fonts, so those assets require network access in the browser. Voice recognition is browser-native and varies by browser and operating system.

## Requirements

- Python 3.10 or newer
- A Google Gemini API key with access to the configured generation and embedding models
- Internet access for Gemini API calls and (when using voice playback) Edge TTS

## Local setup

```bash
git clone https://github.com/MAsTeRlssPd/ramanujan-digital-twinn.git
cd ramanujan-digital-twinn
python -m venv .venv
```

Activate the virtual environment, then install dependencies:

```bash
# Windows PowerShell
.venv\Scripts\Activate.ps1

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

Configure the API key using either method:

1. Set `GOOGLE_API_KEY` in your environment (or in a local `.env` file in the project root); or
2. Enter a key in the app’s API key prompt. The browser stores that key in local storage and sends it with chat requests.

Example `.env` file:

```dotenv
GOOGLE_API_KEY=your_gemini_api_key
```

The `.env` file is ignored by Git. Do not commit API keys.

## Build the knowledge base

The corpus is included in `corpus/`. Create or rebuild its ChromaDB index with:

```bash
python -m rag.ingest
```

This calls the Gemini embedding API and replaces the existing `ramanujan_knowledge` collection. It writes generated index files under `data/chroma_db/`.

To scrape the configured web sources before rebuilding, use `python build_corpus.py`. To scrape without indexing, use `python build_corpus.py --scrape-only`. Scraping requires network access and may change files in `corpus/`.

## Run the application

```bash
python run.py
```

Open [http://localhost:8000](http://localhost:8000). The development entry point enables Uvicorn reload. The API listens on `0.0.0.0` at port `8000` by default.

## Configuration

Configuration is centralized in `config.py` and can be adjusted there. Current defaults include:

| Setting | Default |
| --- | --- |
| Generation model | `gemini-3.5-flash` |
| Embedding model | `gemini-embedding-2` |
| Embedding dimensions | `768` |
| Chroma collection | `ramanujan_knowledge` |
| Retrieval results | `5` |
| Conversation history window | `20` turns |
| Server port | `8000` |

Additional RAG defaults in `config.py` are a chunk size of 700 characters, a 100-character overlap, and a relevance distance threshold of 1.5. Long-term memory extraction is enabled by default.

The Gemini key is read from `GOOGLE_API_KEY`. Chat requests can also provide a user-supplied API key from the browser UI.

## Architecture

```text
Browser (HTML, CSS, JavaScript)
  ├── WebSocket chat and REST requests
  ├── KaTeX, graphing, citations, timeline, voice and memory UI
  └── Optional image and API key input
                │
                ▼
FastAPI (app/main.py)
  ├── Chat orchestration (app/chat_handler.py)
  ├── Gemini generation and memory extraction
  ├── RAG retrieval ── ChromaDB + Gemini embeddings
  └── Session and memory storage ── SQLite
```

For each chat message, the backend loads or creates the session, retrieves relevant corpus passages, adds recent conversation and saved memory to the prompt, and streams the generated response. After the response, it can extract up to three facts for persistent memory.

## Project layout

```text
app/                  FastAPI routes and browser interface
  static/             HTML, CSS, and JavaScript assets
corpus/               Curated Markdown reference documents
memory/               SQLite schema, session manager, and extractor
persona/              Persona prompt and prompt templates
rag/                  Chunking, Gemini embeddings, ChromaDB, retrieval
build_corpus.py       Optional source scraping and corpus build command
config.py             Application paths and model/server settings
run.py                Local development entry point
startup.sh            Deployment startup script
render.yaml           Render service configuration
requirements.txt      Python dependencies
```

Runtime data is generated under `data/` and excluded from version control. It contains the ChromaDB index and SQLite memory database.

## HTTP endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Serve the chat interface |
| `GET` | `/api/health` | Health status |
| `GET` | `/api/timeline` | Ramanujan life timeline |
| `GET` | `/api/corpus/{filename}` | Read a corpus Markdown document |
| `POST` | `/api/tts` | Generate speech audio from text |
| `GET` | `/api/sessions` | List a user’s sessions (`X-User-ID` required) |
| `GET` | `/api/history/{session_id}` | Load session conversation history |
| `GET` | `/api/memories/{session_id}` | Load session memories |
| `PATCH` | `/api/sessions/{session_id}/name` | Rename a session |
| `DELETE` | `/api/sessions/{session_id}` | Delete a session and its stored data |
| WebSocket | `/ws/chat/{session_id}` | Stream chat messages and responses |

### WebSocket message format

The browser sends one JSON object per message. `content` is the text prompt; `image_data` and `mime_type` are optional image fields. `api_key` can carry the key entered in the browser. `user_id` identifies the browser-side user for session storage.

```json
{
  "content": "Explain the significance of 1729.",
  "api_key": "your_gemini_api_key",
  "user_id": "browser-user-id"
}
```

The server streams JSON events. Chat text arrives as `stream` events, source excerpts as a `sources` event, followed by an `end` event. Failures may be sent as an `error` event.

REST routes for sessions, history, and memories expect the `X-User-ID` request header. The browser supplies it when calling these endpoints.

### Health check

```bash
curl http://localhost:8000/api/health
```

Example response:

```json
{"status":"ok","persona":"Srinivasa Ramanujan (1887-1920)"}
```

## Deployment

The repository includes a Render Blueprint in `render.yaml`. It installs `requirements.txt` and starts the service with `startup.sh`. On startup, the script builds the ChromaDB index from the checked-in corpus if the index directory is empty, then starts Uvicorn.

Set `GOOGLE_API_KEY` in the deployment environment. The service needs outbound access to the Gemini API for indexing and chat. Runtime databases live in `data/`; configure persistent storage with the hosting provider if those files must survive instance replacement or restart.

The Render configuration currently defines a Python web service in Singapore, installs dependencies with `pip install -r requirements.txt`, uses `startup.sh`, and checks `/` for health. `startup.sh` uses the `PORT` supplied by the host and defaults to 8000. Corpus ingestion on a fresh instance uses the server-side `GOOGLE_API_KEY`; make sure it is configured before the first start. The deployment manifest does not itself declare a persistent disk.

## Data, privacy, and security notes

- The Gemini key entered in the browser is stored in that browser’s local storage and included in chat messages to this server. Anyone with access to the browser profile can access it. Clear it with the app’s key controls or browser storage.
- The server stores conversation history and extracted memory in `data/memory.db`. The RAG index is stored in `data/chroma_db/`. Both are local runtime data; back them up or configure persistent hosting storage if you need to retain them.
- Session APIs use a client-provided `X-User-ID` value to scope access. This is an application-level identifier, not a full authentication system.
- Do not expose a public deployment without considering rate limits, authentication, abuse controls, and storage policies. The repository is intended as an educational project and does not include a production identity or account system.
- User messages and images are sent to Google Gemini for generation. Review Google’s current API terms and data handling policies before using sensitive content.

## Troubleshooting

| Symptom | Checks |
| --- | --- |
| Gemini authentication or model error | Confirm the API key is valid and enabled for the configured models. Check `GENERATION_MODEL` and `EMBEDDING_MODEL` in `config.py`. |
| No retrieved sources | Run `python -m rag.ingest` with a valid `GOOGLE_API_KEY`; confirm `data/chroma_db/` was created and contains an index. |
| Ingestion fails | Check outbound network access, Gemini embedding access, and API quotas. Ingestion clears the existing collection before rebuilding it. |
| Microphone control is missing or unavailable | Check browser Web Speech API support and microphone permissions. |
| Speech playback fails | Confirm server outbound access to Edge TTS. The `/api/tts` route returns an error if the service is unavailable. |
| Session data disappears after deployment changes | Configure persistent storage for `data/` in the hosting environment. |

## Development and contribution

The project uses a lightweight Python backend and a static, framework-free frontend. To contribute:

1. Create a branch for the change.
2. Keep corpus edits traceable to their sources and review generated scraper changes.
3. Keep secrets, local `.env` files, and generated `data/` out of commits.
4. Document configuration or behavior changes in this README.
5. Include a clear summary of any manual validation performed.

There is currently no test suite or lint configuration in the repository. Validate changes manually by running the server and exercising the affected interface or endpoint.

## Acknowledgments

This project draws on historical material about Srinivasa Ramanujan and uses Google Gemini, ChromaDB, FastAPI, SQLite, KaTeX, Plotly, function-plot, Math.js, and Edge TTS. See `requirements.txt`, `app/static/index.html`, and the corpus files for implementation dependencies and source details.

## License and sources

No project license file is currently included. Check the provenance and reuse terms of the corpus documents and third-party assets before redistribution. Some historical works may be public domain, but this does not automatically apply to every source, transcription, image, or dependency in the repository.
