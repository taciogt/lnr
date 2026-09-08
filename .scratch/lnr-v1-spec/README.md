# lnr v1 spec — moved to GitHub Issues

This effort's wayfinder map and tickets were charted here as markdown, then migrated to GitHub
Issues, which is now the **single source of truth**. The markdown copies were removed rather than
kept in parallel, so the two cannot drift.

- **Map**: https://github.com/taciogt/lnr/issues/1 (label `wayfinder:map`)
- **Tickets**: the map's sub-issues, labelled `wayfinder:<type>`
- **Blocking**: GitHub's native issue dependencies, visible in the issue UI

Frontier query (open, unblocked, unclaimed):

```sh
gh api repos/taciogt/lnr/issues \
  --jq '.[] | select(.number>1) | select(.state=="open")
        | select((.issue_dependencies_summary.blocked_by // 0)==0)
        | select(.assignee==null) | "\(.number)\t\(.title)"'
```

Research findings land under `.scratch/lnr-v1-spec/research/`, linked from their ticket.
