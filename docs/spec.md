# `lnr` v1 spec

The locked specification for `lnr`, a Go CLI that replaces the Linear MCP for daily agent-driven
Linear work. It is assembled from the resolved tickets of the
[wayfinder map](https://github.com/taciogt/lnr/issues/1) and the [ADRs](adr/). **Nothing here is
decided for the first time**: each section states a decision and points at the ticket or ADR that
holds its reasoning. Where this document and a ticket differ, the ticket's *latest amendment* wins
and this document has a bug.

**Linking.** Implementation issues link to a section by its heading anchor (e.g.
`docs/spec.md#5-agent-contract`). Section numbers and titles are stable; new material is appended
under a new number, never inserted.

**What is not here.** Per-command flag lists, help text and per-noun lean field sets beyond the
rules below belong to the implementation; the `--help` tree is the reference
([ADR-0005](adr/0005-the-companion-skill-never-duplicates-help.md)). The questions this spec could
not close are in [§15](#15-open-questions).

## Contents

1. [Purpose and targets](#1-purpose-and-targets)
2. [Command surface](#2-command-surface)
3. [Addressing](#3-addressing)
4. [Output](#4-output)
5. [Agent contract](#5-agent-contract)
6. [Authentication and credentials](#6-authentication-and-credentials)
7. [API client](#7-api-client)
8. [Settings](#8-settings)
9. [Companion plugin](#9-companion-plugin)
10. [Testing](#10-testing)
11. [Versioning and release](#11-versioning-and-release)
12. [Distribution](#12-distribution)
13. [Release phases and the Linear backlog](#13-release-phases-and-the-linear-backlog)
14. [Out of scope](#14-out-of-scope)
15. [Open questions](#15-open-questions)
16. [Where each decision lives](#16-where-each-decision-lives)

---

## 1. Purpose and targets

`lnr` exists to cut the **context cost** of Linear work for a coding agent. The measured baseline
(191 Linear MCP calls, 16 of 69 tools ever used, five tools carrying 83% of traffic) is in
[`README.md`](../README.md). Response payloads (~137,000 tokens) outweigh schema weight (~4,100
tokens per session) roughly 25:1, so **every output decision is judged in tokens**.

The caller is an agent shelling out with no TTY, and only secondarily a human. Design questions are
settled by what the caller can *do* with an answer ([`CONTEXT.md`](../CONTEXT.md), **Caller**).

| Target | Value | Decided in |
|---|---|---|
| `save_issue` equivalent (write confirmation) | ~23 tokens measured; test budget ≤185 characters | [Lean output](https://github.com/taciogt/lnr/issues/4), [Testing](https://github.com/taciogt/lnr/issues/16) |
| `issue get --lean` | ~120 tokens measured; test budget ≤925 characters | same |
| Companion plugin, resident | **≤100 tokens**, both skills combined | [Companion skill](https://github.com/taciogt/lnr/issues/8) |
| Companion skill, on invocation | ≤700 tokens | same |

An earlier ~150-token resident target is superseded by **≤100**.

## 2. Command surface

Decided in [Which commands ship in v1?](https://github.com/taciogt/lnr/issues/5), amended by
[the agent contract](https://github.com/taciogt/lnr/issues/7) and
[comment on a project](https://github.com/taciogt/lnr/issues/13).
Grammar: [ADR-0001](adr/0001-uniform-command-grammar-over-schema-shape.md).

Grammar is `lnr <noun> <verb> [flags]`. Create and update are always separate verbs. A noun's verb
set is uniform.

| Noun | `create` | `update` | `get` | `list` |
|---|:-:|:-:|:-:|:-:|
| `issue` | yes | yes | yes | yes |
| `project` | yes | yes | yes | yes |
| `milestone` | yes | yes | yes | yes |
| `comment` | yes | yes | yes | yes |
| `label` | yes | yes | yes | yes |
| `team` | | | | yes |
| `status` | | | | yes |

That is **22 noun-verb commands**. Beyond them: `lnr api '<graphql>'` (the escape hatch, §2.3),
`lnr auth login` (§6), `lnr --version` (§11) and `lnr help <topic>`, which carries the exit-code
table (`lnr help exit-codes`). None of these four count towards the 22.

### 2.1 Flags the decisions fix

| Flag | On | Meaning |
|---|---|---|
| `--team <KEY>` | `issue create` and other team-scoped commands | team key, resolved client-side; a raw UUID is also accepted |
| `--with-comments` | `issue get` | fetch comments in the same request |
| `--project <name\|uuid>` | `milestone create`/`list` (required); `comment create`/`list` | the project |
| `--issue <ref>` or `--project <name\|uuid>` | `comment create`/`list` | exactly one required; neither or both exits `2` |
| `--project-id`, `--milestone-id`, `--label-id` | name-addressed nouns | UUID escape hatch that skips name resolution |
| `--description-file <path\|->` | `issue create`/`update` | long text from a file; `-` reads stdin |
| `--body-file <path\|->` | `comment create` | same, for the comment body |
| `--timeout`, `--verbose`, `--json` | global | see §4 and §8 |

`comment get` and `comment update` address by UUID regardless of the comment's parent. `comment list`
with no parent is not allowed. There is no `project get --with-comments`.

### 2.2 Deferred to `lnr api`

Delete for every noun, `commentResolve`/`commentUnresolve`, documents and cycles wholesale,
initiatives, and comments on a project update, an initiative or a document. v1 ships **no
destructive command**, which is why there is no `--yes` flag.

### 2.3 `lnr api`

A raw GraphQL passthrough: query string in, Linear's response out. It is the one command where
"`--json` is not raw" does not apply. It maps exit codes like every command and **fails closed on
any GraphQL error**, with empty stdout and the full body on stderr
([ADR-0008](adr/0008-lnr-api-fails-closed-on-any-graphql-error.md)). On a clean response it prints
the body byte-for-byte with exit `0`.

## 3. Addressing

[ADR-0002](adr/0002-addressing-scheme-follows-api-capability-not-grammar.md), decided in
[#5](https://github.com/taciogt/lnr/issues/5).

| Noun | Accepted references |
|---|---|
| `issue` | identifier (`ENG-42`), UUID, or a pasted Linear URL (parsed client-side) |
| `project`, `label` | name (resolved client-side) or UUID |
| `milestone` | name together with its project, or UUID |
| `team` | short key (`ENG`), never a UUID, in any user-facing input |
| `comment` | UUID |

A name that matches several records exits `2` and the message lists the disambiguating UUIDs. A
reference that matches nothing exits `4`. The two are different failures on purpose.

## 4. Output

[ADR-0003](adr/0003-output-tiers-are-flag-only-and-json-is-not-raw.md), decided in
[What exactly does "lean" contain?](https://github.com/taciogt/lnr/issues/4).

- **Three tiers, chosen by flag only**: lean (default), `--verbose`, `--json`. Never inferred from a
  TTY. A TTY may gate interaction; it may never change output shape.
- **`--json` is lean's fields as JSON**, every key rendered, absent values as `null`. It is not a
  passthrough of Linear's response. Full fidelity is `--verbose --json`.
- **Lean and verbose are text, not a parsing contract.** `--json` is the only tier meant to be
  scripted against.
- **Nulls are omitted** in lean and verbose.
- **Verbose** is stable `key: value` lines.
- **Never echo input back on a write.** A write returns `id` and `url` only.
- **`issue get --lean`** truncates the description to ~200 characters and points at `--verbose`.
- **`issue list`** is tabular: id, priority, status, title.
- **A comment's output names its parent** by the reference a caller would pass back (`ENG-42` or the
  project name).
- **Tiers govern success only.** An error is full at every tier, so `--verbose` adds nothing to a
  failure. `--json` changes the encoding of an error, never its stream.

## 5. Agent contract

Decided in [What is the contract between `lnr` and a calling agent?](https://github.com/taciogt/lnr/issues/7).

### 5.1 Streams

stdout carries the payload and nothing else, and is **empty on failure**. Errors, warnings and the
OAuth URL go to stderr. There is no progress output.

### 5.2 Exit codes

Seven codes, keyed to what the caller should *do*, mapped from `extensions.code` and never from the
HTTP status (Linear returns a rate limit as HTTP 400).

| Code | Meaning | Caller's move |
|---|---|---|
| `0` | ok | none |
| `1` | unexpected or internal | give up, report the bug |
| `2` | bad input: unknown flag, missing argument, Linear `userError`, ambiguous reference | fix the input |
| `3` | auth required, or the credential was rejected | stop, ask the human |
| `4` | a reference did not resolve | fix the reference |
| `5` | rate limited | wait, then retry |
| `6` | network failure or timeout | retry once, then ask the human |

`issue get` on a missing reference exits `4`; `list` matching nothing exits `0` with empty stdout.
An eighth code is a breaking change ([§11](#11-versioning-and-release)).

### 5.3 Errors

Every error in the response array is shown, with Linear's `message`, `extensions.code` and
`userPresentableMessage`; `locations`, `meta` and `extensions.http` are dropped, except under
`lnr api`, which passes the array through. `--json` errors carry `"code"` (lnr's stable vocabulary,
the same as the exit codes) and `"linear_code"` (Linear's own). An invalid and a missing token
return an identical 401 from Linear, which is why the exit-`3` message names the credential source
(§6).

### 5.4 Never prompts, never retries

- `lnr auth login` is the only command that may open a browser or wait on a human. Every other
  command that lacks a usable credential exits `3` with one line.
- A rate limit is surfaced with its reset time and **never retried**.
- The default timeout is **30s** total, overridable (§8), folded into exit `6`. `auth login` is
  exempt, with a ~3 minute browser budget, then exit `6`.

### 5.5 Token refresh

Entirely the CLI's concern. `lnr` refreshes an expired token and completes the original request in
the same invocation, serialized behind a lock keyed by workspace ID with the timeout applied. The
only credential state a caller ever sees is exit `3`.

## 6. Authentication and credentials

Decided in [#2](https://github.com/taciogt/lnr/issues/2) and [#10](https://github.com/taciogt/lnr/issues/10),
with research in [`01-oauth.md`](../.scratch/lnr-v1-spec/research/01-oauth.md).

- **OAuth authorization code + PKCE (S256)** against `linear.app/oauth/authorize` and
  `api.linear.app/oauth/token`, with a **shipped non-secret `client_id`** and no `client_secret`.
  Loopback redirect `http://localhost:8123/callback`.
- **`LINEAR_API_KEY` is the fallback**, and the only path on a headless or SSH host (Linear has no
  device-code grant). API keys use a raw `Authorization: <KEY>` header; OAuth uses `Bearer`.
- **Tokens**: access token lives ~24h; refresh rotates with a 30-minute replay grace.
- **Precedence**: `LINEAR_API_KEY` beats a stored token. A rejected credential does **not** fall
  through to the Keychain; it exits `3` and names its source
  ([ADR-0009](adr/0009-one-credential-item-and-the-environment-credential-wins.md)).
- **Storage**: macOS Keychain through `/usr/bin/security`, never the native `SecItem*` API, keyed by
  workspace ID, with **at most one `lnr` item**
  ([ADR-0004](adr/0004-keychain-access-via-the-security-cli.md), ADR-0009). The reasoning is in
  [`07-keychain-non-interactive.md`](../.scratch/lnr-v1-spec/research/07-keychain-non-interactive.md).
  There is no plaintext-file fallback in a shipped build.
- **Admin dependency**: both OAuth and API keys can be disabled by a workspace admin; neither path
  avoids that.
- **Unverified**: that the one shipped `client_id` authorizes against a *different* workspace.
  Tracked as [HF-110](https://linear.app/taciogt/issue/HF-110), with `LINEAR_API_KEY` as the
  fallback if it fails.
- **Not specified**: the `auth` subcommand set beyond `login`, the OAuth scopes requested, and what
  happens when the loopback port is taken — see [§15](#15-open-questions).

## 7. API client

Decided in [Hand-written GraphQL client, or generated from introspection?](https://github.com/taciogt/lnr/issues/3),
research in [`02-graphql-client.md`](../.scratch/lnr-v1-spec/research/02-graphql-client.md).

- **Hand-written**, layered on the same untyped query-in / JSON-out path that `lnr api` already
  forces to exist. Not generated.
- **Drift check in CI**: the vendored `schema.graphql` is a test fixture only, and `gqlparser`
  validates every operation string against it.
- **Pagination** uses the `nodes` shortcut with a generic `{nodes, pageInfo}` struct.
- Linear's introspection is public and its errors carry real HTTP statuses.
- No Go API is promised ([§11](#11-versioning-and-release)); everything is under `internal/`.

## 8. Settings

[ADR-0009](adr/0009-one-credential-item-and-the-environment-credential-wins.md), decided in
[#17](https://github.com/taciogt/lnr/issues/17). **No config file in v1.**

| Setting | Flag | Env var | Default |
|---|---|---|---|
| Request timeout | `--timeout` | `LNR_TIMEOUT` | 30s |
| API URL | none | `LNR_API_URL` | `https://api.linear.app` |
| Keychain file | none | `LNR_KEYCHAIN` | login keychain |
| Output tier | `--verbose`, `--json` | none | lean |
| Credential | none | `LINEAR_API_KEY` | Keychain |

`LNR_API_URL` and `LNR_KEYCHAIN` are test seams: documented in the repo only, absent from `--help`,
not a stable interface. `LNR_API_URL` accepts `https://` anywhere and `http://` only on loopback,
else exit `2` before any request. There is no `auth status` in v1.

## 9. Companion plugin

[ADR-0005](adr/0005-the-companion-skill-never-duplicates-help.md),
[ADR-0006](adr/0006-the-plugin-marketplace-lives-in-this-repo.md),
[ADR-0007](adr/0007-the-pre-approval-allow-list-omits-lnr-api.md); decided in
[#8](https://github.com/taciogt/lnr/issues/8) and [#9](https://github.com/taciogt/lnr/issues/9).

- **Never duplicates `--help`**, which is rich by design and is the reference. **Zero reference
  files.** `lnr` stays fully navigable with no skill installed.
- **The body carries only** that a command exists (grammar rule, noun list, a gloss on `milestone`,
  `comment` and `status`), cross-command house rules where an agent lacking one does the wrong thing,
  and literal **recipes** where a flag's value must be looked up first.
- **`description`** anchors on "Linear", an identifier or `linear.app` URL, carries a negative clause
  for GitHub issues and PRs, and never says "ticket". The draft is in #9; its final wording comes
  from a one-off `skill-creator` eval run before first ship, not a CI gate.
- **Two skills**: the main one, and a setup skill with `disable-model-invocation: true` that writes
  the durable `allow` list into the user's settings. The list enumerates the seven nouns and
  **omits `lnr api`**.
- **Validate, don't emit**: CI extracts every command path and flag from the skill and asserts each
  exists in the built binary, with **set equality** on nouns.
- **Ships from this repo's own marketplace**, one version with the binary (§11). Skew is answered in
  the unknown-command error, which names the binary version.
- **CI is scripts only.** `claude plugin eval` is deferred.
- **Open for the implementation to settle by running it**: whether CI can
  `claude plugin marketplace add .` against the checked-out repo, which the budget check needs.

## 10. Testing

Decided in [How is `lnr` tested before it ships?](https://github.com/taciogt/lnr/issues/16),
including its addendum.

| When | What |
|---|---|
| **Merge** (Ubuntu) | schema check, skill-command check, `goreleaser check`, `go test -race` (fake GraphQL server, golden files, size budget, no-echo property, fake OAuth, fixture-vs-schema), black-box contract suite |
| **Before publishing** (blocking) | live smoke suite (write, read, cleanup per noun); three macOS Keychain tests (round-trip, build-A-writes/build-B-reads, refresh lock); a manual real `lnr auth login` |
| **After publishing** | `brew install taciogt/tap/lnr` then `lnr --version`; a failure pulls the release |

- Fixtures are **recorded from a throwaway workspace with no real data**, never scrubbed from a real
  one. Hand-written fixtures are marked docs-derived and validated against the schema.
- The contract is proven **black-box against the built binary**: all seven exit codes, stdout empty
  on failure, no prompt with stdin closed and no TTY under a hard timeout, one refresh for N
  parallel calls.
- Keychain code sits behind a macOS-only build constraint; non-macOS builds use a test-only
  persistent store and never ship.
- **No coverage threshold.**

## 11. Versioning and release

[ADR-0010](adr/0010-one-version-and-a-tag-is-the-only-release-trigger.md), decided in
[How is `lnr` versioned and released?](https://github.com/taciogt/lnr/issues/18).

- **Semver covers** command paths and flag names, the meaning of each exit code, and `--json` field
  names and types. Adding a field or command is minor; an eighth exit code is major. Not covered:
  lean and verbose layout, error and `--help` wording, stderr, `LNR_*` seams.
- **Phase v0 ships `0.x`**, `1.0.0` is the full v1 surface; the first tag is `v0.1.0`.
- **`lnr --version`** prints `lnr v1.0.3`, exit `0`; an untagged build prints `lnr vdev`.
- **No update notification and no self-update.**
- **One version** for binary and plugin, pinned in `plugin.json` and checked against the tag.
- **A pushed `vX.Y.Z` tag is the only release trigger.** One workflow: `verify`, macOS `keychain`,
  `smoke`, `publish`, `verify-install`. A project-scoped `/release` skill pushes the tag for now.

## 12. Distribution

[How does `lnr` reach a Mac?](https://github.com/taciogt/lnr/issues/6), research in
[`05-homebrew.md`](../.scratch/lnr-v1-spec/research/05-homebrew.md).

- A **personal tap** (`taciogt/homebrew-tap`) with GoReleaser **`homebrew_casks`**; install is
  `brew install taciogt/tap/lnr`. `brews:` is deprecated and is never written.
- **A cask install is quarantined and an unsigned binary is SIGKILLed on first run (exit 137)**, so
  the cask carries the `xattr -dr com.apple.quarantine` post-install hook. No signing or
  notarization. Do not add `url.verified`.
- One cask file serves arm64 and Intel; no universal binary.
- The tap needs a fine-grained PAT (`HOMEBREW_TAP_TOKEN`), since `GITHUB_TOKEN` cannot push to it.
- macOS only; `homebrew-core` is not attainable.
- **Licence**: MIT, one root `LICENSE`, copyright `Tácio Tavares`
  ([#11](https://github.com/taciogt/lnr/issues/11)).

## 13. Release phases and the Linear backlog

Handoff decisions are on [Assemble the locked v1 spec document](https://github.com/taciogt/lnr/issues/19).
Implementation is tracked in the Linear [`lnr` project](https://linear.app/taciogt/project/lnr-6b25dcc7130a),
through the Linear MCP until v0 is usable and through `lnr` afterwards.

**v0** is the first usable subset of v1: `auth login` and `LINEAR_API_KEY`, `issue create`/`update`/
`get`/`list`, `comment create`, `status list`, `lnr api`, and a working `brew install`. v0 never
contains anything v1 lacks. **v1** is everything in this spec. **v2+** is not specified.

The slices, their order and blocking edges live in the Linear project, which is the single record;
each issue's "Spec:" line links the sections it implements.

**Dogfooding.** After v0, `lnr` tracks its own implementation and its token usage is compared with
the MCP's by the **same method as the README baseline** (transcript-derived response sizes per
call, characters to tokens the same way). The Linear MCP calls made while tracking v0 are the
"before" sample; keep those transcripts.

## 14. Out of scope

- **Multi-workspace and profiles.** One account per machine; storage is keyed by workspace ID so
  this stays addable.
- **Reimplementing the 53 unused MCP tools.** `lnr api` covers them.
- **Non-macOS distribution.**
- **Initiatives, documents and cycles as nouns**, and comments on them
  ([#14](https://github.com/taciogt/lnr/issues/14), [#15](https://github.com/taciogt/lnr/issues/15)).
- **A config file**, a default team, `auth status`, an update notice and self-update: each ruled out
  for v1 above and addable later without breaking a caller.

## 15. Open questions

Found while assembling this document. Each is a new grilling ticket on the map rather than a choice
made here; none blocks an implementation slice that does not touch it.

- [Which `auth` commands exist, and what does `lnr auth login` request?](https://github.com/taciogt/lnr/issues/20)
  Whether `auth logout` exists, which OAuth scopes are requested, and what happens when loopback
  port 8123 is busy. Touches §6 and the auth slice (HF-98).
- [How do `list` commands filter and paginate?](https://github.com/taciogt/lnr/issues/21)
  Filter flags, default limit, and how truncation is signalled. Touches §4 and `issue list` (HF-99).
- [What does lean contain for project, milestone, label, comment, team and status?](https://github.com/taciogt/lnr/issues/22)
  Per-noun lean fields. Touches §4 and the v1 noun slices (HF-105 to HF-108).

## 16. Where each decision lives

| Decision | Ticket | ADR | Section |
|---|---|---|---|
| OAuth design | [#2](https://github.com/taciogt/lnr/issues/2), [#10](https://github.com/taciogt/lnr/issues/10) | | §6 |
| API client | [#3](https://github.com/taciogt/lnr/issues/3) | | §7 |
| Lean output | [#4](https://github.com/taciogt/lnr/issues/4) | [0003](adr/0003-output-tiers-are-flag-only-and-json-is-not-raw.md) | §4 |
| Command surface | [#5](https://github.com/taciogt/lnr/issues/5), [#13](https://github.com/taciogt/lnr/issues/13) | [0001](adr/0001-uniform-command-grammar-over-schema-shape.md), [0002](adr/0002-addressing-scheme-follows-api-capability-not-grammar.md), [0008](adr/0008-lnr-api-fails-closed-on-any-graphql-error.md) | §2, §3 |
| Distribution | [#6](https://github.com/taciogt/lnr/issues/6) | | §12 |
| Agent contract | [#7](https://github.com/taciogt/lnr/issues/7) | [0004](adr/0004-keychain-access-via-the-security-cli.md) | §5, §6 |
| Companion skill | [#8](https://github.com/taciogt/lnr/issues/8), [#9](https://github.com/taciogt/lnr/issues/9) | [0005](adr/0005-the-companion-skill-never-duplicates-help.md), [0006](adr/0006-the-plugin-marketplace-lives-in-this-repo.md), [0007](adr/0007-the-pre-approval-allow-list-omits-lnr-api.md) | §9 |
| Licence | [#11](https://github.com/taciogt/lnr/issues/11) | | §12 |
| Testing | [#16](https://github.com/taciogt/lnr/issues/16) | | §10 |
| Settings and precedence | [#17](https://github.com/taciogt/lnr/issues/17) | [0009](adr/0009-one-credential-item-and-the-environment-credential-wins.md) | §6, §8 |
| Versioning and release | [#18](https://github.com/taciogt/lnr/issues/18) | [0010](adr/0010-one-version-and-a-tag-is-the-only-release-trigger.md) | §11 |
| Cross-workspace `client_id` | [#12](https://github.com/taciogt/lnr/issues/12) → HF-110 | | §6 |
