# lnr

`lnr`'s domain is Linear's issue-tracking model, as exposed through a CLI whose own command
grammar is part of that domain — the vocabulary below covers both.

## Language

**Noun**:
A resource `lnr`'s command grammar operates on (`issue`, `project`, `milestone`, `comment`,
`label`, `team`, `status`), spelled as the first word of a command (`lnr issue create`). A noun
exists in `lnr`'s grammar because it earns its own address space and verb set, not because Linear's
schema happens to expose a type for it.

**Verb**:
The second word of a command (`create`, `update`, `get`, `list`), naming the operation on a noun.
`lnr` gives every writable noun the same verb set rather than letting each one's Linear mutation
shape dictate its own — see [ADR-0001](docs/adr/0001-uniform-command-grammar-over-schema-shape.md).

**Tier**:
One of the three shapes a **payload** can take: lean (default), `--verbose`, `--json`. Chosen by
flag only, never inferred from whether stdout is a TTY. Tiers govern *success* output only — a
failure is rendered in full at every tier — see
[ADR-0003](docs/adr/0003-output-tiers-are-flag-only-and-json-is-not-raw.md).

**Lean**:
The default tier: only the fields a caller couldn't already know, plus enough of a record's body
to be useful without its full text. Never echoes back what was just written.
_Avoid_: Terse, minimal, quiet.

**Verbose**:
The `--verbose` tier: expands lean with the fields it drops (full body, dates, relations), as
human/agent-readable text. Not a parsing contract — nothing should script against its exact shape.

**`--json`**:
The one tier meant to be parsed. Renders lean's fields, structured — never a raw passthrough of
Linear's own API response. See [ADR-0003](docs/adr/0003-output-tiers-are-flag-only-and-json-is-not-raw.md).
_Avoid_: Describing `--json` as "full" or "raw" output — pair with `--verbose` for that instead.

**Caller**:
Whoever invoked `lnr` — in practice a coding agent shelling out with no TTY attached, and only
secondarily a human at a terminal. Design questions are settled by what the caller can *do* with an
answer, not by what would read nicely.
_Avoid_: User (ambiguous between the caller and the Linear user whose account the token belongs to).

**Payload**:
A command's result, and the only thing that ever reaches stdout. Everything else — errors,
warnings, the OAuth URL — goes to stderr, so a failed command leaves stdout **empty** rather than
partially written.

**Exit code**:
The contract's machine-readable channel, so a caller never has to parse prose to decide what to do
next. The seven codes enumerate distinct *caller behaviours*, not distinct failure kinds — which is
why a malformed flag and a Linear-side validation rejection share one code (both mean "fix the
input"), while a rate limit and a timeout do not. Derived from `extensions.code`, never from the
HTTP status: Linear returns rate-limit errors as HTTP 400, not 429.

**Credential**:
The Linear token `lnr` authenticates with — an OAuth access token, or a `LINEAR_API_KEY` on a
headless host. Stored in the macOS Keychain, keyed by workspace ID, reached via `/usr/bin/security`
rather than the native API — see
[ADR-0004](docs/adr/0004-keychain-access-via-the-security-cli.md). `lnr` owns its whole lifecycle
including refresh; a caller only learns a credential exists when there isn't a usable one.

**Issue**:
Linear's core work item. The only noun addressable by a human-readable identifier as well as a
UUID.
_Avoid_: Ticket, ask (these mean the GitHub issues that track `lnr`'s own spec work, a different
thing).

**Identifier**:
An issue's human-readable reference (`ENG-42`): a team's **key** plus a sequence number. Distinct
from a UUID, and distinct from a **reference** below.
_Avoid_: Slug, ID (too easily confused with UUID).

**Reference**:
Whatever a caller hands `lnr` to point at a specific record — identifier, UUID, name, or a pasted
Linear URL, depending on what the noun in question supports. Not every noun accepts every form:
see [ADR-0002](docs/adr/0002-addressing-scheme-follows-api-capability-not-grammar.md). A reference
that matches nothing and one that matches several records are different failures, and `lnr` reports
them as such — the second is a caller-fixable ambiguity, not an absence.

**Team**:
The organizational unit that owns issues and workflow states, and that projects can span. Referred
to in `lnr` by its short **key** (e.g. `ENG`), never its UUID, in any user-facing input.

**Project**:
A grouping of issues that can span multiple teams. Addressed by name, not identifier — projects
have no Linear-assigned short code the way issues do.

**Milestone**:
A checkpoint within a single project. Despite belonging to exactly one project in Linear's own
schema, `lnr` treats it as a top-level noun rather than a nested one — see
[ADR-0001](docs/adr/0001-uniform-command-grammar-over-schema-shape.md).
_Avoid_: Using "milestone" for a release phase of the `lnr` project itself — that's **v1**/**v2+**
(see the map, [#1](https://github.com/taciogt/lnr/issues/1)). The two used to collide in this
repo's own prose; the phase sense was renamed away once the entity became a real noun.

**Status**:
An issue's stage in its team's workflow (e.g. Backlog, In Progress, Done). Scoped per-team: the
same status name can mean a different stage on a different team.
_Avoid_: Workflow state, State (Linear's own API name for this — kept out of user-facing language
in favor of the shorter term).

**Label**:
A tag applied to an issue for categorization, independent of workflow status.

**Comment**:
A threaded note attached to an issue. Linear's API lets a comment attach to other kinds of records
too (a project, an initiative, a document) — `lnr`'s comment noun does not yet cover those; each is
tracked as its own open question rather than assumed.

**Companion skill**:
The Claude Code skill shipped as a plugin from this repo's own marketplace. It is an *accelerator*,
never a dependency: `lnr` stays fully navigable through its `--help` tree with no skill installed.
It therefore carries only what `--help` structurally cannot — which commands exist at all,
cross-command house rules, and **recipes** — and never duplicates a command's flags. See
[ADR-0005](docs/adr/0005-the-companion-skill-never-duplicates-help.md).
_Avoid_: Docs, reference files (v1 ships none — the `--help` tree is the reference).

**Setup skill**:
The plugin's second skill, marked `disable-model-invocation: true` so only a human can invoke it.
Its one job is writing the recommended `lnr` permissions into the user's own settings — the
durable grant a **companion skill**'s turn-scoped `allowed-tools` cannot give. See
[ADR-0007](docs/adr/0007-the-pre-approval-allow-list-omits-lnr-api.md).

**Recipe**:
A short command sequence in the **companion skill**'s body. A recipe earns its place *only* if the
task requires a call the **caller** would not know to make — a flag whose value must be resolved
first, like a **status** name that is per-**team**. Anything an agent could reach by reading
`--help` is not a recipe, and worked examples generally are not either.
_Avoid_: Example, cookbook (both invite the unbounded set this rule exists to exclude).

**Escape hatch**:
`lnr api '<graphql>'`, the raw passthrough that lets the 22-command surface stay at 22 by absorbing
every deferred operation (delete on any noun, cycles, documents, comment resolve/unresolve). The
one command where the **tier** rules do not apply: it returns Linear's response as-is, because the
caller explicitly asked for raw.
