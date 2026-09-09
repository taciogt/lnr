# Keychain access goes through `/usr/bin/security`, not the native `SecItem*` API

Decided in [#7](https://github.com/taciogt/lnr/issues/7). `lnr` stores its Linear token in the
macOS Keychain (a decision this ADR qualifies rather than reverses), but reaches it by shelling out
to `/usr/bin/security` the way `gh` does — not through `SecItemCopyMatching` and friends. The
obvious reading is that this is the lazy option. It is the opposite: the native API is the one that
breaks.

**Why the native path fails here.** Go binaries on darwin/arm64 are ad-hoc linker-signed, so the
`cdhash` in their designated requirement changes on **every rebuild**, and there is no Team ID to
anchor an ACL to. Keychain items carry a `partition_id` ACL entry keyed on that hash, checked
independently of the trusted-application list. Probed live (see
`.scratch/lnr-v1-spec/research/07-keychain-non-interactive.md`):

- A rebuilt binary reading the previous build's item **blocks on a `SecurityAgent` GUI dialog** —
  no TTY required for the dialog to appear, and no TTY available to dismiss it. That is
  `brew upgrade`, and it hangs the next non-interactive invocation indefinitely.
- `security -A` (allow-all) does not help: `partition_id` stays `apple-tool:` while the read still
  hangs.
- Worse, the item's `encrypt` ACL is allow-all while only `decrypt` is cdhash-bound. So after an
  upgrade a token refresh **succeeds silently** and leaves an unreadable item. That failure lands
  on the 24h refresh cycle — daily, not per-upgrade.
- The new build cannot even clean up: `SecItemDelete` returns `-25244 errSecInvalidOwnerEdit`.

**Why the `security` CLI works.** Items it creates are partitioned to `apple-tool:` — anchored to
Apple's tool, so the caller's own code identity never enters the check and cdhash rotation is
structurally irrelevant. Confirmed on `gh`, which is ad-hoc linker-signed with a bare-cdhash
requirement exactly as `lnr` will be, uses this path (`add-generic-password -U -s %s -a %s -w %s`
appears in its binary), and has never prompted across upgrades.

**The cost, stated plainly.** `security add-generic-password -w <secret>` puts the token in argv,
and once written, any same-user process can read it back with no prompt. This was judged acceptable
rather than ignored: a same-user process could read an `apple-tool:` item regardless of how it was
written, and on a machine where a coding agent shells out with these credentials, same-user is not
the boundary being defended. Trading that for an availability failure the agent cannot see, retry,
or report was the worse deal.

**Considered and rejected:** cgo `SecKeychainSetUserInteractionAllowed(false)` (verified to convert
the hang into `-25293 errSecAuthFailed`, but costs cgo against the clean cross-compile in
[#6](https://github.com/taciogt/lnr/issues/6), a deprecated-since-10.10 API, and a browser re-login
per upgrade); a plaintext `0600` file (this is `gh`'s *degraded* mode, not its behaviour); Developer
ID signing, which would give a stable `teamid:` partition and remove the problem entirely, but
reverses #6's "no signing, no notarization, no Apple Developer fee" — and was never established
first-hand, only inferred.

Note for anyone reaching for a library: `keybase/go-keychain` v0.0.1 exposes no `SecAccess`/ACL
knob at all, so the native path would have required hand-rolled cgo regardless.
