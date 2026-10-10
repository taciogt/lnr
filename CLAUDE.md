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

GitHub Issues is the tracker of record for the **spec** (recorded in `docs/agents/issue-tracker.md`,
which carries the full `gh` command vocabulary these skills expect). **Implementation** is tracked
in the Linear [`lnr` project](https://linear.app/taciogt/project/lnr-6b25dcc7130a) via the Linear MCP, and via `lnr` itself once v0 is usable. The handoff lives in
[Assemble the locked v1 spec document](https://github.com/taciogt/lnr/issues/19).

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
| Output | Lean by default, `--verbose`, `--json`. **Never echo input back on writes.** Tiers govern success only — errors are always full |
| API client | Hand-written, *not* generated ([#3](https://github.com/taciogt/lnr/issues/3)) |
| Auth | OAuth authorization-code + PKCE (S256), loopback redirect, shipped non-secret `client_id`; `LINEAR_API_KEY` fallback ([#2](https://github.com/taciogt/lnr/issues/2)). Cross-workspace `client_id` unverified — deferred until v0 is usable ([HF-110](https://linear.app/taciogt/issue/HF-110), moved from [#12](https://github.com/taciogt/lnr/issues/12)) |
| Distribution | Personal Homebrew tap, GoReleaser `homebrew_casks` ([#6](https://github.com/taciogt/lnr/issues/6)) |
| Credentials | macOS Keychain, keyed by **workspace ID** even though only one is supported; reached via `/usr/bin/security`, *not* the native `SecItem*` API ([#7](https://github.com/taciogt/lnr/issues/7)) |
| Companion skill | Ships as a plugin from **this repo's own marketplace**; carries **no** command reference — `lnr --help` is rich by design and is the reference. Kept true by a script-only CI check, not codegen ([#8](https://github.com/taciogt/lnr/issues/8)) |
| Agent contract | 7 behaviour-keyed exit codes; stdout is payload-only and empty on failure; never prompts; no internal retry ([#7](https://github.com/taciogt/lnr/issues/7)) |
| Testing | Fake GraphQL server at merge, live smoke suite before release, fixtures recorded only from a throwaway workspace. Lean output asserted by golden files, a character budget and a no-echo property; contract proven black-box against the built binary. Merges on Ubuntu; Keychain tests on macOS at release. No coverage threshold ([#16](https://github.com/taciogt/lnr/issues/16)) |
| Settings | No config file in v1. Flags/env only: `--timeout`/`LNR_TIMEOUT`; test seams `LNR_API_URL` (https, or http on loopback) and `LNR_KEYCHAIN`, env-only and absent from `--help`; output tier is flag-only. At most one Keychain item (account = workspace ID); `LINEAR_API_KEY` beats it with no fallthrough, and exit `3` names the rejected source ([#17](https://github.com/taciogt/lnr/issues/17), [ADR-0009](docs/adr/0009-one-credential-item-and-the-environment-credential-wins.md)) |
| Versioning / release | Semver covers command paths, flags, exit-code meanings and `--json` fields (an eighth exit code is major); lean/verbose text, error wording and `LNR_*` seams are not covered. Phase v0 = `0.x`, `1.0.0` = full v1 surface. `lnr --version` prints `lnr v1.0.3`; no update notice, no self-update. Binary and plugin share one version, pinned in `plugin.json`. A pushed `vX.Y.Z` tag is the only release trigger, pushed for now by a project-scoped `/release` skill ([#18](https://github.com/taciogt/lnr/issues/18), [ADR-0010](docs/adr/0010-one-version-and-a-tag-is-the-only-release-trigger.md)) |
| Out of scope | Multi-workspace/profiles (one account per machine); reimplementing the 53 unused MCP tools; non-macOS distribution; initiatives, documents and cycles as nouns, and comments on them — all reachable only via `lnr api` ([#14](https://github.com/taciogt/lnr/issues/14), [#15](https://github.com/taciogt/lnr/issues/15)) |

## Established facts — read before re-deriving

`.scratch/lnr-v1-spec/research/` holds four researched documents with primary sources, live
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
- `07-keychain-non-interactive.md` — an unsigned Go binary's ad-hoc `cdhash` rotates on every
  rebuild, and a Keychain item's `partition_id` ACL is keyed to it, so the **native `SecItem*` API
  blocks a TTY-less process on a GUI dialog after every `brew upgrade`** (observed 7 times). Worse,
  the post-upgrade write *succeeds silently* while the read stays broken. Shelling out to
  `/usr/bin/security` sidesteps it — that path's items are partitioned `apple-tool:`, which is how
  `gh` survives its own cdhash rotation.
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
