# SCiXus: Architecture Deep Dive

Companion to `CASE_STUDY.md`. This goes module-by-module through the actual
data flow, from raw source content to a cited answer in Teams. Written from
the real implementation (private repo); no corpus content is reproduced here,
only the mechanics. I built and maintain every layer described below,
including ingestion, retrieval, the conversational layer, and the deployment.

## 1. Ingestion: three sources, three cleaning problems

| Source | Script | Format on disk | Core problem solved |
|---|---|---|---|
| Zendesk Help Center export | `ingest.py` | JSON export with `body_html` per article | Strip HTML to clean text while preserving structure (headings, list items, line breaks) as lightweight Markdown so chunk text stays readable outside a browser |
| STORIS Academy (Rise/Articulate courses) | `ingest_academy.py` | Captured course JSON, sometimes with content base64-encoded inside a `__resolveJsonp(...)` call in a locale file | Decode the JSONP/base64 payload, walk an arbitrarily nested course tree in reading order, and separate real lesson content from UI chrome ("Continue," "Knowledge Check," quiz feedback strings) |
| STORIS Academy Training Guide PDFs | `ingest_pdf.py` | PDF workbooks in `academy_guides/` | Strip repeated legal/confidentiality boilerplate and page furniture that appears on every page, without losing real content that happens to contain similar-looking words |

All three converge on the same output shape: a chunk record with
`chunk_id`, `article_id`, `title`, `url`, `category`, `section`, `source`
(`zendesk` or `academy`), `interface` (`ERP` / `NextGen` / `ERP+NextGen`), and
`text`. That common shape is what lets the retriever and the answer-generation
metrics treat all three sources uniformly.

### 1.1 Zendesk path (`ingest.py`)

- `html_to_text()` uses BeautifulSoup/lxml, explicitly removing `<img>` tags
  (no point keeping broken alt-text references) and converting heading tags to
  `#`-prefixed Markdown lines and `<li>` to `- ` bullets *before* extracting
  plain text, so structural cues survive into the chunk text rather than being
  flattened away.
- `chunk_text()` splits on blank-line-delimited blocks and greedily packs them
  up to `max_chars=1500`, carrying the last `overlap=200` characters of a
  chunk forward into the next one so a concept split across a chunk boundary
  still has some shared context on both sides.
- A per-corpus stats pass (`corpus_stats.json`) tracks category counts, empty
  articles, and, notably, how many article titles exist in both the
  outgoing "10.7" category and the current "10.8/11.0" set (1,986 at last
  count), which is the number that motivated the revision-down-ranking logic
  in the retriever rather than a naive delete-the-old-version approach.

### 1.2 Academy course path (`ingest_academy.py`)

Two input shapes are handled:

- **Rise raw export**: a `locales/en.js` file per module containing
  `__resolveJsonp("...", "<base64>")`. The base64 payload decodes to the full
  course JSON. `collect_text()` recursively walks the tree, pulling any string
  value under a fixed set of content-bearing keys (`html`, `paragraph`,
  `text`, `title`, `heading`, `caption`, `subtitle`, `listItem`,
  `items_text`) and skipping everything else; this avoids hand-writing a
  schema for every possible Rise block type.
- **DOM-capture fallback**: when a module was captured as rendered text
  instead of raw JSON, a large noise-pattern regex filters out interactive-UI
  strings (button labels, quiz feedback, progress indicators, "Lesson X of Y,"
  byline patterns) that would otherwise pollute the chunk text.

Both paths de-duplicate consecutive repeated fragments (common in click-through
courses where a heading re-renders per interaction) and apply a simple
substring heuristic (`"nextgen"` / `"next gen"` in the combined title+body,
also checking for `"erp"`) to tag each chunk's `interface`, since courses don't
carry that metadata explicitly.

### 1.3 PDF training guide path (`ingest_pdf.py`)

- Text extraction via `pdfplumber`, page by page.
- A noise regex drops lines matching known page furniture: page-number
  markers, "CONFIDENTIAL INFORMATION," "PROPRIETARY INFORMATION," copyright
  lines, the "Consulting Services | STORIS" footer, trade-secret legal
  boilerplate, and a specific watermark string, all of which repeat on every
  page and would otherwise dominate chunk text if left in.
- Dotted table-of-contents leaders (`"Heading ...... 5"`) are stripped with a
  trailing-dots regex rather than dropping the whole line, since the heading
  text itself is often useful context.
- Guides are explicitly preferred over equivalent Rise course content where
  both exist, since the PDF export is a cleaner capture than a click-through
  course DOM.

## 2. Indexing and retrieval (`retriever.py`)

- Chunks are tokenized (`[a-z0-9]+`, lowercased) and indexed with
  `rank_bm25.BM25Okapi`. The index is a single pickle (`bm25_index.pkl`)
  containing both the BM25 object and the raw doc list, rebuilt at deploy time
  (see §4) rather than committed, since it's large and fully derived from
  `chunks.jsonl`.
- **Query expansion**: before tokenizing a query, `_expand()` scans it against
  a fixed `SYNONYMS` dict of STORIS-specific phrase mappings (e.g. `"write a
  sale"` → `"enter sales order point of sale"`) and appends any matches to the
  query text. This is a deliberately simple substring match, not a learned
  model: cheap, auditable, and easy to extend by adding a dictionary entry
  when a new vocabulary gap is found (e.g. via the knowledge-gap log).
- **Ranking**: `retrieve()` scores all chunks, then walks the top 400 by raw
  BM25 score applying two adjustments before taking the final top-k:
  1. **Revision down-ranking**: any chunk categorized under the outgoing
     "10.7" doc set has its score multiplied by 0.80, so a current-revision
     article with a similar score wins.
  2. **Title-based dedup**: chunks are deduped by a normalized (lowercased,
     punctuation-stripped) title, keeping only the highest-scoring chunk per
     title. This collapses cross-listed FAQs and the 10.7/current pairs into
     a single result rather than showing near-duplicate hits.

## 3. Conversational layer (`assistant.py`)

This is the channel-agnostic core called by both `teams_bot.py` and `app.py`.
`answer_turn(history, user_text, images)` does, in order:

1. **Screenshot understanding** (if an image is attached): a cheap vision call
   asks Claude to name the visible screen, list field/button/error text, and
   emit a `KEYWORDS:` line for retrieval; the human-readable description and
   the keyword line are parsed apart from a single response.
2. **Query construction**: folds together the current message, any screenshot
   keywords, the previous user turn (so a follow-up like "I only see the
   customer fields" still retrieves in context), and a cheap Claude call that
   rewrites the whole thing into canonical STORIS terminology
   (`_normalize_query`) before hitting the retriever.
3. **Retrieval**: calls `retrieve()`, builds a bounded context block per hit
   (title, category, URL, chunk text), and separately computes per-turn
   analytics (`interface` mode of the hits, academy vs. zendesk hit counts,
   top score) that get logged downstream regardless of whether the answer
   itself succeeds.
4. **Answer generation**: sends up to the last 8 turns of history plus the
   current message (with any images) to Claude under a system prompt that
   hard-constrains it to the retrieved context, instructs concise numbered
   steps, defines the STORIS sales-order flow (Customer → Merchandise →
   Fulfillment → Payment) so the model can reason about "which step are they
   probably on" during troubleshooting, and defines the `<ESCALATE>` protocol
   for when the docs don't cover the situation.
5. **Escalation extraction**: the tag is stripped from the visible answer but
   recorded as a boolean, which becomes the `knowledge_gap` signal in the
   logging pipeline described in the case study.

Running without `ANTHROPIC_API_KEY` degrades gracefully to a retrieval-only
mode (returns the matched article titles instead of a generated answer);
useful for verifying the retrieval side in isolation.

## 4. Deployment topology

- **Hosting**: Render (`render.yaml`), Standard plan (2GB), sized because the
  loaded BM25 index measures ~466MB, which would OOM a 512MB instance. The
  build command reinstalls dependencies and calls `retriever.build_index()`
  fresh on every deploy rather than committing the ~43MB pickle to git.
- **Teams integration**: an Azure Bot resource (App Registration + Bot
  Framework channel registration) whose messaging endpoint points at the
  Render service's `/api/messages`. The async `CloudAdapter`/`aiohttp` bot
  (`teams_bot.py`) was specifically chosen over a simpler outgoing-webhook
  integration because Teams' webhook route enforces a 5-second response
  budget that multi-step reasoning plus image handling can't reliably meet.
- **Image auth quirk**: Teams-hosted image attachments require the bot's own
  bearer token to fetch (anonymous fetches 401), but inline `data:` URIs don't.
  `_download_images()` handles both, and additionally sniffs the real image
  format from magic bytes rather than trusting the attachment's declared
  content-type, since Teams doesn't always report it accurately.
- **Secrets**: all credentials (`MicrosoftAppId`, `MicrosoftAppPassword`,
  `ANTHROPIC_API_KEY`, `ESCALATION_CONTACT`, `LOG_WEBHOOK_URL`) are read from
  environment variables with no defaults baked into code, and are configured
  as Render's `sync: false` env vars (never committed).
- **Logging**: `teams_bot.py` fires-and-forgets a JSON POST per turn (and per
  feedback click) to a Power Automate HTTP-trigger webhook, which writes into
  a SharePoint Excel table. A scheduled flow later summarizes that table with
  an LLM and emails a weekly digest. Logging failures are caught and swallowed
  so a webhook outage never breaks the bot's actual response.

## 5. What would change first for a v2

In priority order, based on the limitations already called out in code
comments and deploy notes:

1. Replace or hybridize the BM25 scorer with embedding similarity;
   `retrieve()`'s interface is already stable for this swap.
2. Move conversation state out of in-process `MemoryStorage` into Azure Blob
   or Cosmos DB so state survives restarts and horizontal scaling.
3. Automate the corpus refresh (currently a manual re-export + re-run of the
   ingest scripts) and regenerate `corpus_stats.json` as part of that job so
   it can't silently go stale again.
4. Grow `eval.py` from a fixed 12-question list into something sourced from
   the real `knowledge_gap` log, and add it to CI.
