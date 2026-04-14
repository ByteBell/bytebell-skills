---
name: bytebell
description: >
  ByteBell Knowledge Graph — search, traverse, and retrieve from indexed
  code repos and PDF documents. TRIGGER when using: smart_search, graph_search,
  graph_traverse, keyword_lookup, get_repo_hubs, retrieve_file, retrieve_pdf_page, cypher.
user-invocable: true
argument-hint: "[search query or task description]"
---

# ByteBell Knowledge Server

## Graph Model (Flat Folder)

7 node types: Knowledge → RepoSummary, FolderNode/FolderVersion, FileNode/FileVersion, OrgKeyword.
No levels, no batches. ALL array fields are OrgKeyword nodes → APPEARS_IN_FILE → FileVersion.
Omit knowledgeId to search ALL repos. knowledgeId is ALWAYS a UUID — never a repo name or slug.

## Tools

| Tool              | When                                                             |
| ----------------- | ---------------------------------------------------------------- |
| list_knowledge    | First call — get available repos + their knowledgeId UUIDs       |
| smart_search      | Default search — fuses 9 channels in one call                    |
| graph_search      | Need specific channels (similar, integration, business, folders) |
| keyword_lookup    | Find exact function/class names, cross-repo dependency discovery |
| graph_traverse    | Browse repo structure when you don't know what to search for     |
| get_repo_hubs     | Find entry points + core files by PageRank                       |
| retrieve_file     | Get file metadata + content (ALWAYS metadata first)              |
| retrieve_pdf_page | Get PDF page metadata + content                                  |
| cypher            | Raw Neo4j query when standard tools can't express the query      |

## Which Skill to Read Next

| Your task                                         | Read                      |
| ------------------------------------------------- | ------------------------- |
| Search for code, find files, investigate a bug    | bytebell-code-search.md   |
| Explore unknown codebase, understand architecture | bytebell-graph-explore.md |
| Search or read PDF documents                      | bytebell-pdf.md           |
| Custom graph queries, complex traversals          | bytebell-cypher.md        |
| Anything tied to a specific commit/version        | bytebell-commit-aware.md  |

## Always-On Guardrails (from 132-task empirical analysis)

1. **knowledgeId is a UUID** — never guess from repo name. Call list_knowledge first.
2. **Metadata before content** — ALWAYS call retrieve_file(metadata) before retrieve_file(content). Read section_map, then target specific lines. (Prevents 40-60% of wasted calls)
3. **Max 3 consecutive retrieve_file calls** — then STOP and re-orient with smart_search or keyword_lookup. (Prevents 50-call spirals)
4. **Never call** save_conversation_history or webfetch during code analysis. All code is in ByteBell tools. Use submit_feedback to report genuine tool issues — it helps improve future explorations, routing explorations or comments on file codes to guide future explorations. (Prevents 20% of off-platform tool misuses)
5. **source_type routing** — "code" → retrieve_file, "pdf" → retrieve_pdf_page. Never cross them.
