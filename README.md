# Document Summariser

An async FastAPI service that summarises text and PDF documents using Claude, with concurrent chunked processing for long documents.

## Features

- **POST /summarize** — summarise raw text submitted as JSON
- **POST /uploadfile/** — upload a PDF or `.txt` file for summarisation
- Automatic chunking for long documents, with each chunk summarised concurrently, followed by a final synthesis pass for one coherent result
- Input validation at every entry boundary (file size, content type, text length)

## Architecture
```
Request (JSON text or file upload)
│
▼
Validation (size, type, length)
│
▼
[If PDF] Text extraction (pypdf)
│
▼
chunk_text() — splits on paragraph boundaries,
with a hard-split fallback for oversized paragraphs
│
▼
Per-chunk summarisation (concurrent, via asyncio.gather)
│
▼
Final reduce pass — combines chunk summaries into
one coherent result
│
▼
SummaryResponse (JSON)
For long documents, chunk summarisation happens **concurrently** rather than sequentially — each chunk is an independent Claude API call, so there's no reason to wait for one to finish before starting the next.
```

## Tech stack

- **FastAPI** — async web framework
- **Anthropic SDK** (`AsyncAnthropic`) — Claude API client, Haiku model
- **pypdf** — PDF text extraction
- **Pydantic** — request/response validation

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a `.env` file in this folder:
ANTHROPIC_API_KEY=your-key-here

## Running

```bash
uvicorn main:app --reload
```

Interactive API docs (Swagger UI): `http://127.0.0.1:8000/docs`

## Example usage

**Summarise raw text:**
```bash
curl -X POST http://localhost:8000/summarize \
  -H "Content-Type: application/json" \
  -d '{"text": "..."}'
```

**Summarise an uploaded file:**
```bash
curl -X POST http://localhost:8000/uploadfile/ \
  -F "file=@document.pdf;type=application/pdf"
```

## Design decisions and cost trade-offs

Documents are chunked and summarised concurrently, then combined in a final synthesis pass (see [ADR-001](./adr/001-chunking-strategy.md) for the full reasoning).

This costs roughly 25-30% more in API usage than a single-shot summary, since it generates multiple partial summaries plus one combining pass. The trade-off is deliberate: chunking preserves per-section detail that a single very long summarisation call tends to lose. Output tokens cost significantly more than input tokens on Claude's pricing, which is the main driver of the cost difference — chunking is a quality trade-off paid for in cost, not a cost optimisation.

**Known limitations:**
- Chunking is currently applied more aggressively than the model's actual context window requires — Claude Haiku 4.5 supports far larger single-request inputs than this project's per-chunk size. A future improvement would only chunk documents that approach the model's real context limit, reducing cost for small-to-medium documents while keeping the quality benefit for genuinely long ones.
- Size constants (`QUALITY_CHUNK_CHAR_LIMIT`, `MAX_TEXT_CHARS`, `MAX_BYTES`) are current development/testing values, not yet tuned against real quality or cost benchmarks.

## Possible future improvements

- Only chunk documents that approach the model's real context limit, rather than a low fixed threshold
- Prompt caching for the shared system prompt across chunk calls, to reduce repeated input cost
- Streaming progress updates for long-running requests (Server-Sent Events, paired with `asyncio.as_completed()` to preserve concurrency)
- Support for additional file types (`.docx`, `.md`)
- Configurable summary style (bullet points vs. prose, length)