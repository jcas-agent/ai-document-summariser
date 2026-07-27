# Document Summariser

An async FastAPI service that summarises text and PDF documents using Claude, with concurrent chunked processing for long documents.

## Features

- **POST /summarize** — summarise raw text submitted as JSON
- **POST /uploadfile/** — upload a PDF or `.txt` file for summarisation
- Automatic chunking for long documents, with each chunk summarised concurrently, followed by a final synthesis pass for one coherent result
- Input validation at every entry boundary (file size, content type, text length)

## Architecture
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

## Design decisions

See [`adr/`](./adr) for detailed architecture decision records, including the chunking strategy and validation approach.

## Possible future improvements

- Streaming responses for real-time progress on long documents
- Support for additional file types (`.docx`, `.md`)
- Configurable summary style (bullet points vs. prose, length)
