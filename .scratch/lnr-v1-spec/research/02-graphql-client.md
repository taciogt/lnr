# Research: hand-written GraphQL client, or generated from introspection?

Resolves `.scratch/lnr-v1-spec/issues/02-graphql-client-strategy.md`.
Date: 2026-09-08. All measurements reproducible; probe commands recorded inline.

## RECOMMENDATION

**Hand-written.** Not because generated code is big — that premise is refuted below — but because
`lnr` already ships a raw `lnr api` GraphQL passthrough, which forces an untyped
query-string-in / JSON-out request path to exist regardless. A generated client would be a
*second* request path and a *second* error-handling path beside it, plus a codegen step and a
vendored 43k-line schema that serve only one of the two. Hand-written types ride the single path
the passthrough already requires.

The user's premise — "if Linear provides good API documentation, I can write the client from
scratch without too much effort" — was tested and **largely holds**. See §3.

---

## 1. Is introspection enabled? — YES, and without authentication

Introspection is enabled on the production endpoint for **unauthenticated** callers.

```
$ curl -s -X POST https://api.linear.app/graphql \
    -H "Content-Type: application/json" \
    -d '{"query":"query{__schema{queryType{name}}}"}'
{"data":{"__schema":{"queryType":{"name":"Query"}}}}   # HTTP 200
```

A full introspection query (all types, fields, args, enum values, descriptions) also returns
unauthenticated: **HTTP 200, 2,687,215 bytes**. This is notable — `{viewer{id}}` on the same
endpoint returns HTTP 401 (§5), so introspection is deliberately public.

Linear also publishes the SDL directly, so introspection is not even required:
<https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql>
(confirmed by <https://linear.app/developers/sdk>: "The Linear Typescript SDK exposes the Linear
GraphQL schema").

## 2. How large is the schema?

Measured from the live introspection response (`schema.json`, excluding `__`-prefixed
introspection meta-types):

| Metric | Value |
|---|---|
| Named types | **1,181** |
| — OBJECT / INPUT_OBJECT / ENUM | 630 / 399 / 124 |
| — INTERFACE / UNION / SCALAR | 8 / 7 / 13 |
| Object + interface fields | **5,224** |
| Input-object fields | **3,144** |
| Enum values | 734 |
| `Query` root fields | **170** |
| `Mutation` root fields | **375** |
| `*Connection` types | 66 |

As SDL: **43,283 lines / 1.20 MB** (via `npx get-graphql-schema https://api.linear.app/graphql`).
Linear's own published `schema.graphql` is **51,944 lines / 1.25 MB** with **1,264** type
definitions vs. 1,176 in the introspected SDL. The line delta is mostly docstring formatting
(23,430 `"""` lines in the official file vs. 14,260 in the introspected one); the 88 extra types
suggest the published file tracks `master` and is slightly ahead of, or divergent from,
production. **Quote the live introspection numbers**; treat the published file as convenient, not
authoritative.

### The "50k lines of Go" fear — measured, and it does not survive

The ticket's worry is that codegen produces an unmaintainable blob. I built it and counted.

**Probe: `genqlient` v0.8.1, Go 1.27.1, 9 real operations** (issue get/list, issue create/update,
comment create, teams, workflow states, projects, labels — the milestone-1 surface):

| Variant | Generated LOC | Compiles? |
|---|---|---|
| 9 ops, `issues(filter:)` omitted | **1,786** | yes |
| 9 ops, `$filter: IssueFilter` as a typed argument | **8,578** | only with `optional: pointer` (§2.2) |
| `gqlgenc`, *full* API coverage (chainguard's `go-linear`) | **30,613** (`models.go` alone, 1.42 MB) | yes (published) |

Two conclusions:

1. **genqlient generates only what your operations reach**, not the whole schema. The 30,613-line
   figure is `gqlgenc` at full coverage — a real liability, but *not the option on the table*.
   For 8 operations the honest number is **1,786 lines**.
2. **A single argument dominates the cost.** Adding `$filter: IssueFilter` took the output from
   1,786 → 8,578 lines (+6,792, a 4.8x blow-up) because `IssueFilter` has **70 input fields**
   that transitively pull in 60 `*Filter` types. `IssueCreateInput` has 36 fields,
   `IssueUpdateInput` 34.

**Therefore LOC does not discriminate between the options.** A generated client for this surface
is ~1.8k lines — perfectly maintainable. The recommendation must rest on other grounds, and does
(§0, §6).

### 2.2 Measured genqlient friction

Four failed runs before a clean build. Reproducible costs:

- **Vendoring a 43k-line `schema.graphql`** into the repo, plus a refresh script.
- **Bindings for 8 custom scalars** must be declared or generation aborts, even for scalars your
  operations never touch: `DateTime`, `TimelessDate`, `JSONObject`, `JSON`, `UUID`, `Duration`,
  `DateTimeOrDuration`, `TimelessDateOrDuration`. First failure:
  `linear.graphql:7690: unknown scalar DateTimeOrDuration: please add it to "bindings"`.
- **Default output does not compile.** Linear's `*Filter` inputs are mutually recursive by value,
  which Go rejects:

  ```
  ./generated.go:7534:6: invalid recursive type TeamCollectionFilter
      ./generated.go:7534:6: TeamCollectionFilter refers to TeamFilter
      ./generated.go:7588:6: TeamFilter refers to TeamCollectionFilter
  ./generated.go:1889:6: invalid recursive type IssueCollectionFilter
      ./generated.go:1889:6: IssueCollectionFilter refers to NullableUserFilter
  ./generated.go:5532:6: NullableUserFilter refers to IssueCollectionFilter
  ```

  Fixed by one line — `optional: pointer` in `genqlient.yaml` — but the fix is not discoverable
  from the error text.
- Toolchain skew: `go run github.com/Khan/genqlient@latest` fails outright on Go 1.27.1
  (`golang.org/x/tools@v0.24.0`: `invalid array length -delta * delta`). Requires pinning
  `x/tools` ≥ v0.50.0 in a tools module.

None of these are fatal. All are recurring build-system surface area for a project whose stated
primary design constraint is leanness.

## 3. Documentation quality — GOOD; the user's premise holds

Testing, not confirming, the premise — the evidence came back mostly confirming. Stated as such.

- **94.0% of schema fields carry descriptions** (7,902 of 8,406 object + input fields, measured
  from introspection). The schema is self-documenting; SDL is effectively the reference manual.
- Every in-scope operation exists and is reachable, verified against introspection:
  `Query.issue`, `Query.issues`, `Query.team`, `Query.teams`, `Query.projects`,
  `Query.issueLabels`, `Query.workflowStates`, `Query.searchIssues`, `Query.issueSearch`;
  `Mutation.issueCreate`, `Mutation.issueUpdate`, `Mutation.commentCreate`.
- Hand-written worked examples for **most** of these operations are published at
  <https://linear.app/developers/graphql> — `viewer`, `teams`, `team`, `issue`, `issueCreate`,
  `issueUpdate`, `workflowStates` appear as copy-pasteable queries. Note the prose docs cover a
  *subset*: `commentCreate`, `projects` and `issueLabels` are in scope and have no worked example
  on that page. They were confirmed to exist via introspection, not via prose. The 94% schema
  descriptions cover all of them, which is the load-bearing point.
- A full public schema reference is browsable at
  <https://studio.apollographql.com/public/Linear-API/variant/current/schema>.
- Dedicated pages exist for the two things that usually go undocumented:
  <https://linear.app/developers/pagination> and <https://linear.app/developers/rate-limiting>.

**Payload shape is uniform and trivial to hand-model.** `IssuePayload` is
`{lastSyncId, issue, success}`; `CommentPayload` is `{lastSyncId, comment, success}`. Every
mutation returns the same three-field shape.

## 4. Existing Go client libraries — one real candidate, low adoption

**There is no official Go SDK.** <https://linear.app/developers/sdk> documents only the TypeScript
SDK ("written in Typescript but can also be used in any Javascript environment").

Best third-party candidate — **`chainguard-sandbox/go-linear`** (<https://github.com/chainguard-sandbox/go-linear>):

- Apache-2.0, created 2025-12-10, last push 2026-09-08 (**actively maintained**), **8 stars**.
- Go SDK + CLI + MCP server; v1.0.0 ships "45+ methods"; generated with `gqlgenc`/`genqlient`
  (`make sync-upstream` refetches schema and regenerates), vendored `schema.graphql` = 1.25 MB.
- 3.6 MB of Go across 497 files; `internal/graphql/models.go` alone is 30,613 lines.

GitHub search for Go + Linear + GraphQL returns nothing else with traction: top hits are
`turbot/steampipe-plugin-linear` (4 stars) and several `linear-cli` projects at 0–2 stars.

**Assessment:** `go-linear` is a *competitor* to `lnr`, not a dependency for it — it bundles its
own CLI and MCP server, and it lives in a vendor's `-sandbox` org with 8 stars. Depending on it
would import 3.6 MB and a full second CLI to make 8 calls. Not viable.

## 5. Pagination and error shapes — both simple, both hand-writable

### Pagination: Relay-style, with an escape hatch

Confirmed from introspection:

- `PageInfo` = `{hasPreviousPage, hasNextPage, startCursor, endCursor}`
- `IssueConnection` = `{edges, nodes, pageInfo}`; `IssueEdge` = `{node, cursor}`
- 66 `*Connection` types share this shape.

**The `nodes` shortcut matters.** Linear exposes `nodes` alongside `edges`, so a client can skip
`edges`/`cursor` entirely and page with `first` / `after` + `pageInfo.endCursor`. One generic
`{ nodes: [...], pageInfo: {...} }` Go struct per resource covers every list command. Documented
at <https://linear.app/developers/pagination>.

### Errors: real HTTP status codes plus structured `extensions`

Unlike many GraphQL APIs, Linear sets meaningful HTTP statuses rather than always returning 200.
Probed live:

```jsonc
// {viewer{id}} with no token  → HTTP 401
{"errors":[{"message":"Authentication required, not authenticated",
  "extensions":{"type":"authentication error","code":"AUTHENTICATION_ERROR",
    "statusCode":401,"userError":true,
    "userPresentableMessage":"You need to authenticate to access this operation.",
    "meta":{},"http":{"status":401}}}]}

// {issue(id:"X"){nosuchfield}}  → HTTP 400
{"errors":[{"message":"Cannot query field \"nosuchfield\" on type \"Issue\".",
  "locations":[{"line":1,"column":16}],
  "extensions":{"http":{"status":400,"headers":{}},
    "code":"GRAPHQL_VALIDATION_FAILED","type":"graphql error","userError":true}}]}
```

An invalid token (`Authorization: lin_api_totallyinvalid`) returns the identical 401 body as no
token at all — so `lnr` cannot distinguish "no credential" from "bad credential" by response
alone. Rate limiting surfaces as `extensions.code == "RATELIMITED"`
(<https://linear.app/developers/rate-limiting>).

`extensions.code` + `userPresentableMessage` is precisely what a lean CLI wants for error
reporting, and it is ~20 lines of Go to model.

## 6. Trade-off and decision

The ticket frames it as: generated clients resist API drift but bloat the repo; hand-written
clients stay small but silently rot.

**The bloat half of that frame is refuted** (§2): genqlient for this surface is 1,786 lines.
**The drift half is real but small.** `lnr` touches ~8 operations over `Issue`, `Team`,
`Project`, `IssueLabel`, `WorkflowState`, `Comment` — Linear's most stable core entities. Drift
here would be a breaking change to Linear's own primary API surface.

What actually decides it is the argument the LOC numbers can't reach:

> `.scratch/lnr-v1-spec/map.md` (Out of scope): "Reimplementing the 53 unused MCP tools as
> commands. The raw `lnr api` GraphQL passthrough covers them."
> `.../issues/04-milestone-1-command-surface.md`: "roughly 8 commands plus a raw `lnr api`
> GraphQL passthrough".

The passthrough is settled scope. It requires an untyped `POST` + raw-JSON + `errors[]`-decoding
path. Hand-written typed operations are an **estimated** ~200–400 lines of structs layered on
*that same path* (estimate, not a measurement — unlike the 1,786 figure it sits next to).
Generated code would add a parallel path with its own request builder and error handling, a
vendored 43k-line schema, a scalar-binding table, an `optional: pointer` workaround, and a
codegen step in CI — all to serve 8 of the two mechanisms' operations while the passthrough
handles the other 53 untyped anyway.

**Mitigating the rot risk without codegen — probed, not asserted:** vendor `schema.graphql` as a
*test* fixture only and add a CI check that validates the hand-written operation strings against
it. I built this: **28 lines of Go** using `github.com/vektah/gqlparser/v2` v2.5.37
(`gqlparser.MustLoadSchema` + `gqlparser.LoadQuery`).

```
$ go run . operations.graphql
VALID: 9 operations check out against the schema

$ go run . drifted.graphql          # one field renamed to simulate API drift
INVALID: input:3:41: Cannot query field "renamedField" on type "Issue".
exit status 1
```

It passes on the real 9 operations and **fails with exit code 1 and a precise message** when a
field drifts. That buys the resistance-to-drift benefit of codegen at 28 lines, zero generated
code, and no second request path — which is the crux of why the trade-off resolves toward
hand-written.

## 7. Unknowns

Stated explicitly per instructions; none could be established without a workspace token, which
this session did not have (ticket 01, OAuth, is still open).

- **Does `issue(id:)` accept the human identifier (`BLA-123`) or only a UUID?** The schema types
  the argument `String!`, and Linear's own published example passes `"BLA-123"`
  (<https://linear.app/developers/graphql>) — but this is undocumented coercion behavior and
  untested here. **Unknown.** This is exactly the field-level semantics a hand-written client must
  pin down empirically; codegen would not have answered it either (the generated type is
  `string` in both readings).
- **Whether `IssueCreateInput.teamId` is hard-required at runtime.** All 36 input fields are
  nullable in the schema, so the requirement is enforced server-side. **Unknown.**
- **What `lastSyncId` on `IssuePayload`/`CommentPayload` is for**, and whether `lnr` should ever
  surface it. Undocumented in the pages reviewed. **Unknown.**
- **Actual rate limits** (numbers per hour/complexity) were not read; only the error code shape
  was confirmed. See map.md's open "Rate limiting and retry" item.

## Sources

- Live probes against `https://api.linear.app/graphql` (introspection, error shapes), 2026-09-08.
- <https://linear.app/developers/graphql> — operations, auth headers, endpoint.
- <https://linear.app/developers/pagination> — Relay cursor pagination.
- <https://linear.app/developers/rate-limiting> — `RATELIMITED` error shape.
- <https://linear.app/developers/sdk> — official SDK languages; published schema location.
- <https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql> — published SDL.
- <https://studio.apollographql.com/public/Linear-API/variant/current/schema> — public reference.
- <https://github.com/chainguard-sandbox/go-linear> — repo, README, CHANGELOG, GitHub API metadata.
- <https://github.com/Khan/genqlient> v0.8.1 — codegen probe, Go 1.27.1 darwin/arm64.
- <https://github.com/vektah/gqlparser> v2.5.37 — schema-validation probe (§6).
- Docs retrieved via `ctx7` per the user's global rule (`/websites/linear_app_developers`,
  `/chainguard-sandbox/go-linear`); no quota errors encountered.
