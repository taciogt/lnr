# The pre-approval allow-list deliberately omits `lnr api`

Decided in [#8](https://github.com/taciogt/lnr/issues/8). Everywhere `lnr` is pre-approved for a
calling agent — the companion skill's `allowed-tools`, and the settings block the setup skill
writes — the seven nouns are enumerated (`Bash(lnr issue *)`, `Bash(lnr project *)`, …) and the
escape hatch is left off. This looks like an oversight. It is not, and it should not be "fixed" by
collapsing the list to `Bash(lnr *)`.

The boundary is one [#5](https://github.com/taciogt/lnr/issues/5) and
[#7](https://github.com/taciogt/lnr/issues/7) already drew. #5 deferred **every destructive
operation** — delete on all five nouns — to `lnr api`. #7 then concluded no `--yes` flag was needed
*because v1 ships no destructive command*. Both hold only because the irreversible operations sit
behind the passthrough. Blanket pre-approval of `lnr api` silently retires the safety property the
other two were relying on, without touching either decision's text.

Read-only pre-approval was the other candidate and was rejected for prompting on `issue create` —
the highest-traffic operation in the project's baseline — taxing the hot path to guard something
that is not dangerous.

## Consequences

**`allowed-tools` is turn-scoped and is not the durable mechanism.** Permissions granted through
skill frontmatter cover the turn the skill is invoked and clear when the next message is sent. It
is still worth declaring — free, and the correct boundary — but a day's Linear work stops prompting
only through the user's own `settings.json`, which no skill can set on its own.

**That setup is a user-invoked skill, not a `lnr` command.** `lnr` writing to `~/.claude/settings.json`
was rejected: #7 forbids `lnr` from prompting, so it has no way to ask first, and a CLI silently
reaching into a user's Claude configuration is the kind of surprise that costs trust. A second
skill in the plugin, marked `disable-model-invocation: true`, inverts every part of that — the user
invoked it, Claude does the write through its own edit tooling, and the user sees and approves the
diff. It can also read existing settings and merge rather than clobber, and explain why the escape
hatch is excluded, neither of which a README snippet can do.
