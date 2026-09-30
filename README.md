# Riptide

Voice and text search for the Enron email archive, with zero-trust retrieval checks. The interface follows `DESIGN.md`'s parchment, wine, violet, and restrained editorial style.

## Requirements

- Node.js 20 or newer
- Python 3.11 or newer
- Enough disk space for the Hugging Face data cache, the embedding model, metadata, and FAISS index
- An OpenAI-compatible LLM key for generated answers (the search/security pipeline can be prepared without one)

## Install and run

```powershell
npm install
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

Set `LLM_API_KEY`, `LLM_BASE_URL`, and `LLM_MODEL` in `.env`. Keep the key local; do not commit `.env`.

In separate terminals, from the project root:

```powershell
python -m uvicorn backend.main:app --reload
npm run dev
```

Vite forwards `/api` requests to the local FastAPI server at `127.0.0.1:8000`.

## Prepare the data

Train the injection model on the specified Hugging Face dataset, then ingest the exact Enron dataset. Ingestion embeds once and stores FAISS and metadata on disk; the API loads this saved index at startup and does not re-embed on restart.

```powershell
python -m backend.scripts.train_injection
python -m backend.scripts.ingest --limit 10000
```

The ingestion process streams `haritzpuerto/the_pile_00_Enron_Emails`, parses the email headers and bodies, semantically groups sentence embeddings, and reports progress. It refuses to claim success if fewer than 10,000 rows were actually read. Check `/api/dataset/status` for the live counts. Run ingestion again with a larger `--limit` to index more.

The classifier prints held-out accuracy, precision, and recall. Run it after ingestion to also get an ordinary Enron sample false-positive rate at the configured 0.60 block threshold. Inspect `backend/models/metrics.json`. The optional Kaggle source is documented as a fallback only; this implementation does not silently substitute it.

## Demo flow

1. Wait for `DATASET INITIALIZING` to change to the actual indexed email count.
2. Ask “What did employees discuss about the California energy project?” or use the microphone in Chrome/Edge on localhost or HTTPS.
3. Review the risk label and answer, then open a source card to inspect its chunk metadata.
4. Try an instruction override; a high-risk query should be blocked before LLM generation.

If no LLM key is configured, the UI states that generation is unavailable. It does not create a pretend answer. Query scoring requires the trained classifier, and retrieval requires the saved corpus index.

## Test-only malicious fixture

Enable `RIPTIDE_TEST_MODE=true` in `.env`, then create the fixture with:

```powershell
python -m backend.scripts.ingest --limit 10000 --test-mode
```

The fixture is stored separately, is not included in indexed email counts or the production FAISS index, and is added to retrieval only while test mode is enabled. The final user prompt is written to `backend/data/index/test_prompt.log` in this mode so the blocked chunk can be checked for absence. Turn test mode off for normal use. This is diagnostic logging and can contain test prompts.

## API

- `GET /api/health`
- `GET /api/dataset/status`
- `POST /api/query` (send an `X-Session-ID` header to retain session trust)
- `GET /api/pipeline/{request_id}`
- `GET /api/security/events`
- `GET /api/chunks/{chunk_id}`
- `POST /api/verify`

Voice recognition is browser-provided Web Speech API; when unsupported or denied, type into the same search box. Session Trust is an application-level heuristic, not a standardized metric.
