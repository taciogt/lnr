# The companion skill never duplicates `--help`

Decided in [#8](https://github.com/taciogt/lnr/issues/8). `lnr`'s own `--help` tree is the command
reference. The companion skill ships **no reference files** and no flag documentation, and `--help`
is rich by design rather than deliberately terse.

This inverts the premise #8 was opened with — that a rich `--help` "piped into context on every
invocation recreates the MCP's bloat in a new shape." That reasoning holds only for the *top-level*
`--help`. `lnr issue create --help` is pull-based, scoped to one command, costs zero tokens until
it is run, and **cannot drift, because it is the binary**. It is already the progressive disclosure
the reference files were meant to provide, without a second artifact to keep true.

What decided the split was sorting the agent's failures, not the artifacts:

| Failure | Fixed by |
|---|---|
| "I don't know the flags for the command I chose." | one `--help` round-trip — nothing resident |
| "I don't know `lnr milestone list` exists." | **only** something resident |
| "I don't know the house rules." | resident; `--help` is structurally the wrong place for cross-command rules |

So the skill carries exactly the second and third, and nothing else. Concretely, its body holds:

- **The grammar rule, not the cross-product.** [ADR-0001](0001-uniform-command-grammar-over-schema-shape.md)
  locked a *uniform* verb set, so 22 command lines restate a decision made specifically so it would
  not need restating. The index is the rule plus the noun list — plus a gloss on only the three
  nouns where the grammar misleads: `milestone` is top-level though Linear nests it, `comment`
  attaches to issues only in v1, and `status` is per-team.
- **Only house rules that change behaviour.** A line earns the body if an agent lacking it does the
  *wrong thing*, not if it merely would not know something. This is why the seven-code table stays
  in `--help` while "never retry a rate limit" is in the skill: the agent already sees the exit
  code in its tool result, but its untrained instinct on a rate limit is to sleep and retry.
- **Recipes only where a prior lookup is required.** `--help` can say `--status` takes a string; it
  cannot say you must call `lnr status list --team ENG` first because status names are per-team.
  That dependency is the entire qualifying category.

## Consequences

**The drift question mostly dissolves, and what is left is validated, not generated.** Only the
index is derivable from the binary — roughly the smallest and most stable fraction of the body —
so generating it would mean hand-editing prose around machine-owned blocks for little gain. Instead
CI extracts every command path and `--flag` from `SKILL.md` and asserts each exists in the built
binary's command tree. This is the same move [#3](https://github.com/taciogt/lnr/issues/3) made one
level down, where a 28-line check validates operation strings against a vendored schema; choosing
generation here would mean choosing it in the skill right after rejecting it in the client.

**The check must assert set *equality*, not containment.** Token extraction proves nothing in the
file is wrong; it does not prove nothing is missing. A new noun shipping while the index silently
omits it is failure mode two above — the only failure residency exists to prevent. So the noun list
in `SKILL.md` must equal the binary's exactly.

**Therefore recipes are literal, not abstract.** The only argument for abstract recipes was that
literal ones bake in flag names that could drift. The check removes that risk, so recipes name real
commands and real flags — which is what an agent pattern-matches.

**Discovery does not depend on the skill.** `lnr --help` lists the nouns and states the grammar;
`lnr <noun> --help` the verbs; `lnr <noun> <verb> --help` the flags. A human at a terminal, or an
agent on a machine with no skill installed, navigates the whole CLI unaided. The skill is an
accelerator, never a dependency.
