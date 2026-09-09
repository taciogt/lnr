# Keychain GUI-prompt risk for an unsigned `lnr` binary

Probed live: macOS 26.5.2 (25F84), Darwin 25.5.0, arm64, go1.27.1 darwin/arm64, keychain client
`github.com/keybase/go-keychain v0.0.1`. Every command ran with **no TTY** (`tty` -> `not a tty`),
which is the agent's condition. Login keychain unlocked throughout.

## Answer

**YES — a non-interactive `lnr` invocation can hang indefinitely on a Keychain GUI dialog.**
Observed 7 times; each time the client blocked until SIGKILL. TTY is irrelevant: `SecurityAgent`
draws the dialog in the GUI session and the caller blocks inside `SecItemCopyMatching`.

7 GUI dialogs appeared on the user's screen during this probe. Every blocked process was killed
by the timeout harness. No hung process and no probe keychain item remain (verified: 0 matches
for `lnr-probe` in `login.keychain-db`).

## Mechanism: the ACL `partition_id` entry, keyed by cdhash

`security dump-keychain -a ~/Library/Keychains/login.keychain-db` (OBSERVED):

| item | created by | `decrypt` ACL applications | `partition_id` |
|---|---|---|---|
| `lnr-probe-test` | `/usr/bin/security` | `/usr/bin/security`, req `identifier "com.apple.security" and anchor apple` | `apple-tool:` |
| `lnr-probe-allowall` | `security ... -A` | `<null>` (allow all apps) | `apple-tool:` |
| `lnr-probe-go` | Go binary `probeA` | `probeA`, req `cdhash H"60993b48..."` | `cdhash:60993b486117254204fba53748a0fca5ba832197` |

Note the Go item's **entry 0 is `encrypt` with `applications: <null>`** — writes are allow-all.
Only `decrypt` is cdhash-bound. That asymmetry drives the mitigations below.

`codesign -dv --verbose=4` on every Go build: `flags=0x20002(adhoc,linker-signed)`,
`Identifier=a.out`, `TeamIdentifier=not set`, `Signature=adhoc`, `Internal requirements=none`.
`codesign -d -r-` across rebuilds (one string literal changed each time):

```
# designated => cdhash H"60993b486117254204fba53748a0fca5ba832197"   probeA
# designated => cdhash H"5bbb2b5b7aa5d5c36581eee0fe81a0718dd9d44e"   probeB
# designated => cdhash H"72b8d70a541f360474bebcbd14c9fce0b76ee991"   probeC
# designated => cdhash H"c102d1eae63d2917a673c5b491b6ed875e890a5e"   probeD
# designated => cdhash H"39777d568036df0db7a8c270e921eb418c190f17"   probeE
```

**darwin/arm64 Go binaries are ad-hoc, linker-signed; their designated requirement is the cdhash,
which changes on every rebuild.** No identifier or Team ID clause exists to anchor an ACL to.

## Observed behaviour matrix

| scenario | result |
|---|---|
| `/usr/bin/security` writes, `/usr/bin/security` reads | no prompt, `exit=0` |
| Go binary A writes, **same** binary A reads | no prompt, `exit=0` |
| Go binary A reads item created by `/usr/bin/security` | **DIALOG, BLOCKED, killed** |
| **Go binary B (rebuilt) reads item written by binary A** | **DIALOG, BLOCKED, killed** |
| Go binary A *or* B reads item created with `security -A` | **DIALOG, BLOCKED, killed** |
| **`/usr/bin/security` reads a Go-created item** | **DIALOG, BLOCKED, killed** |
| Go binary B **deletes** item written by binary A (`SecItemDelete`) | fails fast, no prompt, `-25244 errSecInvalidOwnerEdit` |
| Go binary B **re-adds** at that service | fails, `-25299 errSecDuplicateItem` (old item still there) |
| Go binary B **updates in place** (`SecItemUpdate`) item written by A | **succeeds, no prompt**, `err=<nil>` |
| Go binary B reads the item it just successfully updated | **DIALOG, BLOCKED, killed** |
| `/usr/bin/security delete-generic-password` on a Go-created item | **succeeds, no prompt** |
| Go binary B writes fresh item after that delete, then reads it | no prompt, `val="secretFRESH"` |

Mapped to `lnr`'s lifecycle:

- **First read after install** — no prompt, *provided the same binary wrote the token*.
- **First read after `brew upgrade`** — **PROMPT AND HANG.** New cdhash, not in the trusted-app
  ACL nor the partition list. Confirmed, not inferred. Recurs on *every* upgrade.
- **Token refresh after upgrade does not fix it.** `SecItemUpdate` succeeds silently (the
  `encrypt` ACL is allow-all) but the `decrypt` ACL keeps the old cdhash, so the very next read
  still hangs. Writing a fresh 24h token into the existing item is a trap: it looks like it worked.
- **Never** for reads by the exact binary bytes that wrote the item.

`-A` **does not help.** It correctly produced `applications: <null>` on the decrypt ACL, yet both
Go builds still hung: `partition_id` stayed `apple-tool:`, and the partition check is evaluated
independently of the app ACL.

## Recommended mitigation (both halves VERIFIED)

### 1. Disable the dialog: `SecKeychainSetUserInteractionAllowed(false)`

cgo, `-framework Security`, called once at startup before any keychain access:

```
setNoUI -> 0
own-item   read err=<nil> val="secretFRESH"
foreign    read err=The user name or passphrase you entered is not correct. (-25293) val=""
own again  read err=<nil> val="secretFRESH"
```

No dialog, no block; the unauthorized read returns `-25293 errSecAuthFailed` immediately.
The setting is process-wide and persists for the process's lifetime, and (VERIFIED by the
third read above) it does **not** poison subsequent *authorized* reads — the failure is per-item.
The API is deprecated ("first deprecated in macOS 10.10", compile-time warning only) but
functional on macOS 26.5.2.

This converts an indefinite hang into a fast, catchable error. It does not grant access.

### 2. Recover by shelling out `security delete-generic-password`, then re-auth

`SecItemDelete` from the new build cannot remove the stale item (`-25244`), but Apple's tool can,
with no prompt and **no secret in argv**:

```
/usr/bin/security delete-generic-password -s <service> -a <account>
-> "password has been deleted."   exit=0
```

Then the fresh binary writes a new item and owns it (`write err=<nil>`, read back `secretFRESH`).

So the post-upgrade flow is: UI disabled -> read -> on `-25293`, `security delete-generic-password`
-> run the OAuth loopback flow -> write natively. One browser login per `brew upgrade`, which is
tolerable given Linear tokens expire every 24h anyway (`01-oauth.md`). No allow-all exposure, and
the token never passes through a command line.

## Rejected alternative: shell out to `/usr/bin/security` for everything

Works, but is worse than the above. VERIFIED that it never prompts *if `security` both writes and
reads*, including across a rebuild of the caller:

```
probeC writesec lnr-probe-shell -> err=<nil>
probeC readsec  lnr-probe-shell -> "secretSHELL"  exit=0
probeD (different cdhash) readsec lnr-probe-shell -> "secretSHELL"  exit=0
```

Rejected because: (a) the item becomes readable with no prompt by *any* process on the machine
via `security find-generic-password -w`; (b) the write passes the token in argv
(`-w <token>`), visible in `ps` for the duration of the call; (c) a process spawn per call.
It is only prompt-free while `security` owns the item — `security` reading a *Go*-created item
hangs (row 6 of the matrix).

## What I could NOT establish

- Whether Developer ID signing yields a stable `teamid:` partition that survives rebuilds.
  No signing identity on this machine. INFERRED from the table (Apple's tool got an
  anchor/identifier requirement and an `apple-tool:` partition rather than a cdhash), but untested.
- `security add-generic-password -T ""` was not tested. `-A` was, and failed; since the partition
  list is the gate and `-T` only edits the app ACL, `-T ""` is unlikely to help — inference only.
- `security set-generic-password-partition-list` not tested: it needs the login keychain password
  via `-k` or an interactive prompt, defeating the purpose for a non-interactive CLI.
- The data-protection keychain (`kSecUseDataProtectionKeychain`) not tested. go-keychain v0.0.1
  exposes no `SecAccess`, `SecTrustedApplication`, ACL, or data-protection knob at all (grep over
  its sources found no such symbols), so `lnr` needs its own cgo for any of this. That keychain
  also requires a `keychain-access-group` entitlement, which requires signing.
- Behaviour with the GUI session locked or over SSH-only. Not tested; a GUI session was logged in
  throughout, as specified.
- I did not read the dialog's on-screen text. "Dialog appeared" is established by `SecurityAgent`
  spawning plus the client blocking, not by reading the window.

## Method note

Timeout harness used on every keychain call (exit 142 = blocked):

```perl
my $secs = shift @ARGV;
my $pid = fork();
if ($pid == 0) { open(STDIN, "<", "/dev/null"); exec @ARGV; exit 127; }
$SIG{ALRM} = sub { kill 9, $pid; waitpid($pid,0); print "*** TIMEOUT_BLOCKED (killed) ***\n"; exit 142; };
alarm $secs; waitpid($pid, 0); alarm 0;
printf "exit=%d\n", $? >> 8;
```

`perl -e 'alarm 15; exec @ARGV'` does **not** work — the timer does not survive the exec and a
blocked read ran past 120s. The fork+waitpid form is required.

## Cleanup

`security delete-generic-password` run for `lnr-probe-test`, `lnr-probe-allowall`, `lnr-probe-go`,
`lnr-probe-shell`, `lnr-probe-rw`, `lnr-probe-rw2`. `security dump-keychain | grep -c lnr-probe`
-> `0`. No `probe*` processes remain. `SecurityAgent` stays resident as an idle `UIElement` at
0% CPU with no client — its normal post-use state.

---

## Resolution (added while closing #7): `gh` already solves this, differently

The mitigation recommended above (cgo `SecKeychainSetUserInteractionAllowed(false)` + shell-delete
recovery + re-login per upgrade) was **rejected** in favour of an option this probe tested and
dismissed: route *all* keychain access through `/usr/bin/security`.

What decided it was evidence gathered after the probe, from `gh` on this machine — a binary in
**identically** the position `lnr` would be in:

```
$ codesign -dv --verbose=2 /opt/homebrew/bin/gh
CodeDirectory ... flags=0x20002(adhoc,linker-signed)
TeamIdentifier=not set
$ codesign -d -r- /opt/homebrew/bin/gh
# designated => cdhash H"8c7df36d97ceba5ccbe1dc65f97e140dd1558db4"
```

`gh` is ad-hoc linker-signed with a bare-cdhash designated requirement, so its hash rotates on
every release — the exact condition that produces the dialog above. It has nonetheless never
prompted on this machine across upgrades since April. Why:

```
$ strings -a /opt/homebrew/bin/gh | grep generic-password
add-generic-password -U -s %s -a %s -w %s
find-generic-password
delete-generic-password
```

That is `zalando/go-keyring`'s `os/exec` path. (`gh` also links `Security.framework`, but that is
`crypto/x509` reaching the system root store, not keychain access — do not read it as native
`SecItem*` use.) The resulting ACL, OBSERVED via `security dump-keychain -a`:

```
        authorizations (1): partition_id
        description: apple-tool:
        applications: <null>
        description: gh:github.com
```

**`apple-tool:`, not `cdhash:`.** The partition is anchored to Apple's tool, so the caller's own
code identity never enters the check — which is why cdhash rotation is structurally irrelevant on
this path, rather than merely not-yet-observed.

Two arguments carried the decision beyond what this probe weighed:

1. The probe's own §"trap" finding — post-upgrade `SecItemUpdate` succeeding while the `decrypt`
   ACL keeps the stale cdhash — bites on **24h token refresh**, i.e. daily, not per-upgrade. The
   `security -U` path has no such split between encrypt and decrypt authority.
2. The argv exposure this probe rejected the `security` path *for* does not survive its own threat
   model. It is visible only to a same-user process — and a same-user process can already read an
   `apple-tool:`-partitioned item via `security find-generic-password -w`, no prompt. On a machine
   where a Claude Code agent shells out with these credentials, same-user is not the boundary being
   defended.

Recorded as [ADR-0004](../../../docs/adr/0004-keychain-access-via-the-security-cli.md). The
probe's findings above stand as observed; only its recommendation was overturned.
