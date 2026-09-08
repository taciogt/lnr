# Can a Homebrew-shipped `lnr` do a `gh auth login` browser flow?

Type: research
Status: open

## Question

The user wants terminal-initiated browser auth, not copy-paste of an API key. A binary
distributed through Homebrew is a **public client**: it cannot hide a `client_secret`.

Established while charting (do not re-derive):
- `https://mcp.linear.app/.well-known/oauth-authorization-server` advertises PKCE (`S256`),
  `token_endpoint_auth_methods_supported` including `"none"`, a `registration_endpoint`
  (Dynamic Client Registration), and grants `authorization_code` + `refresh_token`.
  **No device-code grant** — so any browser flow is loopback-redirect, not type-a-code.
- `api.linear.app` and `linear.app` both **404** on the same discovery paths. The DCR and
  public-client findings are scoped to the MCP front door, not the raw GraphQL API.

Sub-questions, in the order they fork the design:

1. Does a Linear OAuth application accept a `http://localhost:PORT` (or `127.0.0.1`) loopback
   redirect URI? This single fact enables or kills the `gh auth login` experience.
2. Can a public client with no `client_secret` obtain a token valid against
   `api.linear.app/graphql` — or is `"none"` auth scoped only to `mcp.linear.app`?
3. Is creating a Linear OAuth application gated on workspace admin? The user *has* admin but
   explicitly prefers a design that does not require it, since the repo is public and other
   users may not.
4. Can personal API keys be scoped read-only? This is the fallback path and it determines the
   safety story.
5. What are the token lifetimes and is refresh available? Determines whether the agent hits
   re-auth mid-session.

Use `ctx7` for Linear's API documentation per the user's global rule. Prefer primary sources.
