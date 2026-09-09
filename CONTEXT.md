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
Whatever a user hands `lnr` to point at a specific record — identifier, UUID, name, or a pasted
Linear URL, depending on what the noun in question supports. Not every noun accepts every form:
see [ADR-0002](docs/adr/0002-addressing-scheme-follows-api-capability-not-grammar.md).

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
