# Hand-written GraphQL client, or generated from introspection?

Type: research
Status: claimed (research subagent, in progress)

## Question

The user chose Go and said: "If Linear provides a good API documentation, I can write the API
client from scratch without too much effort." That premise needs testing before it becomes a
design decision.

Linear's API is GraphQL, which means the schema may be introspectable — a typed client could be
generated (`genqlient`, `gqlgen`) instead of hand-written.

Sub-questions:

1. Is introspection enabled on `api.linear.app/graphql` for an authenticated client?
2. How large is the schema? A schema that generates 50k lines of Go is a maintenance liability
   for a tool that needs ~8 operations.
3. What is the actual quality and completeness of Linear's API documentation for the operations
   in scope (issue create/update/get/list, comment create, issue statuses, teams, projects,
   labels)?
4. Are there Go client libraries already, and are they maintained?
5. What does pagination look like (Relay-style cursors?), and error shape on failure?

The trade-off to report on: generated clients resist API drift but bloat the repo and complicate
the build; hand-written clients stay small and readable but silently rot. Recommend one, with the
evidence.

Use `ctx7` per the user's global rule.
