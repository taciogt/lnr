# lnr

A lean Linear CLI for agents and humans.

> **Status: speccing, not built.** Nothing is implemented yet. This repo currently holds a
> [wayfinder map](https://github.com/taciogt/lnr/issues/1) — the open decisions, the research
> behind them, and the measurements the design is accountable to. The numbers below are real,
> taken from a year of actual Linear MCP usage; the tool they justify is not written yet.

## Why this exists

The [Linear MCP server](https://linear.app/docs/mcp) works. It is also expensive in the one
resource an agent cannot buy more of: **context**.

This project started by measuring rather than assuming. Every figure below comes from one
developer's real Claude Code transcripts — 191 Linear tool calls across many sessions — plus a
`/context` breakdown of live tool schemas.

### Finding 1 — 77% of the surface is never touched

| | |
|---|---|
| Tools the Linear MCP exposes | **69** |
| Tools ever called | **16** |
| Share of all calls from the top 5 tools | **83%** (159 / 191) |

Five tools — `save_issue`, `get_issue`, `list_issues`, `save_comment`, `list_issue_statuses` —
carry five-sixths of the traffic. The unused 53 include entire subsystems: diff review, release
pipelines, agent skills, attachment upload.

### Finding 2 — schemas are a real but bounded cost

Loaded eagerly, Linear's 69 tool schemas cost roughly **45,500 tokens**, in every request, all
session. Modern harnesses defer them and load on demand, which helps a lot:

| | cost |
|---|---|
| 69 tool names in the deferred listing | ~800 tokens, every request, every session |
| schemas for the ~5 tools actually loaded | ~3,300 tokens, resident once loaded |
| **realistic per-session total** | **~4,100 tokens** |

*(Calibrated against a real `/context` reading of 109 deferred MCP tools totalling 109,326
tokens — a 660-token mean once three outliers are excluded.)*

### Finding 3 — the responses are the real cost, by 25 to 1

Schema weight is paid **once per session**. Response payloads are paid **on every call**:

| tool | calls | avg response | total |
|---|---|---|---|
| `save_issue` | 84 | 2,981 chars | **~67,700 tokens** |
| `get_issue` | 47 | 3,491 chars | **~44,300 tokens** |
| `list_issues` | 12 | 2,465 chars | ~8,000 tokens |
| *(13 others)* | 48 | — | ~17,000 tokens |
| **total** | **191** | **719 tokens/call** | **≈137,000 tokens** |

A representative `save_issue` response carries **29 fields, 10 of them null or empty**
(`archivedAt`, `completedAt`, `canceledAt`, `dueDate`, `slaStartedAt`, `slaMediumRiskAt`,
`slaHighRiskAt`, `slaBreachesAt`, `attachments`, `documents`) — and **echoes back the entire
3,873-character description that was just sent to it.**

That is ~806 tokens to be told a write succeeded. The useful content was an identifier and a URL:
about 20 tokens.

**Schema savings are worth ~4K per session. Response savings are worth ~100K of that 137K.**
Lean output is not a nice-to-have feature of this project; it is the entire point.

## What `lnr` does differently

**Lean by default.** `--verbose` to expand, `--json` for full fidelity. The agent is the primary
consumer and should not have to remember a flag to avoid a 3,000-character reply. Writes never
echo their input back.

Per-call targets the design is held to:

| operation | target |
|---|---|
| `save_issue` response | under ~50 tokens (identifier + URL) |
| `get_issue --lean` | under ~250 tokens |
| companion skill, always-resident cost | under ~150 tokens |

**A small committed surface, plus an escape hatch.** Roughly 8 commands covering what actually
gets used, and a raw `lnr api '<graphql>'` passthrough in the shape of `gh api`. That covers the
other 53 tools without designing 53 commands — and, since the raw Linear GraphQL API is richer
than the MCP's projection of it, potentially more than the MCP can reach.

**Self-documenting, progressively.** A companion agent skill stays under ~150 tokens resident and
discloses detail progressively, sliced by task rather than by command. The CLI is the source of
truth for its own documentation, so the two cannot drift.

**Composable.** Output goes to stdout, diagnostics to stderr, distinguished exit codes for
not-found / auth-required / rate-limited. Pipe it, script it, branch on it — none of which an MCP
tool call permits.

**One static binary.** Go, distributed via Homebrew. No Node startup penalty on an invocation an
agent may make dozens of times per session.

## Design decisions settled so far

Each links to the ticket holding the full reasoning.

| decision | outcome |
|---|---|
| [Auth](https://github.com/taciogt/lnr/issues/2) | OAuth authorization-code + PKCE (S256) with a loopback redirect, shipping a hard-coded non-secret `client_id` — a `gh auth login`-style browser flow, no copy-paste of API keys. `LINEAR_API_KEY` as the documented fallback. |
| [API client](https://github.com/taciogt/lnr/issues/3) | Hand-written, not generated. Drift guarded by a 28-line CI check validating operation strings against a vendored schema. |
| Language | Go. Single static binary, negligible startup. |
| Output | Lean by default; `--verbose`; `--json`. Never echo input on writes. |
| Multi-workspace | Out of scope — one account per machine. Storage is keyed by workspace ID for forward compatibility. |
| Credential storage | macOS Keychain, with a documented file fallback for CI and headless use. |

### Two findings that overturned an assumption

Research is only useful when it is allowed to contradict the plan. Twice it did:

- **Dynamic Client Registration looked like a way to skip registering an OAuth app entirely.**
  It is not. `mcp.linear.app` and `api.linear.app` keep *separate client registries* — a client
  registered with the former is rejected outright by the latter. Proven with paired probes, not
  inferred.
- **"Generated clients are too big" was the obvious argument for hand-writing. It is false.**
  Running `genqlient` against the real milestone-1 operations produced **1,786 lines**, not the
  feared 50,000 — codegen emits only what the operations reach. Size does not discriminate here.
  The decision rests instead on the raw passthrough already forcing an untyped request path to
  exist, which codegen would duplicate rather than replace.

## Repository layout

```
.scratch/lnr-v1-spec/
  README.md            pointer to the map; frontier query
  research/            research findings, linked from their ticket
docs/agents/
  issue-tracker.md     records GitHub Issues as this repo's tracker
```

The map and its tickets live in [GitHub Issues](https://github.com/taciogt/lnr/issues), not in
markdown, so blocking relationships render natively and the frontier is visible in the UI.

- **Map**: [#1](https://github.com/taciogt/lnr/issues/1) (`wayfinder:map`)
- **Tickets**: sub-issues of the map, labelled `wayfinder:research` / `prototype` / `grilling` / `task`

Frontier — open, unblocked, unclaimed:

```sh
gh api repos/taciogt/lnr/issues \
  --jq '.[] | select(.number>1) | select(.state=="open")
        | select((.issue_dependencies_summary.blocked_by // 0)==0)
        | select(.assignee==null) | "#\(.number)\t\(.title)"'
```

## Honest limitations

- **`lnr` does not exist yet.** This is a spec in progress.
- The measurements come from **one developer's usage**. The shape of the finding — a handful of
  tools dominating, responses dwarfing schemas — is likely general; the exact ratios are not.
- That usage was **shaped by the MCP itself**. Tools may go unused because they were awkward to
  reach, not because they are unwanted. The command surface ticket treats this as an open
  question rather than reading the data naively.
- The secretless OAuth exchange is **documented but not yet observed**. Confirming it needs a real
  OAuth app: [#10](https://github.com/taciogt/lnr/issues/10).
- An MCP server and a CLI are not strictly comparable. An MCP server is discoverable by any
  client with zero setup; a CLI must be installed and taught. This project bets that for a tool
  used every day, that trade is worth making.

## Prior art

`gh` for the auth flow and the `gh api` escape hatch, `kubectl` and `docker` for noun-verb
grammar, and Linear's own [GraphQL API](https://linear.app/developers/graphql) — which is public,
introspectable without a token, and 94% documented at the schema level.

## License

MIT (intended; not yet added).
