# ByteBell Code Search

## The Canonical Pattern (4-6 calls, handles 80% of queries)

The best-performing sessions consistently followed this exact sequence:

```
list_knowledge → smart_search → retrieve_file(metadata) → retrieve_file(content, targeted) → [keyword_lookup / graph_search if needed] → answer
```

### Step by step:

1. **list_knowledge** — get knowledgeId UUIDs (skip if already known from prior calls). Call this ONCE per session — data doesn't change mid-session.
2. **smart_search({ query, knowledgeId?, commitHash? })** — find relevant files. Returns path, repo_name, knowledge_id, score, matched_channels.
3. **retrieve_file({ operation: "metadata", knowledgeId, relativePath })** — get section_map, classes, functions, imports, line_count. Read section_map to find exact line ranges.
4. **retrieve_file({ operation: "content", knowledgeId, relativePath, fromLine, toLine })** — fetch ONLY the section you need from section_map. Never fetch full file blindly.
5. *(optional)* **keyword_lookup({ keyword, types })** — if you need to trace dependencies, find call sites, or map which other files use a class/function discovered in step 3-4.
6. *(optional)* **graph_search({ query, channels })** — if smart_search results were insufficient, use specific channels (similar, integration, semantic) for a targeted second pass.

### What went wrong without this pattern:

| Pattern | Calls | Problem |
|---------|-------|---------|
| Canonical | 4-6 | Efficient — search, verify, read targeted |
| retrieve_file spiral | 20-62 | Skipped metadata, scanned with overlapping ranges |
| Wrong knowledgeId | +2-4 wasted | Guessed repo name instead of UUID, 29 failures |

## knowledgeId: NEVER Guess It

knowledgeId is ALWAYS a UUID like `"eb768039-0fd9-4051-bf82-44ce31015d78"`.
It is NEVER a repo name (`"astropy"`), task ID (`"astropy__astropy-14096"`), or slug.

If you don't have the UUID, call `list_knowledge` first. It's cheap (<100ms).

## smart_search: Default Starting Point

Fuses 9 channels in one call: purpose, classes, functions, imports, keywords, paths, semantic, integration, business.

- Free-text queries → smart_search (always)
- Concept/pattern queries → smart_search
- Omit knowledgeId to search ALL repos (cross-repo analysis)
- Add `path` prefix to scope within monorepo packages (e.g. `"packages/auth"`)
- Use `exclude` to filter: `["tests", "vendor", "config", "generated", "docs", "build"]`

## keyword_lookup: Precision Dependency Mapping

keyword_lookup is the most precise search tool — finds an exact OrgKeyword by name + type and returns ALL files linked to it across every repo. From the exploration data: GPT-5.4-mini and nano used it heavily (28-38 calls, 9-10% of total) for effective cross-repo dependency tracing.

**Prefer keyword_lookup over smart_search/graph_search when:**
- You know the exact class/function/import name
- You need ALL files using something, not just top-N ranked results
- You're mapping a dependency chain or finding call sites
- You're doing cross-repo impact analysis

**Examples:**
- "Find all files that use AuthService" → `keyword_lookup({ keyword: "AuthService", types: ["HAS_CLASS"] })`
- "Which files import express?" → `keyword_lookup({ keyword: "express", types: ["HAS_IMPORT_EXTERNAL"] })`
- "What consumes the /api/users endpoint?" → `keyword_lookup({ keyword: "POST /api/users", types: ["CONSUMES_CONTRACT"] })`
- "Find all event subscribers for user.created" → `keyword_lookup({ keyword: "user.created", types: ["HAS_INTEGRATION_SURFACE"] })`
- "Where is validateToken defined?" → `keyword_lookup({ keyword: "validateToken", types: ["HAS_FUNCTION"] })`

**Valid types (exact strings only):**
HAS_CLASS, HAS_FUNCTION, HAS_IMPORT_EXTERNAL, HAS_IMPORT_INTERNAL, HAS_KEYWORD, HAS_ONTOLOGY_CONCEPT,
HAS_BUSINESS_ENTITY, HAS_SYSTEM_CAPABILITY, HAS_SIDE_EFFECT, HAS_CONFIG_DEPENDENCY,
HAS_INTEGRATION_SURFACE, PROVIDES_CONTRACT, CONSUMES_CONTRACT, HAS_DATA_FLOW_DIRECTION

## graph_search: Channel-Level Control

Use graph_search when you need a specific channel that smart_search doesn't expose individually, or when you need per-channel results (not fused). 12 channels:

| Channel     | Searches                                                  | Example query              |
| ----------- | --------------------------------------------------------- | -------------------------- |
| purpose     | FileNode purpose field (AI description)                   | "authentication middleware" |
| classes     | HAS_CLASS OrgKeywords                                     | "AuthController"           |
| functions   | HAS_FUNCTION OrgKeywords                                  | "validateToken"            |
| imports     | HAS_IMPORT_EXTERNAL OrgKeywords                           | "express"                  |
| keywords    | HAS_KEYWORD OrgKeywords                                   | "rate-limiting"            |
| paths       | FileNode relative_path field                              | "src/auth/"                |
| semantic    | Pinecone vector similarity                                | "how does login work"      |
| glob        | Path pattern matching                                     | "*.test.ts"                |
| integration | PROVIDES/CONSUMES_CONTRACT, HAS_INTEGRATION_SURFACE       | "POST /api/users"          |
| business    | HAS_BUSINESS_ENTITY, HAS_SYSTEM_CAPABILITY                | "payment processing"       |
| folders     | FolderNode purpose + path                                 | "middleware directory"     |
| similar     | Find files related to a known file                        | "/src/auth/jwt.ts"         |

## When to Use Which Search Tool

| Scenario | Tool | Why |
|----------|------|-----|
| Free-text "find something about X" | smart_search | Fuses 9 channels, best for vague queries |
| Exact class/function name | keyword_lookup | Returns ALL files, not ranked top-N |
| "All files importing package Y" | keyword_lookup (HAS_IMPORT_EXTERNAL) | Complete list across repos |
| "Files related to this file" | graph_search (similar channel) | Similarity analysis |
| "Find test files matching *.spec.ts" | graph_search (glob channel) | Pattern matching |
| Cross-service API boundaries | keyword_lookup (PROVIDES/CONSUMES_CONTRACT) | Mirror pair discovery |
| "How does authentication work?" | smart_search or graph_search (semantic) | Conceptual/semantic |

## retrieve_file: The Three-Step Rule (MANDATORY)

From empirical data: 94-96% of retrieve_file calls were blind `content` reads. Only 3-6% used `metadata` first. This caused the #1 cost driver — retrieve_file spirals (229 consecutive triplets in worst run).

**Step 1:** `retrieve_file({ operation: "expand" })` or `retrieve_file({ operation: "metadata" })`
  → Returns: purpose, classes, functions, imports, section_map, line_count
  → Read section_map to find exact line ranges for what you need

**Step 2:** `retrieve_file({ operation: "content", fromLine, toLine })`
  → Fetch ONLY the section you need. Never fetch full file blindly.

**Step 3:** If you need another section, use section_map again. Don't scan with overlapping ranges.

**bulk_search:** Use `retrieve_file({ operation: "bulk_search", search: "pattern" })` to search within a file's content before fetching — avoids blind line-range scanning.

## After 3 Consecutive retrieve_file Calls: STOP

Re-orient with smart_search or keyword_lookup before continuing.
This prevents the 20-50 call retrieve_file spiral that costs 40-60% of wasted tokens.

Worst observed case: 54 consecutive retrieve_file calls (875 seconds) on a single task.

## Bug-Fix Pattern (5 steps — prevents retrieve_file spirals)

When investigating a specific bug or failing test:

1. **Read the failing test first** — `retrieve_file(metadata)` on the test file, then `content` on the failing test function
2. **keyword_lookup** — find the function/class under test by name (`HAS_FUNCTION` or `HAS_CLASS`)
3. **retrieve_file(metadata)** on the source file — read section_map to locate the relevant function
4. **retrieve_file(content, fromLine, toLine)** — targeted read of the bug location
5. **Write the fix** — typically 1-5 lines changed

This 5-step pattern, used by the most efficient runs, prevents the 20-50 call spirals seen in less guided runs.

## Pivot Protocol: When Results Are Empty

1. smart_search returns 0 → try keyword_lookup with exact class/function name
2. keyword_lookup returns 0 → try graph_search with a different channel (semantic, integration)
3. graph_search returns 0 → graph_traverse(repo) to browse structure manually
4. Still nothing → omit knowledgeId (maybe it's in a different repo)

## Cross-Repo Search

Omit knowledgeId → searches ALL repos in the org.
Use keyword_lookup with PROVIDES_CONTRACT / CONSUMES_CONTRACT for API boundary tracing.
Use "integration" channel in graph_search to find cross-service surfaces (api ↔ api_call, event_pub ↔ event_sub).
Use "similar" channel after finding a key file to find related files across repos.
