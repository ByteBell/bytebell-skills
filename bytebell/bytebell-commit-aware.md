# ByteBell Commit-Aware Analysis

## When This Skill Applies

Any query that references a specific commit, version, PR, or point in time.
"At commit abc123...", "in the version where bug X was introduced", "before the refactor..."

## Rule: Carry commitHash Through EVERY Call

1. Extract commitHash from the question (40-char hex string)
2. Pass commitHash to ALL subsequent calls: smart_search, graph_search, graph_traverse, keyword_lookup, retrieve_file
3. If you omit it, you get the latest indexed version — which may be FIXED code, not the buggy version

## How commitHash Works in Tools

| Tool           | What commitHash does                                             |
| -------------- | ---------------------------------------------------------------- |
| smart_search   | Filters OrgKeyword → FileVersion links to that commit's versions |
| graph_search   | Same — channel results are commit-scoped                         |
| graph_traverse | Returns FolderVersion/FileVersion at that commit                 |
| retrieve_file  | Returns FileVersion content at that commit, not current          |
| keyword_lookup | Matches OrgKeywords linked to FileVersions at that commit        |

## Version Node Behavior

- Only files CHANGED in a commit get a new FileVersion node
- Unchanged files retain their previous FileVersion
- FileNode always has CURRENT commit properties — for historical: use FileVersion
- Order by committed_at DESC to find the latest version at or before target commit

## If Results Look Wrong

Re-check: did you include commitHash? If not, re-fetch with it before concluding.
Code at commit X may differ significantly from current code.
