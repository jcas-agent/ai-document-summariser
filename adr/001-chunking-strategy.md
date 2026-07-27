# ADR-001: Chunking Strategy for Long Document Summarization

## Context

Claude's API has practical limits on how much text is sensible to send
in a single request — both a hard context-window ceiling and a softer
quality concern (very long inputs tend to produce vaguer, less detailed
summaries than focused ones). Documents like academic papers or reports
can be long enough that a single-shot summary loses section-level detail.

## Constraints (real vs. anticipated)

- **Real:** real-world PDFs commonly extract to tens of thousands of
  characters; single-shot summarization of long, multi-section documents
  tends to lose detail compared to summarizing sections individually.
- **Real:** PDF text extraction doesn't reliably preserve paragraph
  structure. An early implementation using a literal `"\n\n"` string
  split detected only 2 "paragraphs" in a real 27-page test PDF; a
  whitespace-tolerant regex split (`\n\s*\n`) correctly detected 9. A
  hard-split fallback is also needed for any single paragraph too large
  to fit in one chunk on its own, since some PDFs produce little to no
  real paragraph structure at all.
- **Anticipated, not current:** high request volume / cost at
  production scale. Not a real constraint for a portfolio-scale project,
  so current size thresholds were set for development/testing
  convenience rather than tuned against real quality/cost benchmarks
  (see "Known limitations" below).

## Options considered

1. **Reject long documents outright.** Simple, but fails on exactly the
   documents most worth summarizing.
2. **Chunk and summarize sequentially, then combine.** Correct, but
   wastes the concurrency the API and `asyncio` naturally support, since
   each chunk's summary has no dependency on any other chunk.
3. **Chunk and summarize concurrently, then a final synthesis pass
   (map-reduce).** Chosen.

## Decision

Split input into chunks on paragraph boundaries, with a fallback for any
single paragraph too large to fit a chunk on its own. Summarize all
chunks **concurrently** via `asyncio.gather()`, then run one additional
pass to **combine** the chunk summaries into a single coherent result,
rather than just concatenating fragments.

The per-chunk size constant is named `QUALITY_CHUNK_CHAR_LIMIT` rather
than a generic "chunk size" — its value is chosen for summary quality,
not because the model requires input this small. Claude Haiku 4.5's
actual context window (200K tokens, roughly 600-800K characters using a
conservative chars-per-token estimate) is far larger than the current
per-chunk size; chunking at this threshold is a deliberate quality
choice, not a technical necessity, at the document sizes this project
currently supports.

Validation happens once, at the input boundary (byte size for uploads,
character count for text), rather than per-chunk — an earlier version
validated a minimum length per-chunk and incorrectly failed an entire
batch when one short trailing chunk was valid on its own but under the
per-request minimum. Per-chunk validation only needs an upper bound;
only the original document needs a lower one.

## Consequences

- Each request costs at least 2 API calls (one or more chunk summaries,
  plus 1 combine pass) instead of 1. Concretely, for a 50,000-character
  document split into 2 chunks: a single-shot call costs roughly $0.014
  (Haiku 4.5 rates), while the chunked approach costs roughly $0.018 —
  about 25-30% more, driven mainly by generating 3 summaries' worth of
  output instead of 1, and output tokens costing ~5x input tokens.
  Chunking is a **quality trade-off paid for in cost, not a cost
  optimization** — this is worth being explicit about, since it's the
  opposite of what "splitting work into smaller pieces" often implies.
- Concurrency keeps wall-clock time close to the slowest single chunk,
  not the sum of all chunks.
- **Known limitation, not yet addressed:** the system currently chunks
  every document above a low threshold, regardless of whether it's
  actually close to the model's real context ceiling. A more
  cost-efficient version would only chunk when a document approaches
  the model's actual context limit, and process smaller documents in a
  single call. Identified during design review but not implemented, to
  keep scope focused on the current project's goals.
- **Current constant values are provisional.** `QUALITY_CHUNK_CHAR_LIMIT`,
  `MAX_TEXT_CHARS`, and `MAX_BYTES` were set during active development
  for testing convenience and have not been tuned against real
  quality/cost benchmarks. They should be revisited — ideally backed by
  actual comparison of summary quality at different chunk sizes — before
  treating current thresholds as a considered design choice.
- Incremental progress feedback for long-running requests was
  considered (concurrent processing via a queue, paired with
  Server-Sent Events, using `asyncio.as_completed()` to preserve
  concurrency while surfacing per-chunk completion) but scoped out as
  unnecessary complexity for a portfolio-scale project without real
  concurrent users.