# mmail

Keep consumer Gmail as the mail store and the Gmail apps as the UI, but send custom-domain
mail through an external provider after Gmail ends third-party "Send as" in January 2027.

- [OPTIONS.md](OPTIONS.md): what's changing, every option considered (including just sending
  from @gmail.com, the rejected Workspace and relay approaches, the Gmail app's IMAP
  accounts, other clients and migrating), alternative workflows, a comparison, and the
  experiments to run first.
- [DESIGN.md](DESIGN.md): the design for building our own "mmail Outbox" (compose in Gmail,
  dispatch via smtp2go/Forward Email), its web-only mode, Sent-copy handling, and an
  inventory tool.
- [RELAY.md](RELAY.md): exploratory side project. Run our own MX and reverse-alias
  rewriter for zero-tap replies, designed to fail open, with anti-spam, bounces and
  observability. It helps replies only, so it's not the plan.

Status: design only; no code yet.
