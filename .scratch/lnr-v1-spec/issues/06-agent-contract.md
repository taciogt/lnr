# What is the contract between `lnr` and a calling agent?

Type: grilling
Status: open
Blocked by: 03

## Question

The primary consumer is an agent shelling out, not a human. That makes several normally-boring
CLI decisions load-bearing.

To decide:

1. **Exit codes.** A flat 0/1, or distinguished codes (not-found, auth-required, rate-limited,
   validation-failed)? An agent that can branch on the code avoids parsing prose.
2. **stdout vs. stderr.** Does anything but the payload ever touch stdout? Progress, warnings, and
   deprecation notices must not pollute a piped result.
3. **Error shape.** When the API rejects a write, does the agent get a one-line reason or a
   GraphQL error dump? The lean discipline applies to failures too — arguably more, since errors
   are what trigger retries and re-reads.
4. **Non-interactive safety.** Keychain access can prompt, and a prompt hangs a non-interactive
   shell-out indefinitely. What is the behaviour when no TTY is attached and the token is
   unavailable — fail fast with a distinguished exit code, or fall back to the file store? This
   was flagged during charting as an explicit acceptance criterion, not a discovery.
5. **Idempotency and confirmation.** Do destructive or write operations ever prompt? They must
   not, when non-interactive. Is there a `--yes` and is it the default without a TTY?
6. **Rate limiting.** Does `lnr` retry with backoff internally, or surface 429 and let the agent
   decide? Silent retries burn wall-clock time an agent cannot see.

Depends on the lean output tiers being settled first.
