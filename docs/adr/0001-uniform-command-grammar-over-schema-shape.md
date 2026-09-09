# Uniform command grammar overrides Linear's own schema shape

`lnr`'s noun-verb grammar deliberately diverges from Linear's API in two places where following
the schema literally would have produced an inconsistent CLI, decided in
[#5](https://github.com/taciogt/lnr/issues/5):

- **Create and update are always separate verbs.** Linear's own MCP tools (`save_issue`,
  `save_project`, `save_milestone`, `save_comment`) are all upsert-shaped — one verb doing both.
  `lnr` splits every one of them into `create`/`update` instead, so a coding agent declares intent
  rather than the CLI inferring it from whether an ID was passed.
- **`milestone` is a top-level noun, not `project milestone`.** Every `ProjectMilestone` requires a
  parent `projectId` in Linear's schema — nesting would have mirrored that literally. `lnr` keeps
  it flat (`lnr milestone create --project ...`) to avoid three-level nesting for the sake of one
  entity, at the cost of the parent relationship being a flag instead of visible in the command
  path.

The rule going forward: a new noun gets `lnr`'s standard verb set and grammar position by default;
deviating from Linear's schema shape is expected, not something each new ticket needs to
re-litigate.
