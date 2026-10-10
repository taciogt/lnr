# One credential item, and an environment credential wins without fallback

Decided in [#17](https://github.com/taciogt/lnr/issues/17). `lnr` has no config file in v1, so
every setting is a flag, an environment variable or a default. Two rules about *credentials*
follow from that and are hard to reverse once callers script against them.

**The Keychain holds at most one `lnr` item.** Its account attribute is the workspace ID, as
[ADR-0004](0004-keychain-access-via-the-security-cli.md) requires, but `lnr` reads it by service
name alone and parses the account back out. `auth login` deletes any existing item before writing
the new one. Probed on a throwaway keychain: `security find-generic-password -s lnr` with two items
silently returns the first, and exits 44 with none, so the single-item invariant is what makes a
service-only lookup deterministic. It also keeps "one account per machine" structural, with no
pointer item and no state file to drift.

**`LINEAR_API_KEY` beats a stored token, and a rejected key does not fall through.** The
environment is the only channel a headless caller controls, so it wins. If the winning credential
is rejected the result is exit `3` with an error that names its source (`LINEAR_API_KEY` vs the
stored token), because Linear returns an identical 401 for an invalid token and for none. Falling
back to the Keychain would hide a misconfigured key behind a working login.

## Considered and rejected

**A config file** (default team, timeout). Not because of output shape — [ADR-0003](0003-output-tiers-are-flag-only-and-json-is-not-raw.md)
does not apply — but because hidden state changes *where a write lands*, adds a path, format and
permissions, and callers already pass `--team`. Adding a default team later breaks nothing.

**A pointer item or state file naming the current workspace.** Extra storage to keep in sync with
the Keychain, for a case (several workspaces) that is out of scope.

## Settings that exist

| Setting | Flag | Env var | Default |
|---|---|---|---|
| Request timeout | `--timeout` | `LNR_TIMEOUT` | 30s |
| API URL | none | `LNR_API_URL` | `https://api.linear.app` |
| Keychain file | none | `LNR_KEYCHAIN` | login keychain |
| Output tier | `--verbose`, `--json` | none | lean |
| Credential | none | `LINEAR_API_KEY` | Keychain |

`LNR_API_URL` and `LNR_KEYCHAIN` are test seams: documented in the repo, absent from `--help`, and
not a stable interface. `LNR_API_URL` accepts `https://` anywhere and `http://` only for loopback
(`localhost`, `127.0.0.1`, `[::1]`); anything else exits `2` before a request is sent, since the
token travels in the request. `LNR_KEYCHAIN` applies to `auth login` writes as well as reads.
