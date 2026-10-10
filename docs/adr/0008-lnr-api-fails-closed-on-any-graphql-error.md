# `lnr api` fails closed on any GraphQL error

Decided while planning the skeleton slice (Linear HF-97). `lnr api '<graphql>'` is the one command
whose success output is Linear's response as-is, but its *failure* behaviour is the same as every
other command's.

A GraphQL response may carry `data` and `errors[]` together. `lnr api` treats any non-empty
`errors[]` as a failure:

- the exit code is mapped from the response's own `extensions.code`, like every other command
  (see [ADR-0003](0003-output-tiers-are-flag-only-and-json-is-not-raw.md), amended);
- stdout is **empty**;
- the full response body, partial `data` included, goes to **stderr**.

A clean response is printed byte-for-byte to stdout with exit `0`.

## Considered and rejected

**Faithful passthrough: print the whole response, exit `0` when `data` is present.** It is the
literal reading of "raw", and it would let a caller keep partial results. It was rejected because it
produces the shape that makes callers write `if [ $? -eq 0 ]` and be wrong: an exit `0` with a body
full of errors. It also breaks the agent contract's one unconditional promise — stdout is empty on
failure — for the single command a caller is most likely to script against.

## Cost accepted

A caller that genuinely wants partial data from a multi-field query cannot get it from stdout. It
can read it from stderr, or split the query. Reversing this later would silently change what
existing scripts see on stdout, which is why it is recorded rather than left implicit.
