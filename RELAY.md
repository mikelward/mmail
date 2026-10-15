# Own mail server with reverse aliases (exploratory)

Status: exploratory, 2026-10-15. **Not the plan.** The plan is Option 4 and/or the mmail
Outbox ([OPTIONS.md](OPTIONS.md), [DESIGN.md](DESIGN.md)). This writes up the "run our own
MX and rewriter" idea properly: a possible side project, and the only way to get zero-tap
replies. Read it against OPTIONS.md **R7**: nothing invasive on a critical path.

## 1. What it would and wouldn't solve

| Case | Solved? | Why |
|---|---|---|
| **Reply / Reply-all** in the Gmail app or web | ✓ zero extra taps | Inbound mail already shows relay addresses, so Gmail replies to the relay, which sends as `me@example.net`. |
| **Typing a new recipient in To:** | ✗ | Gmail delivers straight from Google to the recipient's MX; nothing of ours is in that path. You'd have to type an encoded address (`bob=example.org@r.example.net`) or pick it from autocomplete, right next to the real address. Pick the real one and it goes out as `me@gmail.com`. |
| **Forwarding** a message to someone new | ✗ | Same as To:. |
| **Accurate Sent folder** | ✗ | Gmail's copy says From `me@gmail.com`, To the relay addresses. |
| **Calendar RSVPs and invites** | ✗ | Sent by Calendar, not by Gmail compose. |

**So a relay is an add-on, never the whole answer.** New mail still needs Option 4 or the
Outbox, which also handle replies with one or two taps. The relay's marginal value is those
taps.

## 2. Topologies

Three ways to build it, compared with today. Each "hop" is a party that has to be up for mail
to flow.

| | Inbound path | Reply path | Parties | We build |
|---|---|---|---|---|
| **Today** | sender → Forward Email → Gmail | Gmail → smtp2go → recipient | FE, smtp2go | nothing |
| **A. Rented plumbing** | sender → Forward Email → *our webhook* → Gmail | Gmail → FE → *our webhook* → smtp2go → recipient | FE, smtp2go, us | rewriter only |
| **B. Own MX, rented smarthost** | sender → *our MX* → Gmail | Gmail → *our MX* → smtp2go → recipient | smtp2go, us | MX + rewriter |
| **C. Own everything** | sender → *our MX* → Gmail | Gmail → *our MX* → recipient's MX | us | MX + rewriter + outbound delivery |

What each topology trades:

- **A** is the least code. But it puts two vendors and us in series on *both* paths, which is
  the worst for "fewer interactions".
- **B** has the same hop count as today, with Forward Email replaced by us. Inbound
  reliability is ours to engineer, which can be done with the fail-open design in §3.
  Outbound reputation stays outsourced, and that part can't be engineered quickly.
- **C** has the fewest parties, but adds the hardest part of running mail: getting *other
  people's* servers (Gmail, Microsoft, Yahoo, corporate gateways) to trust a small new IP.
  See §6.

**If built: B**, with Forward Email kept as a fail-open backup.

## 3. Architecture (topology B)

```
                    MX 10                          rewrite OK
  world ───────────────────► mmail-mx ──────────────────────────► Gmail (API import;
    │                     (Postfix+rspamd                            SMTP fallback)
    │                      +rewriter)       rewrite fails ×N
    │                         │   └──────── deliver unmodified ───► Gmail
    │  MX 20 (only when MX 10 is down)
    └──────────────────► Forward Email ──── plain forward ─────────► Gmail

  Gmail ── reply to …@r.example.net ──► mmail-mx ── verify, rewrite, DKIM ──► smtp2go ──► recipient
           (MX for r.example.net)        (if down: Gmail queues and retries for days)
```

### Rules that keep it inside R7

These are the point of the design.

1. **Never accept-then-drop.** A message is either refused during the SMTP conversation, so
   the sender's server bounces it and the sender knows, or delivered into Gmail, possibly to
   Spam. Nothing is silently discarded after a `250 OK`. That includes losing the machine:
   a `250 OK` is only as good as the disk holding the queue, so the queue must survive the
   host (rule 6).
2. **Fail open on rewriting.** Postfix accepts and queues durably *before* our code runs
   (after-queue content filter). If the rewriter crashes or can't parse a message, it stays
   queued and is retried. After N attempts or 30 minutes it is delivered **unmodified** and
   you get an alert. Mail arrives un-rewritten rather than not at all.
3. **Fail open on outage.** MX 20 is Forward Email doing today's plain forwarding. If
   mmail-mx is down, mail still arrives the way it does now, just not rewritten. Replies to
   those messages go out as `me@gmail.com` unless you use Option 4 or the Outbox.
   - Check whether Forward Email accepts being the lower-priority MX. If it doesn't, the
     backup is a second tiny VM that only forwards.
   - Spammers aim at backup MXes, but Forward Email filters spam itself.
4. **The reply path has a natural buffer.** If mmail-mx is down, Gmail queues replies to
   `r.example.net` and retries for days, and tells you about the delay. They're never lost.
5. **Rewrite only what authenticates.** Inbound mail is rewritten only when the sender passes
   DMARC or aligned DKIM. Everything else is delivered unmodified, with a header saying so.
   No relay address is ever minted for a forged sender.
6. **Accepted mail survives losing the host.** Rebuilding the VM from config doesn't bring
   back messages that were already accepted but not yet delivered. So:
   - **The Postfix queue lives on a synchronously replicated disk** (e.g. a Google Cloud
     regional persistent disk, replicated across two zones). It's reattached to a
     replacement VM, which then finishes delivering whatever was in flight.
   - **For a stronger guarantee,** a before-queue proxy (`smtpd_proxy_filter`) writes each
     raw message to multi-zone object storage *before* Postfix answers `250 OK`. After an
     unrecoverable loss, the journal is replayed, with delivery de-duplicated by Message-ID.
   - **Queues normally drain in seconds,** so the exposed window is small; the canary
     (§4) would notice a stuck queue.

### Components: assemble the boring parts, write the interesting ones

| Component | Use | Notes |
|---|---|---|
| SMTP server, queue, retries, TLS, bounces (DSNs), rate limits | **Postfix** | Don't write an MTA; its queue is what makes "fail open" safe. |
| SPF/DKIM/DMARC/ARC checks, spam scoring, greylisting, DNSBL/RBL, DKIM signing | **rspamd** | Greylisting and RBL modules built in. |
| DNS resolver | **unbound** (local) | Spamhaus refuses DNSBL queries that come through big public resolvers, so run your own resolver or use Spamhaus DQS. |
| Inbound rewriter (ours) | after-queue content filter, in Go or Python | Rewrites the other parties in From/To/Cc/Reply-To into relay addresses, keeps display names, adds `X-Mmail-Original-*`. Mapping in SQLite. |
| Reply verifier and rewriter (ours) | same service | Requires DKIM pass `d=gmail.com` and `From` = `me@gmail.com`. Takes recipients from the signed headers, checks `h=` covers to/cc, enforces a recent `t=`, de-duplicates Message-IDs, and rejects any non-relay recipient. Then rewrites back and hands the message to Postfix to relay via smtp2go. |
| Gmail delivery agent (ours) | a Postfix `pipe` transport calling Gmail API `messages.import` | Applies Gmail's spam classification, with no need for Gmail to trust our IP. A non-zero exit is a temporary failure, so Postfix retries. Reuses the mmail OAuth project and the `gmail.insert` scope. If tokens break for hours, fall back to SMTP forwarding with SRS + ARC. |
| Bounce intake (ours) | smtp2go webhook → small endpoint | Delivery failures on replies become a message in your Gmail ("Undeliverable: …"). |

An all-in-one alternative is Stalwart, which has its own queue, spam filter and DKIM, but
the rewriting still needs custom code. Writing more of the stack yourself (Go:
`emersion/go-smtp`, `go-msgauth`, `go-message`) is more fun, but then *you* own queue
durability, and that is where silent loss comes from.

### Address scheme

- **Relay subdomain:** `r.example.net`, with its own MX pointing at mmail-mx.
- **Readable addresses:** `alice=example.org@r.example.net`, so new contacts can be typed by
  hand.
- **Identity prefix:** `shop+alice=example.org@r.example.net` selects which `example.net`
  identity the reply comes from.
- **Optional:** a short HMAC tag, so stray guesses bounce instead of reaching the verifier.

### DNS and authentication

- **MX:** `example.net` at 10 = mmail-mx, 20 = backup. `r.example.net` = mmail-mx.
- **SPF:** smtp2go only (topology B), then `-all`.
- **DKIM:** smtp2go's keys, plus our own key if mmail-mx ever signs directly. Rotate yearly.
- **DMARC:** `p=reject`, with `rua` aggregate reports parsed (e.g. parsedmarc) for monitoring.
- **TLS:** MTA-STS and TLS-RPT for inbound TLS. DANE only if the zone has DNSSEC.
- **Host:** a PTR and a matching HELO name for mmail-mx; some senders check it.

## 4. Observability

This is what makes it trustworthy rather than scary.

- **End-to-end canary.** This is the one that matters. Every 10 minutes an external sender
  mails a probe address on `example.net`: a scheduled Apps Script in a separate Google
  account works. A checker confirms it arrived in Gmail *rewritten*, within 5 minutes. A
  second canary replies through the relay and confirms delivery. Alert after two misses.
- **Metrics:**
  - Postfix queue size and oldest message age;
  - deferred counts by reason;
  - rejects by reason (DNSBL, DMARC, greylist, spam score);
  - rewriter outcomes (rewritten / unmodified-unauthenticated / fail-open);
  - Gmail import latency and errors;
  - smtp2go delivery and bounce events;
  - certificate expiry;
  - whether our IP appears on blocklists (topology C).
- **Per-message trace.** "What happened to the mail from X around 14:00?" Message-ID and
  queue ID are joined across Postfix, rspamd, rewriter and import logs, behind a small search
  page.
- **Daily digest into Gmail.** Counts, everything *rejected* at SMTP time (the only mail you'd
  never otherwise see), fail-open events, and DMARC report anomalies.
- **Alerts** (Web Push or ntfy):
  - canary miss;
  - queue age over 15 minutes;
  - any fail-open delivery;
  - Gmail import auth failure;
  - DMARC failures for `example.net`.

## 5. Failure modes

| Failure | What happens | Visible to |
|---|---|---|
| mmail-mx down | Senders use MX 20; mail arrives unrewritten. Replies to `r.` addresses queue at Gmail. | canary alert |
| Rewriter bug | Messages wait in the Postfix queue, then are delivered unmodified. | fail-open alert |
| Gmail import auth broken | Postfix queues (days of headroom); fall back to SMTP forwarding after N hours. | alert |
| Spam false positive at SMTP time | Sender gets a bounce; you see it in the digest. Keep thresholds conservative: reject only high-confidence cases (Spamhaus ZEN listings, DMARC `p=reject` failures) and let Gmail classify the rest. | sender + digest |
| Greylisting delays a legit first-time sender | 5–30 minute delay; senders retry automatically. | metrics |
| smtp2go down | Replies wait in the Postfix queue. | alert |
| VM disk or host dies | MX 20 takes over for new mail. The replicated queue disk is reattached to a replacement VM, which delivers already-accepted mail (or the journal is replayed; rule 6). The mapping DB is on the same replicated disk and backed up. | canary alert |
| You pick a real address from autocomplete | Mail goes straight from `me@gmail.com`. The relay never sees it. | nobody (inherent, §1) |

## 6. Topology C extras: delivering outbound mail ourselves

Only if deliverability itself is the fun part.

- **Port 25:** a static IP with outbound port 25. Google Cloud blocks egress on port 25
  entirely, and many VPS providers block it until you ask.
- **DNS for the IP:** a clean IP with PTR/rDNS; IPv6 needs its own PTR, or disable outbound
  IPv6.
- **Sender requirements:** meet Gmail's and Yahoo's (SPF, DKIM, DMARC, PTR, TLS, low
  complaint rates). Register with Microsoft SNDS/JMRP.
- **Warm-up:** expect months of spam-foldering at Microsoft and some corporate gateways for
  a low-volume new IP.
- **Upkeep:** watch blocklists, handle receivers' greylisting and rate limits (Postfix
  retries), and generate your own DSNs.

**Verdict:** keep smtp2go as the smarthost. Personal volume never builds IP reputation; it
just inherits whatever the IP had.

## 7. Effort and running cost

- **Build:** roughly 3–6 weekends to trustworthy, most of it in the rewriter, the
  verifier and observability.
- **Burn-in:** run it for a month on a *secondary* address before moving the primary
  domain's MX.
- **Running:**
  - a small VPS or VM with a static IP: ~$5–10/month (Google Cloud is fine for B, since
    inbound 25 is open and outbound goes via smtp2go on 587);
  - monthly patching, yearly DKIM rotation, glancing at the digest.

## 8. Where this leaves the decision

- It **can't replace** Option 4 or the Outbox, because it does nothing for typed recipients.
- Built as B, with fail-open backup MX, reject-or-deliver and canaries, it can plausibly meet
  R7. But "plausibly" means a month of burn-in on a secondary address before the primary
  domain depends on it.
- It's a good project if the goal includes *running mail* for fun. For saving one or two
  taps on replies alone, it's a lot of machinery.
