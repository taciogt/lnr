# Output tiers are chosen by flag only, and `--json` is not raw API passthrough

Decided in [#4](https://github.com/taciogt/lnr/issues/4). Two temptations were considered and
rejected, both in service of the same promise: output shape never depends on context the caller
isn't controlling.

- **TTY-sniffing.** Many CLIs change shape depending on whether stdout is a terminal (color,
  pagination, column width). `lnr` doesn't: lean, `--verbose`, and `--json` are selected by flag
  alone. An agent's non-interactive output must not silently change shape depending on how it
  happens to be invoked.
- **`--json` meaning "the real API response."** The obvious reading of `--json` is full fidelity —
  everything Linear's API actually returned. `lnr` rejects that: `--json` renders exactly the lean
  tier's fields, structured for scripts. A prototype against real captured payloads made the cost
  of the alternative concrete — raw passthrough for `save_issue` ran ~615 tokens against lean's
  ~23, a 20x regression that reopens the exact 806-token echo-back cost the project exists to cut.
  Full fidelity, when it's genuinely wanted, is `--verbose --json` together, not `--json` alone.

A new command's `--json` output gets the same treatment by default: lean's fields as JSON, not the
underlying GraphQL response shape.
