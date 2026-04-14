# ByteBell Graph Exploration

Use this skill when you're entering an unfamiliar codebase, need to understand architecture, or are investigating a bug that spans multiple files. The key insight: **orient first, search second** — models that jumped straight to smart_search on unknown codebases made 2-3x more calls than those that oriented first.

## When to Use Exploration (vs Code Search)

| Situation | Use graph-explore | Use code-search |
|-----------|-------------------|-----------------|
| "What does this repo do?" | Yes — start with repo + hubs | No |
| "How is the project structured?" | Yes — folder_tree | No |
| "Find where auth is handled" | No | Yes — smart_search |
| "This bug touches 5+ files" | Yes — map dependencies first | Then targeted retrieval |
| "Which modules depend on each other?" | Yes — folder_dependency_graph | No |
| "I keep finding irrelevant files" | Yes — step back and orient | Then resume search |

## First Contact: The 4-Step Orientation

When you've never seen this codebase before:

```
list_knowledge → get_repo_hubs → graph_traverse(repo) → graph_traverse(folder_tree) → [ready to search]
```

1. **list_knowledge** — what repos are available? Get knowledgeId UUIDs.
2. **get_repo_hubs({ knowledgeId, topK: 10, sortBy: "importance" })** — PageRank-ranked entry points. Returns: path, purpose, business_context, classes, functions for each hub file. This tells you the structural backbone in one call.
3. **graph_traverse({ operation: "repo", knowledgeId })** — bird's-eye view via RepoSummary: purpose, dataFlow, keyPatterns, majorSubsystems. Tells you what the repo does and how data moves through it.
4. **graph_traverse({ operation: "folder_tree", knowledgeId, path: "/" })** — full recursive dir structure with purpose for each folder. Tells you where things live.

After these 4 calls, you know: what the repo does, which files are central, and how the folders are organized. NOW you can search effectively.

**Cap rule:** After orientation (get_repo_hubs + repo + folder_tree), start targeted retrieval. Don't keep exploring — you have enough context.

## graph_traverse Operations

| Operation | Returns | Use when | Example |
|-----------|---------|----------|---------|
| repo | RepoSummary: purpose, dataFlow, keyPatterns, majorSubsystems | First orientation — "what does this repo do?" | `graph_traverse({ operation: "repo", knowledgeId })` |
| folder | Direct children of a folder (subfolders + files with purpose) | Browsing a specific directory | `graph_traverse({ operation: "folder", knowledgeId, folderPath: "src/auth" })` |
| folder_tree | Full recursive subtree with purposes | Understanding a module's full structure | `graph_traverse({ operation: "folder_tree", knowledgeId, folderPath: "src/" })` |
| folder_dependency_graph | Import/dependency relationships between folders | Architecture analysis — which modules depend on what | `graph_traverse({ operation: "folder_dependency_graph", knowledgeId, folderPath: "src/" })` |

## get_repo_hubs: Find the Important Files

get_repo_hubs uses PageRank on the OrgKeyword graph — files sharing more high-frequency keywords rank higher. This identifies the files that most of the codebase depends on.

```
get_repo_hubs({ knowledgeId, topK: 10, sortBy: "importance" })
```

Returns for each hub file: path, purpose, business_context, classes, functions.

**sortBy options:**
- `"importance"` (default, recommended) — PageRank-based, considers structural centrality
- `"pagerank"` — raw PageRank score
- `"import_count"` — number of incoming imports

Use this BEFORE smart_search when exploring unfamiliar repos. The hub files are your best starting point for understanding architecture.

## Multi-File Bug Investigation

The hardest tasks in testing required code from 5+ interconnected files. Models that didn't map dependencies first ended up in 40-60 call spirals.

### The Exploration-First Pattern:

1. **keyword_lookup({ term: "BuggyClass", type: "HAS_CLASS" })** — find where the class/function lives and all files that reference it
2. **graph_traverse({ operation: "folder_dependency_graph", folderPath: "src/" })** — understand which modules the buggy file sits in and what depends on it
3. **keyword_lookup({ term: "BuggyClass", type: "HAS_IMPORT_EXTERNAL" })** — find all files that import it (the impact chain)
4. **retrieve_file(metadata)** ONLY on the files in the dependency chain — don't scan broadly

### vs The Wasteful Pattern:

```
smart_search → retrieve_file → retrieve_file → retrieve_file → ... (×30) → maybe found the right files
```

The exploration-first pattern maps the territory in 3-4 calls, then reads targeted files. The wasteful pattern reads files one-by-one hoping to stumble onto the right ones.

## Cross-Repo Exploration

When investigating how services interact across repos:

1. **graph_search({ query: "ServiceA", channels: ["integration"] })** — find what ServiceA exposes (API contracts, events, queues)
2. **For each surface:** graph_search({ query: surface_value, channels: ["integration"] }) across ALL repos (omit knowledgeId)
3. **Mirror pairs** — surface types come in producer/consumer pairs:
   - `api` ↔ `api_call`
   - `event_pub` ↔ `event_sub`
   - `queue_write` ↔ `queue_read`
   - `PROVIDES_CONTRACT` ↔ `CONSUMES_CONTRACT`
4. **retrieve_file(metadata)** for each dependent → section_map → targeted content

Or use a single Cypher query to get the full contract map at once (see bytebell-cypher.md, Pattern 4).

## Exploration → Search Transition

Don't over-explore. The point of exploration is to make your subsequent searches precise.

| Calls spent exploring | Status | Next action |
|-----------------------|--------|-------------|
| 1-3 calls | Still orienting | Continue exploring |
| 4-5 calls | Should have enough context | Transition to targeted search/retrieval |
| 6+ calls | Over-exploring | STOP. Use what you know and start retrieving files |

After exploration, switch to bytebell-code-search.md patterns: smart_search or keyword_lookup with the knowledge you gained about the codebase structure.
