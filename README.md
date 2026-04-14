# bytebell-skills

Claude Code plugin marketplace for the [ByteBell Knowledge Graph](https://github.com/ByteBell/bytebell-skills) MCP server.

Enables smart code search, graph traversal, and PDF retrieval across indexed repositories directly from Claude Code.

## Install

```sh
# 1. Register the marketplace (run once)
/plugin marketplace add ByteBell/bytebell-skills

# 2. Install the skill
/plugin install bytebell@skill
```

## Plugins

| Plugin | Description |
|--------|-------------|
| `bytebell` | Search, traverse, and retrieve from indexed code repos and PDF documents |

## What it enables

- `smart_search` — fused 9-channel search across all indexed repos
- `keyword_lookup` — find exact function/class names and trace dependencies
- `graph_traverse` — browse repo structure when you don't know what to search for
- `get_repo_hubs` — find entry points by PageRank
- `retrieve_file` — get file metadata + targeted content sections
- `retrieve_pdf_page` — read indexed PDF documents
- `cypher` — raw Neo4j queries for advanced traversals

## Requirements

ByteBell Knowledge Graph MCP server must be configured in your Claude Code settings.
