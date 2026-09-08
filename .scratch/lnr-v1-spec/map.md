# Map: lnr v1 spec

Label: `wayfinder:map`

## Destination

A **locked spec + ADRs** for `lnr`, a Go CLI that replaces the Linear MCP for daily agent-driven
Linear work — complete enough to hand to an implementation session with nothing left to decide.
Documented in a public GitHub repo on the personal account (`taciogt/lnr`). The map ends at the
spec; it does not ship the binary.

## Notes

**Domain**: Linear GraphQL API client, Go, distributed via Homebrew. Consumed primarily by a
Claude Code agent shelling out, secondarily by a human at a terminal.

**Skills every session should consult**: `mattpocock-skills:grilling` and
`mattpocock-skills:domain-modeling` by default. Use `ctx7` (per the user's global rule) for any
Linear API / goreleaser / Homebrew documentation question rather than training data.

**Standing preferences for this effort**:
- Planning only. Produce decisions, not deliverables. No implementation code beyond throwaway
  prototypes linked from tickets.
- Lean output is the primary design constraint, not a feature. Every output decision is judged in
  tokens.
- The CLI is the source of truth for its own documentation. Skill reference files are emitted or
  validated from the binary, never hand-maintained in parallel.
- Even with one workspace per machine, key config and Keychain entries by **workspace ID** from
  day one. Near-free now, a breaking change later, and the repo is public.

## Baseline measurements

Established while charting, from the user's own transcripts and `/context`. These are the numbers
the destination is accountable to.

| Fact | Value |
|---|---|
| Tools the Linear MCP exposes | 69 |
| Tools the user has ever called | 16 |
| Share of calls from the top 5 tools | 83% (159 / 191) |
| Linear MCP schema weight if eagerly loaded | ~45,500 tokens |
| Realistic per-session cost in deferred mode | ~4,100 tokens (~800 name floor + ~3,300 loaded schemas) |
| **Total response payload across 191 calls** | **~137,000 tokens** |
| Mean response size per call | ~719 tokens |
| `save_issue` mean response | 2,981 chars (~806 tokens), 29 fields, 10 null/empty, echoes the full description back |
| `get_issue` mean response | 3,491 chars (~943 tokens) |

Per-call acceptance targets (falsifiable, unlike the aggregate):
- `save_issue` response under **~50 tokens** (identifier + URL)
- `get_issue --lean` under **~250 tokens**
- companion skill frontmatter under **~150 tokens** resident

## Decisions so far

<!-- one line per closed ticket: gist + link -->

_(none yet — charting session)_

## Not yet specified

- **Milestone 2+ surface.** Which of the remaining used-but-not-core capabilities (cycles,
  documents, project milestones, label management) graduate into first-class commands, in what
  order, and which stay behind the raw passthrough forever.
- **Testing strategy.** Recorded response fixtures vs. a live scratch Linear team vs. a mocked
  GraphQL layer. Hangs on how the client is built (hand-written vs. generated).
- **Where the companion skill ships.** In this repo, or in the existing `taciogt/ai-plugins`
  marketplace as a plugin. Hangs on the skill's file layout.
- **Release and versioning behaviour.** Semver policy, update-available notification, whether the
  binary self-checks for new versions.
- **Config precedence.** Flags vs. environment vs. config file, and the config file's format.
- **Rate limiting and retry.** Linear's API limits are unknown; behaviour under 429 is undecided.

## Out of scope

- **Multi-workspace / profile model.** Each workspace is used on a different machine; the CLI is
  locked to a single account per machine. No `--profile`, no context switching. Forward-compat is
  preserved by keying storage on workspace ID (see Notes).
- **Reimplementing the 53 unused MCP tools as commands.** The raw `lnr api` GraphQL passthrough
  covers them without designing 53 command surfaces.
- **Non-macOS distribution.** Mac-first via Homebrew. Go makes Linux cheap later, but it is not
  part of this effort.
