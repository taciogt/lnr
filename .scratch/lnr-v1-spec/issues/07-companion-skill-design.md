# How is the companion skill structured, and how does it stay true?

Type: grilling
Status: open
Blocked by: 04

## Question

Settled while charting: a thin skill whose frontmatter `description` stays under ~150 tokens (the
only always-resident cost), using **progressive disclosure** — the skill body loads on trigger,
and detailed reference splits across multiple files loaded only for the task at hand.

The user's reasoning, which stands: a rich `lnr --help` piped into context on every invocation
recreates the MCP's bloat in a new shape. Skill files can be sliced **by task** ("creating
issues", "querying and filtering") rather than by command, which is a better cut than `--help`
gives.

The synthesis to validate: **the CLI is the source of truth**, and emits or validates the skill's
reference files (`lnr docs --emit-skill`), so the docs cannot drift from the binary while still
being task-shaped.

To decide:

1. The file layout. How many reference files, sliced along which task boundaries?
2. What lives in `SKILL.md` itself vs. a reference file? The body should be enough for the common
   case (create an issue, read an issue) without loading anything further.
3. Emit vs. validate. Does the CLI *generate* the markdown, or does CI *check* hand-written
   markdown against the binary's actual surface? Generation resists drift harder; validation
   keeps the prose human.
4. Does `--help` stay terse deliberately, given the skill carries the teaching load?
5. How does the skill tell the agent when *not* to use `lnr`?

Depends on the command surface being settled first.
