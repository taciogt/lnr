# Addressing scheme follows confirmed API capability, not a uniform rule

Decided in [#5](https://github.com/taciogt/lnr/issues/5). It would be simpler for every `lnr` noun
to accept the same kinds of references. It doesn't:

- `issue` accepts an identifier (`ENG-42`), a UUID, or a pasted Linear URL (parsed client-side).
  Linear's own docs confirm `issue(id:)` and `issueUpdate(id:)` coerce both identifier and UUID
  natively — no client-side lookup needed.
- `project`, `milestone`, and `label` accept a name (resolved client-side to a UUID via a
  list-and-filter lookup) or a UUID directly. Live introspection against
  `api.linear.app/graphql` showed `project(id:)`, `projectMilestone(id:)`, and `issueLabel(id:)`
  take a bare `id` argument with no documented shorthand coercion — unlike issue, there is no
  identifier form the API itself understands, so `lnr` has to do the name resolution that Linear
  does for issues.
- `team` accepts its short **key** (e.g. `ENG`), resolved client-side to the UUID `teamId`
  mutations actually require — `issueCreate`'s schema has no `teamKey` shortcut.

This was verified, not assumed: the difference is a fact about what each API argument actually
coerces, confirmed live rather than inferred from the schema's types (which are all just `String`
and would have looked identical either way). A future noun should get the same treatment —
introspect and test what its `id`-shaped arguments actually accept before deciding whether it needs
client-side name resolution, rather than assuming uniformity with `issue` or with any other noun
already shipped.
