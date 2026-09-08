# What exactly does "lean" contain?

Type: prototype
Status: open

## Question

Lean output is the project's primary lever — measured at roughly **25:1** against schema savings
(~137,000 tokens of response payload across 191 calls, vs ~4,100 tokens/session of schema). But
nobody has specified the lean shape for a single command.

Make it concrete. Real payloads already exist in the user's transcripts at
`~/.claude/projects/**/*.jsonl` (tool results for `mcp__linear-server__*`); a representative
`save_issue` response is 2,981 chars / 29 fields / 10 null or empty, and echoes the entire
description back.

Produce a side-by-side, with token counts, for at least:

- `save_issue` (create and update) — target **under ~50 tokens**: identifier + URL, nothing echoed
- `get_issue --lean` — target **under ~250 tokens**; decide whether the description body is in or
  out by default
- `list_issues` — decide the per-row shape and whether it is tabular or one-line-per-issue

Then answer the design questions the comparison surfaces:

1. What are the three tiers? Proposal from charting: lean default, `--verbose`, `--json` for full
   fidelity. Does `--json` mean *raw API JSON* or *lean fields as JSON*? These are different
   promises.
2. Are nulls omitted entirely, or rendered? Omitting is cheaper but makes absence ambiguous.
3. Does lean output stay stable enough to parse, or is it explicitly human/agent-readable prose
   that agents should not script against?
4. Should output shape depend on whether stdout is a TTY, or only on flags? Charting leaned
   toward flags-only for predictability — confirm or overturn.

Link the prototype from this ticket rather than pasting it.
