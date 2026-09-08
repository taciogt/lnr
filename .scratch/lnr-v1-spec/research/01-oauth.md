# Research: can a Homebrew-shipped `lnr` do a `gh auth login` browser flow?

Resolves `.scratch/lnr-v1-spec/issues/01-oauth-loopback-public-client.md`.
Date: 2026-09-08. Sources: Linear's own developer docs and help docs, plus live probes of
Linear's OAuth endpoints (curl transcripts inline).

## Verdict

**Yes — the `gh auth login` flow is viable, with one caveat that is a distribution problem, not
a protocol problem.**

Linear's *first-party* OAuth server (`linear.app/oauth/authorize` + `api.linear.app/oauth/token`)
documents PKCE with `client_secret` **optional** on both the initial code exchange and the
refresh, and its own examples use `http://localhost:3000/oauth/callback` as the redirect URI.
That is exactly the public-client loopback shape `lnr` needs.

The caveat: **Dynamic Client Registration is not a shortcut.** It belongs to `mcp.linear.app`
only, and clients registered there are rejected by `api.linear.app/oauth/token`. `lnr` must ship
a hard-coded `client_id` from an OAuth app that a human created once in the Linear UI. That is
fine (it is what `gh` does), but it means the repo owner registers one app and ships its
`client_id` in the binary — a public, non-secret value.

## Sub-question 1 — Does a Linear OAuth app accept a `http://localhost:PORT` loopback redirect URI?

**Strongly supported by primary docs; not empirically confirmed** (confirming it requires
creating an app in a real workspace, which I deliberately did not do — see "How to close the
last gap").

Evidence:

1. Linear's own authorization-request examples use a loopback redirect, including the PKCE
   variant — source <https://linear.app/developers/oauth-2-0-authentication>:

   ```http
   GET https://linear.app/oauth/authorize?client_id=client1&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Foauth%2Fcallback&response_type=code&scope=read,write HTTP/1.1
   ```
   ```http
   GET https://linear.app/oauth/authorize?client_id=client1&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Foauth%2Fcallback&response_type=code&scope=read,write&code_challenge=challenge&code_challenge_method=S256 HTTP/1.1
   ```

2. The app-manifest schema constrains `oauth.redirect_uris` only to
   *"Format: absolute HTTP or HTTPS URL"*, min 1 / max 32, unique — plain `http` is explicitly
   allowed, and **no** loopback restriction is stated. The manifest docs restrict loopback hosts
   only for *webhook* URLs, not redirect URIs.
   Source: <https://linear.app/developers/oauth-app-manifests>

3. `mcp.linear.app`'s DCR endpoint accepts loopback redirects outright (probe below), which at
   minimum shows Linear's auth stack has no blanket loopback ban.

**Design note:** because the redirect URI must be registered up-front and Linear does not
document port-wildcarding, register a small fixed set of ports (e.g. `http://localhost:8123/…`,
`:8124`, `:8125`) rather than assuming a random ephemeral port will be accepted. `gh` does the
same thing. The 32-URI cap gives plenty of headroom. Path and scheme must match exactly.

## Sub-question 2 — Can a public client with no `client_secret` get a token valid against `api.linear.app/graphql`?

**Two distinct answers, and the distinction is the important finding of this ticket.**

### 2a. DCR / `token_endpoint_auth_method: "none"` is scoped to `mcp.linear.app` only — PROVEN

`mcp.linear.app` happily registers an anonymous public client with loopback redirects:

```
$ curl -s -i -X POST https://mcp.linear.app/register -H 'Content-Type: application/json' \
    -d '{"client_name":"lnr-research-probe",
         "redirect_uris":["http://localhost:8123/callback","http://127.0.0.1:8123/callback"],
         "grant_types":["authorization_code","refresh_token"],
         "response_types":["code"],
         "token_endpoint_auth_method":"none",
         "application_type":"native"}'
HTTP/2 201
{"client_id":"GcwYZSKR3qavd9Tl",
 "redirect_uris":["http://localhost:8123/callback","http://127.0.0.1:8123/callback"],
 "token_endpoint_auth_method":"none", ...}
```

That `client_id` is then recognised by the MCP token endpoint but **rejected outright** by the
main Linear token endpoint:

```
$ curl -s -X POST https://mcp.linear.app/token \
    -d 'grant_type=authorization_code&code=bogus&client_id=GcwYZSKR3qavd9Tl&redirect_uri=http://localhost:8123/callback&code_verifier=aaaa…'
{"error":"invalid_grant","error_description":"Invalid authorization code format"}    <-- client accepted, code rejected

$ curl -s -X POST https://api.linear.app/oauth/token \
    -d 'grant_type=authorization_code&code=bogus&client_id=GcwYZSKR3qavd9Tl&redirect_uri=http://localhost:8123/callback&code_verifier=aaaa…'
{"error":"invalid_client","error_description":"Invalid client: client is invalid"}    <-- client unknown
```

Two separate client registries. Confirmed further by the MCP's protected-resource metadata,
which scopes its tokens to a single resource:

```
$ curl -s https://mcp.linear.app/.well-known/oauth-protected-resource/mcp
{"resource":"https://mcp.linear.app/mcp","authorization_servers":["https://mcp.linear.app"],
 "scopes_supported":["read","write"],"bearer_methods_supported":["header"]}
```

**Conclusion (proven): the two hosts keep separate client registries**, so a DCR-issued
`client_id` cannot be used to run the authorization-code exchange at `api.linear.app/oauth/token`.

**Open, and worth more than it looks (inference only):** could `lnr` use `mcp.linear.app` purely
as its *authorization server* — DCR a public client, run the flow at `mcp.linear.app/authorize`
and `mcp.linear.app/token` — and then use the resulting access token directly against
`api.linear.app/graphql`, never touching the MCP transport? If that worked, `lnr` would need no
pre-registered app, no shipped `client_id`, and no app-creation admin gate at all — a strictly
better distribution story than the recommendation below. I did **not** test it (it needs a real
browser authorization). It is *likely* to fail: the protected-resource metadata above names a
single `resource` (`https://mcp.linear.app/mcp`), which is RFC 8707 resource-indicator audience
scoping and points at the token being rejected elsewhere. But that is inference, not observation,
and it is cheap to settle — see "How to close the last gap".

### 2b. A hard-coded public client on the *first-party* OAuth server — **confirmed live in #10**

`https://linear.app/developers/oauth-2-0-authentication` documents a PKCE flow whose token
request parameters are:

| Parameter | Status (per Linear docs) |
|---|---|
| `code` | Required |
| `redirect_uri` | Required |
| `client_id` | Required |
| `client_secret` | **Optional** |
| `code_verifier` | Required |
| `grant_type` | Required (`authorization_code`) |

(For the non-PKCE authorization-code flow the same page marks `client_secret` **required** — so
"optional" is specifically the PKCE affordance, which is the public-client case.)

Token endpoint: `POST https://api.linear.app/oauth/token` — i.e. the token is issued by the same
host that serves `api.linear.app/graphql`, and the docs describe using it as
`Authorization: Bearer <ACCESS_TOKEN>` against the GraphQL API
(<https://linear.app/developers/graphql>). There is no documented notion of an
`api.linear.app`-issued OAuth token that is not valid at `/graphql`.

**Honest limit on this claim.** The only thing I could probe unauthenticated is that
`api.linear.app/oauth/token` accepts a POST with **no** `client_secret` and **no** basic-auth
header and fails on `invalid_client` (unknown client id) rather than on a missing-secret error:

```
$ curl -s -X POST https://api.linear.app/oauth/token \
    -d 'grant_type=authorization_code&code=x&client_id=00000000-…&redirect_uri=http://localhost:8123/callback&code_verifier=aaaa…'
{"error":"invalid_client","error_description":"Invalid client: client is invalid"}
```

That was weak evidence on its own: the server never got far enough to check a secret. **#10 closed
the gap with a real `client_id`** (`1ef6a5d2c62d8863c302b917e0ab8c3f`): a secretless PKCE exchange
at `api.linear.app/oauth/token` returned a working `access_token`, and the `viewer` query against
`/graphql` with that token returned real account data. The positive half of 2b is now observed,
not inferred.

## Sub-question 3 — Is creating an OAuth app gated on workspace admin?

Two separate gates, and they land on different people.

### 3a. Creating the app (affects only the `lnr` maintainer, once)

**Almost certainly admin-gated; not stated verbatim anywhere I could find — recorded as
inference.** Linear's help docs place OAuth app creation in the admin area:
*"Create and manage webhooks and OAuth applications in Settings > Administration > API."*
(<https://linear.app/docs/api-and-webhooks>). The workspace-owner permission list includes
*"Integrations and API (enabling/disconnecting integrations, managing API settings, creating
webhooks and OAuth apps)"* (<https://linear.app/docs/workspace-owner>), and the developer docs
state webhook creation *"is restricted to workspace admins"* (<https://linear.app/developers/webhooks>).
I could not find a permissions matrix row that says in so many words whether a plain Member can
create an OAuth app; `https://linear.app/docs/member` 404s.

**This does not matter much for the design**: only the maintainer creates the app, once, and the
user has admin. Set `distribution=public` so other workspaces can install it (default is
`private` = current workspace only — <https://linear.app/developers/oauth-app-manifests>). I
found **no** documented Linear review or approval step required to mark an app public; the
Integration Directory listing is a separate, optional thing. Unverified.

### 3b. Authorizing the app (affects every `lnr` user) — this is the real admin dependency

Linear has a workspace setting, **Third-Party App Approvals**:

> "A workspace owner in an Enterprise workspace—or a workspace admin in any other paid
> workspace—can turn on third-party application approvals … When a workspace member tries to
> install a third-party app, Linear will display a screen which allows the member to request
> approval and optionally include a reason for their request."
> — <https://linear.app/docs/third-party-application-approvals>

Available on any paid plan. **I could not establish whether it is on or off by default**, nor
whether it applies to `distribution=private` apps. So: for some users, `lnr auth login` will end
in "request sent to your admin" rather than a token. The CLI should detect and explain that
failure mode rather than looping. This is unavoidable for *any* OAuth app and is not a reason to
prefer API keys — note that API keys have their own admin gate (3c/4 below).

### 3c. API keys are also admin-gatable

> "Admins can choose whether or not Members can create their own API keys from Settings >
> Administration > API > Member API keys. This setting will not apply to Admins who can always
> create API keys."
> — <https://linear.app/docs/api-and-webhooks>

So "API key fallback avoids the admin dependency" is **false**. Both paths can be
administratively disabled; they are just disabled by different switches.

## Sub-question 4 — Can personal API keys be scoped read-only?

**Yes. Confirmed by Linear's help docs.**

> "Admins and permitted Members can create personal API keys from Settings > Account > Security
> & Access. For each key you create, you can choose to give it full access to the data your user
> can access, or restrict it to certain permissions (*Read, Write, Admin, Create issues, Create
> comments*). You can also limit an API key's access to specific teams in your workspace."
> — <https://linear.app/docs/api-and-webhooks>

Also: *"Personal API keys can be generated with specific permissions and scoped to individual
teams"* (<https://linear.app/docs/security-and-access>), and Linear's own MCP FAQ refers to using
*"restricted read-only API keys"* (<https://linear.app/docs/mcp>).

API keys authenticate with a **raw** header — `Authorization: <API_KEY>`, **no** `Bearer` prefix
— whereas OAuth tokens use `Authorization: Bearer <ACCESS_TOKEN>`
(<https://linear.app/developers/graphql>). `lnr` must branch on credential type when building the
header; this is a real, easy-to-miss difference.

No key expiry is documented. Unknown whether keys expire at all (they appear not to).

## Sub-question 5 — Token lifetimes and refresh

From <https://linear.app/developers/oauth-2-0-authentication>:

| Fact | Value |
|---|---|
| Access token lifetime | **24 hours** — *"The access token is valid for 24 hours and will need to be refreshed when it expires."* Observed `"expires_in": 86399` in the documented response. |
| Refresh token | Issued alongside the access token; the refresh response returns a **new** refresh token (rotation). |
| Refresh + PKCE public client | `client_secret` is *"optional if you're using HTTP basic authentication **or refreshing a token generated using PKCE**"* — **so the secretless design survives past hour 24.** |
| Refresh token absolute lifetime | **Not documented. Unknown.** |
| Rotation replay tolerance | *"30-minute grace period to allow for network errors"* on replaying a refresh request. |
| Revocation | `POST https://api.linear.app/oauth/revoke` |
| Client-credentials tokens (not relevant here) | valid 30 days |
| Scopes | `read` (default), `write`, `issues:create`, `comments:create`, `timeSchedule:write`, `admin` |

**Agent-session implication:** a 24-hour access token means a long-lived agent *will* hit
expiry. `lnr` must store the refresh token (Keychain, keyed by workspace ID per the map) and
refresh transparently — with the 30-minute grace window making a concurrent-refresh race
survivable, but concurrent `lnr` invocations from an agent still argue for a lock or
last-writer-wins around the stored refresh token.

## Recommendation

1. Primary path: **OAuth authorization-code + PKCE (S256) with a loopback redirect** against
   `linear.app/oauth/authorize` → `api.linear.app/oauth/token`. Ship a hard-coded, non-secret
   `client_id` from a single `distribution=public` app the maintainer creates. Register a small
   fixed set of loopback ports. Store access + refresh token in Keychain by workspace ID.
2. Fallback: `LINEAR_API_KEY` / `lnr auth login --with-token`, documented as the escape hatch for
   workspaces where third-party app approvals block the OAuth app. Recommend a restricted
   (read-only, or read + create-issue) key in the docs; remember the **no-`Bearer`** header form.
3. Do **not** build on Dynamic Client Registration *as a route into
   `api.linear.app/oauth/token`* — the registry split is proven. But before locking (1), spend
   ten minutes testing whether an `mcp.linear.app`-issued token is accepted at
   `api.linear.app/graphql`; if it is, (1) collapses into a no-registration design.
4. There is **no device-code grant**, so there is no copy-a-code fallback for headless/remote
   shells. If `lnr` must work over SSH, the answer is the API key path — call that out in the
   agent contract ticket.

## How to close the last gap (5 minutes, needs a human in a browser)

**Done — see #10.** Recipe kept below for reference (e.g. for #12's cross-workspace re-run).

I deliberately did not create an OAuth app in the user's workspace. To turn 2b from
"documented" into "observed":

1. Create an app at <https://linear.app/settings/api/applications/new> with redirect URI
   `http://localhost:8123/callback` — **this also empirically settles sub-question 1**: if the
   form rejects the loopback URI, the whole design dies here.
2. Open `https://linear.app/oauth/authorize?response_type=code&client_id=<ID>&redirect_uri=http%3A%2F%2Flocalhost%3A8123%2Fcallback&scope=read&state=x&code_challenge=<S256(verifier)>&code_challenge_method=S256`
   with `nc -l 8123` listening; grab the `code`.
3. `curl -X POST https://api.linear.app/oauth/token -d 'grant_type=authorization_code&code=<CODE>&client_id=<ID>&redirect_uri=http://localhost:8123/callback&code_verifier=<VERIFIER>'`
   — **omitting `client_secret` entirely**.
4. `curl -X POST https://api.linear.app/graphql -H 'Authorization: Bearer <TOKEN>' -H 'Content-Type: application/json' -d '{"query":"{ viewer { id name } }"}'`

If step 3 returns a token and step 4 returns a viewer, the design is locked.

Separately, and worth doing **first** because a positive result removes the need for any
registered app: complete a browser authorization against `mcp.linear.app/authorize` using the
already-registered public client `GcwYZSKR3qavd9Tl` (redirect `http://localhost:8123/callback`,
PKCE S256), exchange at `mcp.linear.app/token`, then try that access token at
`api.linear.app/graphql` with `Authorization: Bearer …`. Expected to fail on audience scoping —
but if it succeeds, `lnr` needs no OAuth app at all.

## Housekeeping

The probe registered a throwaway public client `GcwYZSKR3qavd9Tl` against `mcp.linear.app`'s DCR
endpoint. The registration response carried no `registration_access_token`, so it cannot be
deleted; it is an unauthorized, credential-less client id and holds no access to anything.

## What I could not establish

- ~~Whether `linear.app/settings/api/applications/new` actually accepts a `http://localhost:PORT`
  redirect URI~~ **Resolved in #10: yes, accepted unchanged.**
- ~~Whether an `api.linear.app`-issued PKCE token with no client secret really works at
  `/graphql`~~ **Resolved in #10: yes — secretless exchange returns a working token.**
- ~~**Whether an access token issued by `mcp.linear.app/token` to a DCR-registered public client is
  accepted at `api.linear.app/graphql`.**~~ **Resolved in #10: no — rejected with a 401
  `AUTHENTICATION_ERROR`.** The separate-registries claim above was inference from documentation;
  it is now an observed fact.
- Whether a plain Member (non-admin) can create an OAuth application. *(Still open — #10's app was
  created by a workspace admin.)*
- Whether Third-Party App Approvals is on or off by default, and whether it applies to
  `distribution=private` apps. *(Still open — #10 found no such setting on the app's own edit page;
  it may live elsewhere in workspace/org settings and wasn't located.)*
- Refresh token absolute lifetime. *(Still open — #10 observed `expires_in: 86399` on the access
  token, ~24h as expected, but did not test refresh-token expiry.)*
- Whether personal API keys expire. *(Still open — out of scope for #10.)*
- ~~Whether marking an app `distribution=public` requires any Linear-side review.~~ **Resolved in
  #10: no — Availability shows "public available" immediately, no pending-review state.**
- **New, from #10:** whether the same shipped `client_id` authorizes unmodified against a
  *different* Linear workspace — the actual precondition for shipping one non-secret `client_id`
  to every `lnr` user. Open in #12.

## Method note

Per the user's global rule, `ctx7` was the primary documentation source: `npx ctx7@latest library
"Linear" …` resolved `/websites/linear_app_developers` and `/websites/linear_app`, and both were
queried (3 `ctx7` commands total, no quota errors). Findings were then verified directly against
the owning pages on `linear.app/developers` and `linear.app/docs` via WebFetch, plus the live
endpoint probes reproduced above.
