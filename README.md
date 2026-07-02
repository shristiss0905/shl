# SHL Assessment Recommender

A conversational agent that recommends SHL Individual Test Solutions through
dialogue: clarifies vague requests, recommends a grounded shortlist, refines
it as constraints change, and answers comparison questions -- all backed by
a real (scraped) SHL catalog, never invented facts.

## How it works (short version)

- **Catalog**: 370 Individual Test Solutions (Job Solutions bundles excluded
  per the assignment), loaded from `data/shl_catalog.json`. At startup the
  app also tries to refresh from SHL's live JSON endpoint and falls back to
  the bundled copy on any failure (network error, bad shape, etc).
- **Retrieval**: BM25 lexical search (`app/retrieval.py`) over
  name/description/category text, plus structured filters (test type, job
  level, duration, language, remote-only).
- **Agent**: Gemini 2.5 Flash with function calling (`app/agent.py`). The
  model can call `search_catalog` / `get_item_details` to ground itself,
  then must call `respond` with a short reply + a list of *identifiers*
  (not free text) for its recommendations. The server resolves every
  identifier against the real catalog and silently drops anything that
  doesn't match -- the model physically cannot get a fabricated name or URL
  into the final response. The markdown table shown to the user is built
  server-side from the resolved catalog rows.
- **API**: FastAPI, `GET /health` and `POST /chat`, matching the assignment
  spec's exact request/response schema (see `app/schemas.py`).

See `APPROACH.md` for the full design write-up (trade-offs, what didn't
work, eval results).

## Local setup

```bash
cd shl-recommender
python -m venv venv && source venv/bin/activate   # optional but recommended
pip install -r requirements.txt
cp .env.example .env   # then paste your GEMINI_API_KEY into .env
export $(cat .env | xargs)   # or use python-dotenv / your shell's env loading
uvicorn app.main:app --reload
```

Then:

```bash
curl http://localhost:8000/health

curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hiring a Java developer who works with stakeholders"}]}'
```

## Running the eval harness

Replays all 10 provided sample traces (user turns only) against your *real*
running agent and reports Recall@10 (against URLs mentioned anywhere in
each trace, as a proxy ground truth) plus hallucination/schema checks:

```bash
python scripts/eval_harness.py
```

This makes real Gemini API calls, so `GEMINI_API_KEY` must be set in your
environment first.

## Rebuilding the catalog

If you want to regenerate `data/shl_catalog.json` from a fresh scrape/export:

```bash
python scripts/build_catalog.py path/to/raw_catalog.json data/shl_catalog.json
```

## Deployment (Render)

1. Push this repo to GitHub.
2. In Render: New -> Web Service -> connect the repo -> it will detect
   `render.yaml` (Docker-based) automatically, or set the Dockerfile
   manually.
3. Set the `GEMINI_API_KEY` environment variable in Render's dashboard
   (never commit it).
4. Deploy. First `/health` call may take up to ~60-90s if the free-tier
   instance was asleep -- this is expected and the assignment explicitly
   allows for it.

Any other Docker-friendly free host (Fly.io, Railway, Hugging Face Spaces)
works the same way -- the app only needs `GEMINI_API_KEY` set and a port to
bind to.

## Project layout

```
app/
  main.py        FastAPI app, /health and /chat
  agent.py        Tool-calling orchestration, system prompt, hallucination guard
  catalog.py       Catalog loading (bundled + live refresh w/ fallback)
  retrieval.py      BM25 search index + structured filters
  llm_client.py     Thin Gemini REST client
  schemas.py        Pydantic request/response models (matches assignment spec exactly)
data/
  shl_catalog.json   Cleaned, bundled catalog snapshot (370 items)
scripts/
  build_catalog.py   Raw scrape -> cleaned catalog
  eval_harness.py    Replays sample traces against the live agent
traces/               Provided sample conversations (for reference/eval)
```
