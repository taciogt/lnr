# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo currently is

**A spec in progress, not a codebase.** There is no Go source, no `go.mod`, no build, no tests.
Do not invent build/lint/test commands or scaffold a project structure unless asked — the whole
point of the current phase is to finish deciding before anyone writes code.

`lnr` will be a Go CLI replacing the Linear MCP for daily agent-driven Linear work. The tracked
content is the reasoning that will produce it: a wayfinder map on GitHub Issues, plus research
findings under `.scratch/lnr-v1-spec/research/`.

## Where the work is tracked

GitHub Issues is the tracker of record (recorded in `docs/agents/issue-tracker.md`, which carries
the full `gh` command vocabulary these skills expect).

- **Map**: [#1](https://github.com/taciogt/lnr/issues/1), labelled `wayfinder:map`. Holds the
  destination, standing constraints, baseline measurements, Decisions-so-far, fog, and out-of-scope.
- **Tickets**: sub-issues of #1, labelled `wayfinder:research` / `prototype` / `grilling` / `task`.
- **Blocking**: GitHub's *native* issue dependencies, not a body convention.
- **Board**: <https://github.com/users/taciogt/projects/7>. Columns derive from issue state
  (closed → Done, `blocked_by > 0` → Blocked, assigned → In progress), so fix the issue, not the card.

Find the frontier — open, unblocked, unclaimed:

```sh
gh api "repos/taciogt/lnr/issues?state=all&per_page=100" \
  --jq '.[] | select(.number>1) | select(.state=="open")
        | select((.issue_dependencies_summary.blocked_by // 0)==0)
        | select(.assignee==null) | "#\(.number)\t\(.title)"'
```

⚠️ `gh api repos/OWNER/REPO/issues` returns **open issues only** by default. Omitting
`state=all` silently misreports closed tickets — this has already caused one wrong result here.

## The one constraint that drives every design decision

Lean output. It is not a feature, it is the justification for the project, and it was measured
rather than assumed (full numbers in `README.md`):

- The Linear MCP exposes **69 tools**; only **16** were ever called, and 5 carry **83%** of traffic.
- Schema weight: ~45,500 tokens eager, **~4,100 per session** when deferred.
- Response payloads: **~137,000 tokens across 191 calls** — `save_issue` alone averages ~806 tokens
  to confirm a write, echoing back the description it was just sent.

**Response savings beat schema savings roughly 25:1.** Judge every output decision in tokens
against the per-call targets on the map: `save_issue` under ~50 tokens, `get_issue --lean` under
~250, companion skill under ~150 resident.

## Decisions already locked

Do not relitigate these without a reason; each links to the ticket holding its reasoning.

| | |
|---|---|
| Language / name | Go; binary `lnr`; noun-verb grammar (`lnr issue create`) |
| Surface | 22 noun-verb commands (full `create`/`update`/`get`/`list` CRUD on `issue`/`project`/`milestone`/`comment`/`label`, read-only `team list`/`status list`) + a raw `lnr api '<graphql>'` passthrough ([#5](https://github.com/taciogt/lnr/issues/5)) |
| Output | Lean by default, `--verbose`, `--json`. **Never echo input back on writes.** |
| API client | Hand-written, *not* generated ([#3](https://github.com/taciogt/lnr/issues/3)) |
| Auth | OAuth authorization-code + PKCE (S256), loopback redirect, shipped non-secret `client_id`; `LINEAR_API_KEY` fallback ([#2](https://github.com/taciogt/lnr/issues/2)) |
| Distribution | Personal Homebrew tap, GoReleaser `homebrew_casks` ([#6](https://github.com/taciogt/lnr/issues/6)) |
| Credentials | macOS Keychain, keyed by **workspace ID** even though only one is supported |
| Out of scope | Multi-workspace/profiles (one account per machine); reimplementing the 53 unused MCP tools; non-macOS distribution |

## Established facts — read before re-deriving

`.scratch/lnr-v1-spec/research/` holds three researched documents with primary sources, live
probe transcripts, and explicit "what I could not establish" sections. **Read the relevant one
before investigating Linear's API, OAuth, or Homebrew** — these questions are answered, including
the traps:

- `01-oauth.md` — `mcp.linear.app` and `api.linear.app` keep **separate client registries**
  (proven). No device-code grant exists, so headless/SSH has no browser path. Tokens live 24h.
  Both OAuth *and* API keys are admin-disableable.
- `02-graphql-client.md` — introspection is public (no token needed); 94% of schema fields are
  documented; pagination has a `nodes` shortcut past `edges`/`cursor`; errors carry real HTTP
  statuses plus `extensions.code` and `userPresentableMessage`. An invalid token and *no* token
  return an identical 401.
- `05-homebrew.md` — GoReleaser's `brews:` is fully deprecated; use `homebrew_casks`. **Cask
  installs are quarantined and an unsigned binary is SIGKILLed on first run (exit 137)** without
  the `xattr -dr com.apple.quarantine` post-install hook. Do not add `url.verified` — deprecated,
  and it fails `goreleaser check` despite appearing in GoReleaser's own migration example.

Research output is a linked asset of its ticket and **is tracked in git**, despite living under
`.scratch/`.

## Working conventions

- **Plan, don't build.** The map's destination is a locked spec. Produce decisions, not
  deliverables; the pull to just write the code is the signal that the map is finished.
- **Resolve at most one ticket per session** (research tickets excepted). Claim it by assigning
  yourself *before* any work; resolve by commenting the answer, closing, then appending a one-line
  gist to the map's Decisions-so-far.
- **Refer to tickets by name, never a bare number**, in anything a human reads.
- Use `ctx7` for any library/API documentation question rather than training data — this is a
  standing user rule, and every research ticket here followed it.
- Verify claims by running the thing (`goreleaser check`, a real probe) rather than by reading
  docs alone. Both corrections that reshaped this design came from doing that.
