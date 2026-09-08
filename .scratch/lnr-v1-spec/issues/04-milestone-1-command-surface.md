# Which commands ship in milestone 1?

Type: grilling
Status: open

## Question

Settled while charting: roughly 8 commands plus a raw `lnr api` GraphQL passthrough, binary named
`lnr`, noun-verb grammar (`lnr issue create`).

Usage data from the user's transcripts (191 calls, 16 distinct tools ever used):

| calls | tool |
|---|---|
| 84 | `save_issue` |
| 47 | `get_issue` |
| 12 | `list_issues` |
| 9  | `save_comment` |
| 7  | `list_issue_statuses` |
| 5  | `list_teams`, `list_projects`, `list_comments`, `create_issue_label` |
| 1–2 | `list_issue_labels`, `save_project`, `get_project`, `save_milestone`, `get_document`, `list_milestones` |

To decide:

1. The exact command list and its noun-verb spelling. Note `save_issue` is one MCP tool doing
   both create and update — does `lnr` split it into `issue create` / `issue update`, or keep a
   single upsert?
2. Required vs. optional flags per command, and which have sensible defaults (team, status).
3. How issues are addressed: identifier (`HF-84`), UUID, or URL — and whether all three are
   accepted everywhere.
4. Whether reads that the MCP splits (`get_issue` + `list_comments`) collapse into one command
   with a flag (`lnr issue get HF-84 --with-comments`), saving a round-trip.
5. Where the milestone-1 line falls: what is deliberately *excluded* and left to `lnr api`.

Consider that usage was **shaped by the MCP** — the user may not have used cycles or documents
because they were awkward, not because they are unwanted. Probe that.
