# One version for binary and plugin, and a tag is the only release trigger

Decided in [How is `lnr` versioned and released?](https://github.com/taciogt/lnr/issues/18).
`lnr`'s binary and its companion plugin share a single version, a pushed `vX.Y.Z` tag is the only
thing that releases, and for now a project-scoped `/release` skill is what a human uses to push it.

## What semver promises

The caller is an agent, so *breaking* means "a caller that worked yesterday now does the wrong
thing".

- **Covered (breaking = major):** command paths and flag names; the meaning of each of the seven
  exit codes; `--json` field names and types. Adding a field or a command is minor.
- **Not covered:** lean and `--verbose` layout, error and `--help` wording, stderr, and the
  `LNR_*` test seams ([ADR-0009](0009-one-credential-item-and-the-environment-credential-wins.md)).
- **Adding an eighth exit code is major.** Callers branch on "anything else".
- **No Go API is promised.** Everything lives under `internal/`, so a v2 never needs a `/v2` module
  path.

Release phases map to numbers as follows: phase **v0** ships as `0.x.y`, and `1.0.0` is cut only
when the full v1 surface exists. Under `0.x` a minor may break anything; the promises above start at
`1.0.0`. The first tag is `v0.1.0`.

## Version reporting

`lnr --version` prints one line, `lnr v1.0.3`, to stdout with exit `0`. It is a flag, not a command,
so the surface stays at 22, and tiers do not apply. An untagged `go build` reports `lnr vdev`
(`-X main.version`, with a `debug.ReadBuildInfo` fallback for `go install`). The same string is the
one the unknown-command error names ([ADR-0006](0006-the-plugin-marketplace-lives-in-this-repo.md)).

## No update notification, no self-update

`lnr` makes no network call except to the Linear API. A "new version available" line would cost
every agent call latency and stderr tokens, add a second endpoint that can fail, and sit badly with
"never prompts". Upgrading is `brew upgrade lnr`; skew is made loud by the unknown-command error.

## One version, set explicitly

`plugin.json` carries `version`, equal to the binary's tag, written in the release commit. A plugin
with no `version` tracks the marketplace branch's commit SHA, which would deliver skill text for
commands the released binary lacks on every merge — the harmful skill-ahead skew
[ADR-0006](0006-the-plugin-marketplace-lives-in-this-repo.md) names, made routine. Pinning means
plugin users update only when a release is cut. The pipeline fails if tag and `plugin.json` differ.

**Cost accepted:** a fresh plugin install clones `main`, which can be ahead of the latest release.
The skew error covers it; a pinned `source.ref` is the escape hatch if it ever matters.

## The tag is the trigger

- **One workflow, tag-triggered**, in this order, each job needing the previous:
  1. `verify` (Ubuntu): tests, `goreleaser check`, tag equals `plugin.json` version, the
     skill-versus-binary check.
  2. `keychain` (macOS): the round-trip and build-A-writes/build-B-reads tests, and the
     refresh-lock test.
  3. `smoke` (Ubuntu, a `release` GitHub environment holding the throwaway workspace's key): the
     live suite.
  4. `publish`: `goreleaser release` creates the GitHub Release and commits the cask to the tap.
  5. `verify-install` (macOS): `brew install taciogt/tap/lnr`, then `lnr --version` must equal the
     tag. This catches the quarantine SIGKILL (exit 137).
- **Pulling a release** (a failed step 5): delete the GitHub Release and revert the tap commit. The
  tag stays and its number is burned; the fix is the next patch.
- **Tag lifetime.** An unpublished tag may be deleted and re-pushed. A published tag never moves or
  is reused.
- **Prerelease tags** (`v1.0.0-rc.1`) publish a GitHub prerelease only, never the tap
  (`skip_upload: auto`), so the pipeline can be rehearsed without touching users.
- **Release notes** are GoReleaser-generated from commit subjects. There is no hand-kept
  `CHANGELOG.md`. A breaking change is flagged on the first line.

## Who pushes the tag

For now a human does, through a project-scoped `/release` skill at `.claude/skills/release/`.
It is tracked in git, is **not** part of the shipped companion plugin, and is
`disable-model-invocation: true`, so an agent never decides on its own to release. In order, it:

1. runs read-only preflight: clean tree, on `main`, up to date, last CI green;
2. proposes the next version from commits since the last tag using the rules above; you confirm;
3. walks the manual checklist — a local snapshot build and a real `lnr auth login` with browser
   consent — asking you to confirm each item, never faking one;
4. makes the release commit (`plugin.json` version) on `main`;
5. shows the tag and commit, asks, then pushes the tag;
6. watches the workflow and reports the outcome, including a pulled release.

It stops at the first failure and changes nothing before step 4. Its job ends at "tag pushed".

## Automation comes later, and changes only who pushes the tag

The workflow above is the single release path now and after. Moving to CI later replaces the skill's
steps, never the workflow. The one thing CI cannot do is the browser-consent `lnr auth login`; when
automation is designed, that item must be dropped or kept as a manual gate.

## Considered and rejected

**Independent plugin and binary versions.** They ship from one repo at one commit
([ADR-0006](0006-the-plugin-marketplace-lives-in-this-repo.md)), so a second number would be
bookkeeping with no information.

**An omitted plugin `version` (track the commit SHA).** Rejected above: it makes skill-ahead skew
the default on every merge.

**A "new version available" notice, or a self-updating binary.** Rejected above; it is the one
feature here that spends every caller's tokens to serve the occasional human.
