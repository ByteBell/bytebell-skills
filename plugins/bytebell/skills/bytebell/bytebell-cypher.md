# ByteBell Cypher Queries

Use cypher when standard tools can't express the query — especially when you'd otherwise need 5+ tool calls to get the same answer. Cypher is read-only (no CREATE/MERGE/DELETE). `$orgId` is auto-injected — always use it in WHERE clauses.

## When to Reach for Cypher Instead of Standard Tools

| Situation | Without Cypher | With Cypher |
|-----------|---------------|-------------|
| Find which files share multiple keywords/classes | 3-4 keyword_lookup calls, manually intersect | 1 query with multi-match + GROUP BY |
| Map dependency chain between files | 5-8 retrieve_file calls reading imports blindly | 1 query traversing APPEARS_IN_FILE edges |
| Find all files related to a concept cluster | Multiple smart_search + graph_search retries | 1 query matching multiple OrgKeywords |
| Get section_map without full file read | retrieve_file(metadata) per file | 1 query returning section_map for multiple files |
| Version history for a file | Multiple retrieve_file calls with different commitHash | 1 query ordered by committed_at |

**Rule of thumb:** If you're about to make your 3rd search call on the same topic, stop and write a Cypher query instead.

## Graph Schema (7 node types)

```
Knowledge { knowledge_id, org_id, repo_name, display_name, branch_name, commit_hash }
RepoSummary { data_flow, key_patterns, major_subsystems, architecture, org_id, knowledge_id }
FolderNode { relative_path, purpose, org_id, knowledge_id }
  └─ FolderVersion { relative_path, purpose, commit_hash, committed_at }
FileNode { relative_path, purpose, summary, section_map, org_id, knowledge_id }
  └─ FileVersion { relative_path, purpose, summary, section_map, commit_hash, committed_at, change_type }
OrgKeyword { keyword, type, org_id, content_type, total_frequency, file_count }
  types: HAS_CLASS, HAS_FUNCTION, HAS_IMPORT_EXTERNAL, HAS_IMPORT_INTERNAL, HAS_KEYWORD,
         HAS_ONTOLOGY_CONCEPT, HAS_BUSINESS_ENTITY, HAS_SYSTEM_CAPABILITY,
         HAS_SIDE_EFFECT, HAS_CONFIG_DEPENDENCY, HAS_INTEGRATION_SURFACE,
         PROVIDES_CONTRACT, CONSUMES_CONTRACT, HAS_DATA_FLOW_DIRECTION
```

## Relationships

```
Knowledge -[:HAS_REPO_SUMMARY]-> RepoSummary
Knowledge -[:HAS_FOLDER]-> FolderNode
Knowledge -[:HAS_FILE]-> FileNode
FolderNode -[:CONTAINS_FOLDER]-> FolderNode
FolderNode -[:CONTAINS_FILE]-> FileNode
FolderNode -[:HAS_VERSION]-> FolderVersion
FileNode -[:HAS_VERSION]-> FileVersion
OrgKeyword -[:APPEARS_IN_FILE {frequency, commit_hash}]-> FileVersion
OrgKeyword -[:APPEARS_IN_FILE {frequency, commit_hash}]-> FileNode  (current-state edges)
OrgKeyword -[:APPEARS_IN_FOLDER {frequency, commit_hash}]-> FolderVersion
```

## Power Patterns: Replace Multi-Call Spirals

### Pattern 1: Find Files Containing Multiple Related Keywords

Instead of 3-4 sequential keyword_lookup calls to find where `ClassA`, `ClassB`, and `ClassC` co-exist:

```cypher
MATCH (k:OrgKeyword {org_id: $orgId})-[:APPEARS_IN_FILE]->(fv:FileVersion)
WHERE k.keyword IN ['ClassA', 'ClassB', 'ClassC']
  AND k.type IN ['HAS_CLASS', 'HAS_FUNCTION']
WITH fv.relative_path as path, count(DISTINCT k.keyword) as keyword_count,
     collect(DISTINCT k.keyword) as matched_keywords, fv.purpose as purpose
WHERE keyword_count >= 2
ORDER BY keyword_count DESC
RETURN path, keyword_count, matched_keywords, purpose
LIMIT 10
```

### Pattern 2: Map the Dependency Graph from a Starting File

Instead of reading 5-8 files blindly chasing imports:

```cypher
MATCH (fn:FileNode {org_id: $orgId})-[:HAS_VERSION]->(fv:FileVersion)
WHERE fn.relative_path CONTAINS $startFilePath
WITH fv, fn.relative_path AS startPath
MATCH (k:OrgKeyword {org_id: $orgId, type: 'HAS_IMPORT_EXTERNAL'})
      -[:APPEARS_IN_FILE]->(fv)
WITH k, startPath
MATCH (k)-[:APPEARS_IN_FILE]->(other_fv:FileVersion)
WHERE other_fv.relative_path <> startPath
RETURN DISTINCT other_fv.relative_path, other_fv.purpose,
       count(DISTINCT k) as shared_imports
ORDER BY shared_imports DESC
LIMIT 10
```

### Pattern 3: Find All Files in a Concept Cluster

When smart_search gives partial results and you'd otherwise retry with different queries:

```cypher
MATCH (k:OrgKeyword {org_id: $orgId})
      -[:APPEARS_IN_FILE]->(fv:FileVersion)
WHERE k.keyword =~ '(?i).*auth.*' OR k.keyword =~ '(?i).*token.*' OR k.keyword =~ '(?i).*jwt.*'
WITH fv.relative_path as path, count(DISTINCT k) as relevance,
     collect(DISTINCT k.keyword)[0..5] as top_keywords,
     fv.purpose as purpose
ORDER BY relevance DESC
RETURN path, relevance, top_keywords, purpose
LIMIT 15
```

### Pattern 4: Cross-Repo Contract Tracing

Instead of multiple graph_search(integration) calls across repos:

```cypher
MATCH (provider:OrgKeyword {org_id: $orgId, type: 'PROVIDES_CONTRACT'})
      -[:APPEARS_IN_FILE]->(prov_fv:FileVersion)
MATCH (consumer:OrgKeyword {org_id: $orgId, type: 'CONSUMES_CONTRACT', keyword: provider.keyword})
      -[:APPEARS_IN_FILE]->(cons_fv:FileVersion)
RETURN provider.keyword as contract,
       prov_fv.relative_path as provider_file,
       cons_fv.relative_path as consumer_file
ORDER BY contract
```

### Pattern 5: Version History + What Changed Between Commits

Instead of multiple retrieve_file calls with different commitHash values:

```cypher
MATCH (fn:FileNode {org_id: $orgId})-[:HAS_VERSION]->(fv:FileVersion)
WHERE fn.relative_path CONTAINS $filePath
RETURN fv.commit_hash, fv.committed_at, fv.summary, fv.change_type
ORDER BY fv.committed_at DESC
LIMIT 10
```

### Pattern 6: Find Hub Files by Keyword Density

Instead of get_repo_hubs + multiple graph_search calls to understand a module:

```cypher
MATCH (k:OrgKeyword {org_id: $orgId})-[:APPEARS_IN_FILE]->(fv:FileVersion)
WHERE fv.relative_path STARTS WITH $modulePath
WITH fv.relative_path as path, fv.purpose as purpose,
     count(DISTINCT k) as keyword_density
ORDER BY keyword_density DESC
RETURN path, purpose, keyword_density
LIMIT 10
```

### Pattern 7: Find Test Files for a Specific Class/Function

Instead of smart_search("test for X") which often returns noisy results:

```cypher
MATCH (k:OrgKeyword {keyword: $className, type: 'HAS_CLASS', org_id: $orgId})
      -[:APPEARS_IN_FILE]->(fv:FileVersion)
WHERE fv.relative_path CONTAINS 'test'
RETURN fv.relative_path, fv.purpose, fv.summary
ORDER BY fv.committed_at DESC
```

## Basic Reference Queries

```cypher
-- Find all files that import a specific external package
MATCH (k:OrgKeyword {keyword: "express", type: "HAS_IMPORT_EXTERNAL", org_id: $orgId})
      -[:APPEARS_IN_FILE]->(fv:FileVersion)
RETURN fv.relative_path, fv.commit_hash ORDER BY fv.committed_at DESC

-- Find files that provide a specific API contract
MATCH (k:OrgKeyword {keyword: "POST /api/users", type: "PROVIDES_CONTRACT", org_id: $orgId})
      -[:APPEARS_IN_FILE]->(fv:FileVersion)
RETURN fv.relative_path, fv.purpose

-- Count functions per file (top 10 most complex)
MATCH (k:OrgKeyword {type: "HAS_FUNCTION", org_id: $orgId})-[:APPEARS_IN_FILE]->(fv:FileVersion)
RETURN fv.relative_path, count(k) AS fn_count ORDER BY fn_count DESC LIMIT 10

-- Get section_map for multiple files at once (avoids N metadata calls)
MATCH (fn:FileNode {org_id: $orgId})-[:HAS_VERSION]->(fv:FileVersion)
WHERE fn.relative_path IN [$path1, $path2, $path3]
RETURN fn.relative_path, fv.section_map
ORDER BY fv.committed_at DESC
```

## Rules

- Always include `org_id: $orgId` — it's auto-injected, always available
- Use `ORDER BY fv.committed_at DESC LIMIT 1` to get the latest version of a file
- Use `count(DISTINCT k)` for frequency ranking — more shared keywords = more relevant
- Use `collect(DISTINCT k.keyword)[0..5]` to preview matched keywords without overwhelming output
- Regex is supported: `k.keyword =~ '(?i).*pattern.*'` for case-insensitive fuzzy matching
- Keep queries focused — prefer LIMIT 10-15 over unbounded results
- `APPEARS_IN_FILE` targets both `FileVersion` (historical snapshots, 15.8M edges) and `FileNode` (current-state, 4.3M edges) — use `FileVersion` for commit-scoped queries, `FileNode` for latest-state queries
