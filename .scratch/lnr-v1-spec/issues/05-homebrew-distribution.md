# How does `lnr` reach a Mac?

Type: research
Status: claimed (research subagent, in progress)

## Question

The user wants easy Mac distribution, Homebrew named as the preferred channel. The repo will be
public on the personal GitHub account.

Sub-questions:

1. `goreleaser` + a personal tap (`taciogt/homebrew-tap`) — what is the current minimal setup for
   a single-binary Go project, and what does the release workflow look like end to end?
2. Personal tap vs. submitting to `homebrew-core`. What are homebrew-core's acceptance criteria
   (notability, stability, no HEAD-only) and is a brand-new tool plausibly eligible? A tap is the
   safe answer; confirm what it costs the user in install ergonomics (`brew install taciogt/tap/lnr`).
3. Does the formula need to be signed or notarized for macOS Gatekeeper, given a binary
   downloaded from a GitHub release? What is the actual user experience on first run?
4. Cross-compilation and Apple Silicon vs. Intel: what does goreleaser produce, and does the
   formula need both?
5. What does the GitHub Actions release pipeline need in terms of secrets and permissions?

Use `ctx7` for goreleaser and Homebrew documentation per the user's global rule.
