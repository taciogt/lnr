# What makes the skill trigger at the right moment?

Type: grilling
Status: open
Blocked by: 07

## Question

The skill's frontmatter `description` is the only part resident in every session, and it is the
sole thing deciding whether the skill fires at all. A skill that never triggers is worse than no
skill: the agent falls back to whatever it can guess, or to the MCP the user is trying to retire.

To decide:

1. The `description` text, under ~150 tokens, covering the trigger surface: "linear", "issue",
   "ticket", workspace-specific vocabulary, and the user's own habitual phrasings.
2. How it avoids false positives against the other 48 skills already in the user's listing —
   several of which (`mattpocock-skills:wayfinder`, `code-review`) also talk about issues and
   tickets.
3. Whether it should trigger on Linear *URLs* and identifier patterns (`HF-84`) appearing in the
   conversation, not just on intent words.
4. Whether to validate the description with `skill-creator`'s eval tooling, which the user already
   has installed, rather than guessing at trigger accuracy.

Depends on the skill's structure being settled first.
