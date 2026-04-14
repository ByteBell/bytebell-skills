# ByteBell PDF Documents

Use this skill when searching or reading indexed PDF documents. PDFs use a different graph model and different retrieval tools than code — **never cross them** (`retrieve_file` for code, `retrieve_pdf_page` for PDFs).

## How PDF Results Appear

PDF results arrive through two paths:
- **smart_search** — automatically includes PDF results alongside code results. Look for `source_type: "pdf"` in results, with a `pdf` sub-object containing `page_number`, `pdf_title`, `chapter_title`, `summary`.
- **search_pdf** — dedicated PDF search with three parallel tiers (graph, fulltext, vector), fused via Reciprocal Rank Fusion. More powerful for PDF-specific queries.

**Routing rule:** When you see `source_type: "pdf"` in any search result → use `retrieve_pdf_page`. When you see `source_type: "code"` → use `retrieve_file`. Never cross them.

## The PDF Search → Read Pattern

```
smart_search / search_pdf → retrieve_pdf_page(metadata) → retrieve_pdf_page(content) → answer
```

### Step by step:

1. **Search** — `smart_search({ query })` or `search_pdf({ query, knowledgeId? })`. Returns node_ids, page_numbers, summaries, chapter info.
2. **Verify** — `retrieve_pdf_page({ operation: "metadata", knowledgeId, nodeIds: [...] })`. Returns purpose, summary, headlines, entities, definitions, questions_answered, `hasImages` flag. Verify these are the pages you actually need.
3. **Read** — `retrieve_pdf_page({ operation: "content", knowledgeId, nodeIds: [...] })`. Returns verbatim text excerpts + definitions + image paths. This is the actual page content.
4. *(optional)* **Images** — if `hasImages: true` in metadata, call `retrieve_image({ knowledgeId, nodeIds: [...] })` to get base64-encoded images with descriptions.

**Never use the summary alone as the final answer** — always read the actual content via step 3.

## PDF Graph Model

```
Knowledge (type: "PDF", pdf_title)
  └─ ChapterNode { title, summary, start_page, end_page, keywords, topics }
       └─ PageNode { page_number, purpose, summary, headlines, entities,
                     definitions, keywords, questions_answered, continuation_status,
                     images (JSON), pdf_title }
            ├─ VerbatimExcerptNode { text }     ← the actual quoted text
            ├─ DefinitionNode { text }           ← extracted definitions
            └─ NEXT_PAGE → adjacent PageNode     ← for context windowing
```

OrgKeywords for PDFs have `content_type: "pdf"` and link via `APPEARS_IN_PAGE` (not APPEARS_IN_FILE).

## retrieve_pdf_page Operations

### metadata — Verify before reading
```
retrieve_pdf_page({ operation: "metadata", knowledgeId, nodeIds: ["id1", "id2"] })
```
Or by page range: `{ operation: "metadata", knowledgeId, fromPage: 1, toPage: 10 }` (max 10 pages)

Returns per page: purpose, summary, headlines, definitions, entities, keywords, topics, questions_answered, continuation_status, `hasImages` flag, chapterTitle.

### content — Get verbatim text
```
retrieve_pdf_page({ operation: "content", knowledgeId, nodeIds: ["id1", "id2"] })
```
Or by page range: `{ operation: "content", knowledgeId, fromPage: 5, toPage: 9 }` (max 5 pages)

Returns per page: verbatim text excerpts, definitions, image paths with descriptions, summary for context.

### chapter — Browse chapter structure
```
retrieve_pdf_page({ operation: "chapter", knowledgeId, chapterTitle: "Introduction" })
```

Returns: chapter summary, start/end page, and all page summaries within the chapter. Use this to understand chapter scope before diving into individual pages.

## search_pdf: Dedicated PDF Search

More powerful than smart_search for PDF-specific queries. Runs three parallel search tiers:

| Tier | What it searches | Best for |
|------|-----------------|----------|
| graph | Chapter keywords/topics, OrgKeywords, entity arrays, questions_answered | Structured lookups ("definitions of X", "chapter about Y") |
| fulltext | PageNode + ChapterNode purpose/summary, VerbatimExcerpt text, Definition text | Exact phrase matching |
| vector | Pinecone semantic embeddings | Conceptual/natural language questions |

```
search_pdf({ query: "machine learning fundamentals", knowledgeId?, topK: 10, windowSize: 1 })
```

- `windowSize` (default 1, max 5) — includes ±N neighbor pages around each result for context
- `tiers` — optionally limit to specific tiers: `["graph", "fulltext", "vector"]`
- Omit `knowledgeId` to search across ALL PDFs in the org

## retrieve_image: Get Page Images

After confirming `hasImages: true` in metadata:

```
retrieve_image({ knowledgeId, nodeIds: ["page_node_id"] })
```

Returns base64-encoded image data with descriptions. Max 5 nodeIds per call. Supports PNG, JPG, GIF, WebP, SVG, BMP.

## Limits

| Operation | Max per call |
|-----------|-------------|
| metadata (by page range) | 10 pages |
| content (by page range) | 5 pages |
| retrieve_image | 5 nodeIds |
| search_pdf topK | 50 results |

## Quick Reference

| Question | Tool call |
|----------|-----------|
| Find PDF docs about a topic | `smart_search({ query: "..." })` — PDF results auto-included |
| Search only PDFs | `search_pdf({ query: "...", knowledgeId? })` |
| Search a specific PDF | `smart_search({ query: "...", knowledgeId: "<pdf-id>" })` |
| Verify pages before reading | `retrieve_pdf_page({ operation: "metadata", knowledgeId, nodeIds })` |
| Read actual page text | `retrieve_pdf_page({ operation: "content", knowledgeId, nodeIds })` |
| Browse a chapter | `retrieve_pdf_page({ operation: "chapter", knowledgeId, chapterTitle: "..." })` |
| Get images from a page | `retrieve_image({ knowledgeId, nodeIds })` — check `hasImages` first |
| Identify PDF knowledge bases | `list_knowledge` — look for entries with `type: "PDF"` |

## Chapter-Based Exploration

When you need to understand the structure of a PDF before diving in:

1. **search_pdf({ query })** — results include `chapter.title`, `chapter.start_page`, `chapter.end_page`
2. **retrieve_pdf_page({ operation: "chapter", chapterTitle: "..." })** — get all page summaries in that chapter
3. Pick the most relevant pages from the chapter listing
4. **retrieve_pdf_page({ operation: "content", nodeIds: [selected pages] })** — read those specific pages
