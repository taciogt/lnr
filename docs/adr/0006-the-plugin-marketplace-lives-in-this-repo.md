# `lnr`'s plugin marketplace lives in this repo, not `taciogt/ai-plugins`

Decided in [#8](https://github.com/taciogt/lnr/issues/8). `taciogt/lnr` hosts its own
`.claude-plugin/marketplace.json`, publishing the companion skill as a plugin from the same repo as
the Go source. The obvious home was the existing personal marketplace at `taciogt/ai-plugins`,
alongside the other plugins; this records why it is not there.

**The CI check decides it.** [ADR-0005](0005-the-companion-skill-never-duplicates-help.md) puts a
validation step between `SKILL.md` and the binary's command tree. That check needs both artifacts
**in one repo at one commit**. Publishing from `ai-plugins` would split the two things the check
exists to hold together, across a repo boundary, and let them version independently — which is the
drift the check was written to prevent, reintroduced at a level the check cannot see.

Hosting a marketplace inside a repo that is mostly source code is a well-trodden shape, not an
abuse of one: the documented example of adding a marketplace is `/plugin marketplace add
anthropics/claude-code`, itself overwhelmingly a source repo.

## Consequences

**Updates are pulled, not pushed.** They arrive via `claude plugin marketplace update`. Better than
"copy this file into place," but the spec should not promise silent auto-update.

**Skew between skill and binary is possible again, and is answered in the error text.** The plugin
updates through Claude's mechanism while the binary updates through `brew upgrade` — two
independent paths against one repo, so a user can hold a newer skill than binary. The harmful
direction is skill-ahead: the agent tries a command the installed binary lacks. `lnr`'s
unknown-command error therefore names the installed version and points at `--help`:

```
unknown command 'cycle' (lnr v1.0.3) — run `lnr --help` for this binary's surface;
your companion skill may be newer than this binary.
```

Loud, self-diagnosing, zero resident cost, and consistent with
[#7](https://github.com/taciogt/lnr/issues/7), which already makes errors full at every tier. A
`lnr skill check` command was the alternative and fails on the obvious ground that nobody would run
it.

**`go:embed` was considered and rejected.** Compiling `SKILL.md` into the binary and installing it
from there makes skew structurally impossible, since the installed skill is byte-identical to the
one CI validated against that exact binary. Real distribution beat structural lockstep: the skew
failure is cheap to make loud, and an install path nobody can discover is not an install path.
