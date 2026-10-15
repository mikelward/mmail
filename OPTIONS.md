# Options: life after Gmail "Send as" for third-party addresses

Status: draft, 2026-10-15. Companion to [DESIGN.md](DESIGN.md), which covers the
build-it-ourselves path in detail.

## TL;DR

- **Deadline:** Gmail removes "Send as" for non-Google addresses in **January 2027**, in the
  web UI *and* the mobile apps. During Q3–Q4 2026 it "may restrict new configurations".
  So **don't delete, re-add or re-verify** the existing Send-as entries, and don't rotate
  the SMTP credentials behind them. Gmail's own help says a re-verify means delete and
  re-add, and re-adding may already be blocked.
- **Not changing:** inbound forwarding into Gmail, IMAP/POP/Gmail API access, "Send as" for
  Google-hosted identities (other Gmail addresses, Google Workspace addresses), and
  third-party IMAP/SMTP accounts inside the Gmail Android/iOS apps.
- **Ruled out: Google Workspace for the domain.** Gmail still allows Send-as for Workspace
  addresses, but a Google account for `me@example.net` would take over that address across
  Google. Calendar invites and Drive shares sent to it would go there instead of to your main
  account. People would share with both, and you'd end up with two half-accounts (R6,
  Option 1).
- **Recommended: build a small personal service, "mmail Outbox".**
  You keep composing in the Gmail apps, including Gemini, but instead of pressing Gmail's
  *Send* you leave the draft and tap a notification such as "Send as me@example.net". The
  service sends the draft through smtp2go or forwardemail and files a Sent copy back into the
  Gmail thread. Gmail stays the store, the Gmail apps stay the UI, and work and home share
  one interface.
- **Ruled out: reverse-alias relays** (addy.io, SimpleLogin, or our own). They'd give
  zero-tap replies, but only by putting a rewriter in the path of every incoming email, so
  any bug or outage means mail you never see (R7, Option 3).
- **Test first, 15 minutes:** the Gmail Android app can still add non-Google IMAP/SMTP
  accounts. If one that points back at your own Gmail can send via smtp2go (Option 4, E2),
  Android needs no code at all, and the Outbox shrinks to a web-only helper.
- **Free fallback:** send from @gmail.com, optionally with Reply-To set to the domain
  address, plus a small exception path for the services that require the registered address.
  Option 0 below covers what that looks like and what it costs you.
- **Not recommended:** building a full mail client; migrating to Fastmail or self-hosting.
  Work is Gmail anyway, so these make the "one interface" goal worse, not better.

Placeholders used throughout: `me@gmail.com` is the primary Gmail account; `me@example.net`
is the custom-domain address (and `*@example.net` its other aliases).

## Contents

1. [What's changing (facts)](#1-whats-changing)
2. [Requirements and constraints](#2-requirements-and-constraints)
3. [The options](#3-the-options)
4. [Alternative workflows](#4-alternative-workflows)
5. [Comparison](#5-comparison)
6. [Recommendation and decision tree](#6-recommendation-and-decision-tree)
7. [Experiments to run before January 2027](#7-experiments-to-run-before-january-2027)
8. [Open questions](#8-open-questions)
9. [Sources](#9-sources)

Confidence tags: **(doc)** stated in Google or vendor documentation; **(reported)** from
third parties or user reports; **(untested)** our inference, needs an experiment.

## 1. What's changing

From Google's help article "Learn about changes to third-party email account support in
Gmail" ([17101213][g-17101213]) (doc):

| When | What |
|---|---|
| Q3 2026 | Announcement; in-product notices to affected users. |
| Q3–Q4 2026 | Transition. Existing Send-as, Gmailify and POP keep working, but "Gmail may restrict new configurations". |
| January 2027 | "Send as" for third-party accounts, Gmailify and POP fetch removed. |

- **Removed:** "You will no longer be able to send mail from third-party email accounts in
  Gmail on the web or in the Gmail mobile apps."
- **Kept, Send as:** "You can continue to use 'Send as' for any Google-hosted identities,
  including other Gmail addresses or aliases associated with your Google Workspace account."
- **Kept, mobile apps:** "Access through the Gmail mobile app for Android, iPhone, and iPad
  will not change. You can continue to add and use non-Google accounts in the Gmail mobile
  app."
- **Kept, other access:** IMAP, POP and the Gmail API "remain fully supported."
- **Kept, forwarding:** forwarding from another provider into Gmail keeps working.
- **Google's suggestions:** use the provider's own app, a desktop IMAP/SMTP client, the Gmail
  mobile app, or Google Workspace for a custom domain.
- **Google's warning:** it cautions against third-party tools or extensions that "replicate
  Send as", because they need your credentials or OAuth tokens. Anything we build holds our
  own token in our own project; see DESIGN.md §8.

The general Send-as help page ([22370][g-22370]) now carries the same banner. It also notes
that a bounced Send-as address "must be deleted and added again" to re-verify, which is
another reason not to touch the existing setup until the cutover.

Not covered by the article: whether the *Reply-To* setting on the primary address survives
(§8). It probably does: it is a property of `me@gmail.com`, not a third-party Send-as.

Context: Forward Email has already changed its "Send Mail As using Gmail" FAQ to say the
setup is deprecated and to point users at its webmail and apps ([FE FAQ][fe-faq])
(doc). Hacker News threads show the same confusion about whether custom domains count as
"third-party". By Google's wording, a domain not hosted on Workspace does count.

## 2. Requirements and constraints

Must:

- **R1** Consumer Gmail (`me@gmail.com`) stays the mail store and primary Google account.
  Calendar, Gemini, Photos etc. stay put.
- **R2** Recipients see mail from `me@example.net`, DKIM-signed and DMARC-aligned for
  `example.net`, sent through an external provider (smtp2go, forwardemail).
- **R3** Works well on Android, the primary device, where the Gmail app is the heavy-use client.
- **R4** Works acceptably on desktop web (moderate use).
- **R5** One interface for home and work. Work is Google Workspace Gmail, and work policy
  usually rules out third-party clients on work mail. **In practice the shared interface has
  to be the Gmail apps themselves.**
- **R6** One Google identity. Calendar invites and Drive/Docs shares addressed to
  `me@example.net` must keep reaching the consumer account (e.g. through its alternate
  email). No other Google account, Workspace tenant or Cloud Identity tenant may own the
  address or the domain. Otherwise people share with two "you"s and the Google identity
  splits.
- **R7** Nothing invasive on a critical path.
  - Inbound mail never passes through anything we build or a third party rewrites: Forward
    Email forwards straight into Gmail, as today.
  - On the outbound side, a failure must be visible and safe: the draft stays in Gmail, you
    get told, and Gmail's own Send still works.
  - Background tidying of your own sent copies, like the optional Sent fixer (DESIGN.md
    §4.7), is allowed only if it is opt-in, narrowly matched, off the critical path and
    recoverable: Trash, never hard delete.

Should:

- **S1** Replies stay in the right Gmail thread, and sent copies are visible in Gmail (Sent,
  search, Gemini).
- **S2** Picks the right domain identity automatically when replying to mail sent to a domain
  address.
- **S3** Low ongoing maintenance and cost; no single point of failure that silently drops mail.
- **S4** No new party gets broad access to the mailbox.

Nice to have:

- **N1** Zero extra taps compared with today.
- **N2** Works for brand-new recipients, not just replies.

## 3. The options

Options are grouped by how much they change. Each says what it is, how it feels day to day,
pros, cons, cost and confidence.

### Option 0 — Just send from @gmail.com

**What it looks like**

- **Inbound:** unchanged. Forward Email keeps forwarding `*@example.net` into Gmail
  indefinitely.
- **Outbound:** after January 2027 every message, reply and forward, on Android and web, goes
  out `From: me@gmail.com`. Gmail simply stops offering `me@example.net` in the From picker.
- **Optional Reply-To:** set Reply-To on the Gmail address to `me@example.net`: Settings →
  Accounts → Send mail as → *Edit info* on the `me@gmail.com` row → "Specify a different
  'reply to' address" ([22370][g-22370]) (doc for the setting; survival untested).
  Replies then come back via the domain and keep the domain as the conversational address.
- **Display name:** stays the same. Gmail and Workspace recipients will additionally see your
  Google profile photo, which helps them recognise you.
- **Exceptions:** the few cases that *must* come from the domain use an exception path (see
  Option 0b).

**Downsides**, starting with the two you raised:

1. **Systems that match on the sender's address.** These include:
   - helpdesks (Zendesk, Freshdesk, Help Scout, ServiceNow, Jira Service Management), which
     file mail from an unknown sender as a new requester or reject it;
   - reply-by-email on GitHub, GitLab, Discourse and Jira, which only accept verified account
     addresses;
   - mailing lists (Mailman, Google Groups, groups.io), which moderate or reject
     non-subscriber posts;
   - "approved sender" ingest addresses (Send-to-Kindle, TripIt, Expensify receipts and
     similar);
   - banks, schools and government portals that whitelist one address.

   *Mitigation:* add `me@gmail.com` as a secondary or verified address where the service
   allows it (GitHub, GitLab, Kindle, TripIt and Expensify do); subscribe `me@gmail.com` to
   lists with delivery off; ask helpdesks to merge identities; route the rest through the
   exception path. The inventory script (§7, E0) tells you how long this list really is.
2. **Confusion and identity dilution.** Recipients' autocomplete and contacts learn
   `me@gmail.com`; reply-alls spread it; over a few years the domain stops being your de facto
   address. That erodes the main reason to own a domain, which is portability. Reply-To only
   partly counters this, because most clients save the From address, not the Reply-To.
3. **Recipients' filters keyed to the domain address.** VIP lists, spam allowlists and
   corporate gateway rules may misfile mail from the new address.
4. **Privacy.** It exposes the Gmail address, which is also your Google account
   login and recovery target.
5. **Lock-in.** Leaving Gmail later becomes harder, because correspondents know the Gmail
   address.

**Pros:** costs nothing, needs no effort and has nothing to break. Every Gmail feature works
everywhere, and Gmail's own DKIM/DMARC is perfectly aligned.

**Verdict:** a fine *fallback* and a reasonable default for casual mail, but it trades away
long-term identity. Better as part of a hybrid (Option 0b, workflow F in §4) than on its own.

### Option 0b — Gmail by default, domain on demand

Hybrid of Option 0 and any domain-capable path below. Use `me@gmail.com` for casual and
new mail. Use `me@example.net` only where it matters:

- replies to people and systems that know only the domain address;
- sender-matched systems;
- formal correspondence.

The domain sends go through whichever exception path is cheapest: Forward Email webmail,
FairEmail, Thunderbird, or the mmail Outbox. This limits dilution to new and casual contacts
while keeping the important relationships on the domain.

### Option 1 — Make the domain a Google-hosted identity (Workspace): rejected

Google keeps "Send as" for Google-hosted identities, so the obvious moves were:

- **1a.** One Workspace Business Starter seat (~$84/yr) for `me@example.net`, added as a
  Send-as address in consumer Gmail over `smtp.gmail.com`. Whether consumer → Workspace
  Send-as is meant to survive is unconfirmed either way.
- **1b.** Move home mail into that Workspace account, which is Google's own suggestion for
  custom domains.

**Both violate R6.** A Google account that owns `me@example.net` takes over that address
across Google, even if it is only used for sending:

- Calendar invitations and Drive/Docs shares addressed to `me@example.net` resolve to the
  Workspace account, not `me@gmail.com`.
- Collaborators see two "you"s and share with whichever they picked, so over time you have
  two half-accounts.
- 1b additionally moves the store (R1), and Photos, Play and Google One can't follow.

The same applies to anything else that registers the domain with Google as an organisation
(Cloud Identity Free, Workspace Essentials). **Don't verify `example.net` with Google
Workspace or Cloud Identity at all**, not even for a trial.

**The single-identity variants don't rescue it either.** You could make Workspace your *only*
Google account, either by migrating or by converting `me@gmail.com` in place through domain
verification. But Workspace accounts can't create, join or manage a family group or
supervise children with Family Link. They also can't have Google One/AI Pro, YouTube Premium,
Play Pass, Fitbit linking or Android backup, and Gemini is limited and loses Personal
Intelligence. Full pros and cons: [WORKSPACE.md](WORKSPACE.md).

### Option 2 — mmail Outbox: compose in Gmail, dispatch externally (build)

The build-our-own option; full design in [DESIGN.md](DESIGN.md).

- **The loop:** you write (or reply) in the Gmail app as usual, Gemini included, but leave
  the message as a draft instead of pressing Send. A small personal backend watches drafts
  through the Gmail API with push notifications. It offers "Send as me@example.net" via an
  Android notification action, or a button injected into Gmail web by our own extension.
- **What the backend does:** sends the draft's MIME through smtp2go or forwardemail with
  `From: me@example.net`, inserts the sent copy into the original Gmail thread, and deletes
  the draft.
- **Day to day, Android:** reply in Gmail, Bcc `send@example.net` to mark the draft finished,
  then back (the draft is saved) and tap **Send** on the notification. Without that marker,
  the notification only offers **Review**, which opens the draft before sending. That's two
  taps, so a paused, half-written draft can't go out by accident. There's a short undo
  window either way.
- **Day to day, web:** a "Send as ▾" button next to Gmail's Send, plus a guard that catches
  Gmail's own Send on domain threads.
- **Pros:**
  - Gmail stays the store *and* the UI for home and work.
  - Inbound mail is untouched, so Gmail spam filtering, avatars and unsubscribe all work.
  - Works for new recipients.
  - The domain identity is picked automatically from the thread.
  - Needs only narrow Gmail scopes (drafts, metadata, insert).
- **Cons:**
  - One extra tap.
  - Hitting Gmail's own Send by habit sends from `me@gmail.com`. The web guard prevents
    this; Android can only warn after the fact.
  - You run a small service: Cloud Run plus Pub/Sub, effectively free.
  - Needs a personal OAuth client with restricted scopes, published "In production" but
    unverified, since "Testing" tokens expire after 7 days (doc).
  - Some API behaviours need confirming (§7, E3).
- **Effort:** a usable Android loop in roughly 2–3 weekends; the web extension about 1–2 more.

### Option 3 — Reverse-alias relay: rejected

**What it is.** It's how SimpleLogin and addy.io work, or we could build one on Forward Email
webhooks.

1. Inbound mail is rewritten so each sender appears as a relay address, e.g.
   `alice=example.org@r.example.net`.
2. Gmail's normal Reply goes to that address.
3. The relay swaps in `From: me@example.net` and sends it on.

It's the only option with zero extra taps, **but only for Reply and Reply-all.** When you
type a recipient in To: (or forward to someone new), Gmail delivers straight from Google to
that person's server, and no relay is in the path. You'd have to type an encoded address
like `bob=example.org@r.example.net`, or pick it from autocomplete next to the real one. So a
relay can only ever be an add-on to Option 4 or the Outbox, never a replacement.

**Rejected for the plan because it violates R7.** Every incoming email has to pass through the rewriter
before Gmail sees it. A bug, outage, misjudged authentication check or vendor problem means
mail that silently never arrives. It also:

- weakens Gmail's spam filtering, because the original sender's DKIM no longer verifies;
- breaks sender photos and unsubscribe;
- fills autocomplete with relay addresses;
- still leaks `me@gmail.com` whenever you type someone's real address.

**Possible side project:** running our own MX and rewriter (keeping smtp2go for outbound,
with Forward Email as a fail-open backup MX) could plausibly meet R7. It is written up in
[RELAY.md](RELAY.md): topologies, fail-open rules, anti-spam, bounces and observability.

### Option 4 — Gmail Android app: domain identity as an "Other" IMAP account

Use the Gmail app's still-supported third-party account feature.

**Setup:**

1. Add another account → Other → *Personal (IMAP)*, email `me@example.net`.
2. Incoming: `imap.gmail.com` with `me@gmail.com` and an app password. This logs back in to
   your own Gmail.
3. Outgoing: `mail.smtp2go.com:587` (or Forward Email SMTP).
4. Set sync to *Never* and turn notifications off. The account then exists only as a sending
   identity.

**Day to day:** reply in the Gmail account, switch *From* to `me@example.net`, then Send.

**Pros:** no code and no server. If it works, it's native Android sending with Gemini-written
drafts.

**Cons and unknowns (all untested):**

- The app may refuse `imap.gmail.com` in an "Other" account. No reports of refusal were
  found.
- The From switch may not work across accounts on a reply.
- The sent copy may land in a new "Sent" label rather than Gmail's real Sent Mail: AOSP
  Email, the app's ancestor, looks for a folder literally named "Sent" (reported). If so, the
  optional Sent fixer (DESIGN.md §4.7) can re-file those copies. If the app saves no copy at
  all, nothing can.
- A duplicate account sits in the account switcher.
- App passwords are unavailable with Advanced Protection.
- Android only; web needs something else.

**Verdict:** a 15-minute experiment (§7, E2). If it passes, it can be the Android answer with
no build, paired with something for web.

### Option 5 — Gmail web browser extension

**5a. Build:** an InboxSDK-based MV3 extension (MIT/Apache licence, actively maintained,
v2.2.30 on 2026-10-08) that adds a "Send as" button and cancels Gmail's native send on domain
threads (InboxSDK's `presending` hook) (doc). The extension doesn't send mail itself; it hands
the draft ID to a backend. This is the web half of Option 2 and not much use without it.

**5b. Buy:** MailBridge ($14/yr and up) offers an *early-access* extension that adds From
choices to Gmail web and sends via the source mailbox's SMTP (reported, unverified). It is
desktop only.

Closest open-source prior art is Plume (alpha). It adds a "Send as" button that sends
through a local helper and files a Sent copy via the Gmail API; plain text only (doc).

### Option 6 — Gmail add-on (Apps Script)

**Not recommended.** Add-ons do appear in the Gmail Android app, including compose actions.
But a compose trigger only gets the draft's recipients (`draftAccess: METADATA`): no body, no
draft ID. Nor can an add-on send or close the draft (doc). It could only guess which draft to
dispatch.

### Option 7 — Third-party client over the Gmail store

- **Android:** **FairEmail** supports Gmail OAuth with per-identity SMTP. It picks the
  identity that matches on reply, files sent copies into Gmail's Sent itself when the provider
  doesn't, threads by headers, shows labels as folders, and is actively maintained (doc).
  Thunderbird for Android has no per-identity SMTP; the workaround is a second account.
  Aqua Mail has per-identity SMTP (doc).
- **Desktop:** **Thunderbird/Betterbird** have per-identity SMTP and per-identity Sent folder
  (doc). eM Client can't.
- **Web:** self-hosted **Roundcube** with the `identity_smtp` plugin.
- **Lose Send-as in January 2027:** Mimestream, Superhuman, Shortwave and Spark, which ride
  on Gmail's own Send-as.
- **Pros:** off the shelf and robust. FairEmail is the best "works next week" stopgap for
  Android.
- **Cons:** a second UI for home mail. Work mail stays in the Gmail app, which violates R5. No
  Gemini, and Gmail's smart features are absent.

**Verdict:** the interim and exception path, not the destination.

### Option 8 — Build our own mail client

A Gmail-API Android app plus web app with a custom send path. It satisfies R1–R4, and it's
technically straightforward per message. But rebuilding search, threading, notifications,
offline, attachments, labels, Calendar cards and Gemini to Gmail's standard is a multi-month
project that never finishes, and it still fails R5 because work stays in the Gmail app.
**Not recommended.** Options 2 and 3 get ~95% of the benefit for ~5% of the effort.

### Option 9 — Domain mailbox at the provider, shown in the Gmail app

Turn on IMAP storage at Forward Email ($3/mo for 10 GB) and add `me@example.net` as an
"Other" account in the Gmail Android app. Desktop: Forward Email webmail or Thunderbird.

- **Pros:** officially supported by both Google and Forward Email; native Send on Android.
- **Cons:**
  - The domain mail lives outside Gmail, against R1. A forwarded copy into Gmail can keep
    Gmail as the archive, but sent mail won't be there unless synced.
  - No Gemini or Gmail spam filtering on that account, now that Gmailify is gone (doc).
  - No web UI inside Gmail.

### Option 10 — Migrate (not preferred)

What you'd lose by moving mail to another provider (Gemini over mail, Calendar's auto-added
events, Gmail spam filtering, one web UI), a provider-by-provider table against your criteria,
and a "Fastmail plus copies into Gmail" hybrid are in [MIGRATE.md](MIGRATE.md).

| Option | Price (2026) | Notes |
|---|---|---|
| **Fastmail** | ~$5–6/mo | Custom domain, wildcard send-as, Gmail import, two-way Google Calendar sync. No AI; an MCP server for external AI tools. Decent Android app. Fails R5: work stays in Gmail, so two UIs. |
| **Self-host** (Stalwart, Mox, Mailcow) | VPS + relay | Software is mature. Residential IPs are blocklisted and port 25 is blocked, so outbound needs a relay anyway. You become the on-call. Solves nothing Option 2 doesn't. |
| Proton, Zoho, iCloud+, Purelymail, Migadu, MXroute | $10/yr–$5/mo | Viable stores. All fail R1/R5 the same way. Proton has no IMAP on Android, so it can't be added to the Gmail app at all. |
| Microsoft 365 custom domain | — | No new custom domains since Nov 2023. |

### Cross-cutting: the Sent folder

Every path except Gmail's own Send has to get the copy into Gmail itself. `SENT` is a
system label that can only be set when a message is created, never added later, so how each
option files its copy matters. Several Gmail features depend on it: the Sent view, threading
of replies, collapsing your own list echoes, and reply nudges.

| Option | Who files the copy | Lands in Gmail's Sent? | Matches what recipients got? |
|---|---|---|---|
| 0 / 0b @gmail.com | Gmail | ✓ | ✓ |
| 2 mmail Outbox | mmail, at send time | ✓ if one of the strategies in [DESIGN.md §4.7](DESIGN.md#47-sent-copies-special-handling) works (E3b) | ✓, or From shows `me@gmail.com` under the fallback |
| 4 Gmail app IMAP identity | the app, via IMAP APPEND | maybe: it may create a "Sent" user label instead (E2); the Sent fixer can re-file those | ✓ |
| 7 FairEmail / Thunderbird | the client, via IMAP APPEND | ✓ once pointed at `[Gmail]/Sent Mail` | ✓ |
| 9 Domain mailbox at provider | the provider | ✗ only the provider's Sent, unless synced | ✓ |

## 4. Alternative workflows

The options above are building blocks. These are complete day-to-day workflows built from
them, ranked roughly by how close they stay to today.

| # | Workflow | Android | Desktop web | Extra effort per domain email | Built from |
|---|---|---|---|---|---|
| B | **Draft then dispatch** | Gmail app → Bcc `send@` → back → tap notification (or back → Review → Send) | Gmail web → "Send as" button | +1 tap and a Bcc, or +2 taps | 2 (+5a) |
| D | **Two-account Gmail app** | Gmail app, switch From to the IMAP identity | Thunderbird or Forward Email webmail for domain sends | +2 taps on Android; another app on desktop | 4 (+7) |
| E | **Domain on demand** | Gmail app sends as @gmail.com; FairEmail for the occasional domain send | Gmail web; Thunderbird or webmail for domain sends | 0 normally; switch apps when it matters | 0b + 7 |
| F | **Gmail address, domain replies** | Gmail app as @gmail.com with Reply-To domain | same | 0 | 0 |
| G | **Defer domain replies to the desk** | Star or label "reply-as-domain" on the phone | Batch replies in Thunderbird | Delay; batching | 7 |
| H | **Separate home client** | FairEmail for all home mail; Gmail app for work | Thunderbird for home; Gmail web for work | Two UIs | 7 |

Notes:

- **B and D combine.** If E2 passes, D covers Android natively and B's extension covers the
  web. That's the likely end state.
- **E and F** are where you land if you build nothing. E keeps sender-matched systems working
  at the cost of an occasional app switch. F is the simplest, with the downsides in Option 0.
- **D's desktop half** can be swapped for B's web extension later.
- **Any workflow can be phased:**
  - **Now → January:** today's Send-as, frozen.
  - **January → whenever B ships:** E or D.
  - **Then:** B.
- **There is no "nothing changes" workflow.** The only way to keep today's exact UX was a
  Google-hosted identity (Option 1), which splits your Google account.

## 5. Comparison

✓ good · ~ partial · ✗ poor

| Option | Android UX | Web UX | One UI with work (R5) | Gmail stays store (R1) | Gemini/Calendar intact | Inbound untouched | Build effort | Running cost | Fragility |
|---|---|---|---|---|---|---|---|---|---|
| 0 @gmail.com | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | none | $0 | none (but identity cost) |
| 0b Hybrid | ~ | ~ | ~ | ✓ | ✓ | ✓ | none | $0 | low |
| 1a Workspace identity | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | none | ~$84/yr | **rejected: splits Google identity (R6)** |
| 1b Move to Workspace | ✓ | ✓ | ✓ | ✗ | ~ | ✓ | migration | ~$84/yr | **rejected: splits Google identity (R6)** |
| 2 mmail Outbox | ~ (+1–2 taps) | ✓ (with ext) | ✓ | ✓ | ✓ | ✓ | medium | ~$0 | medium (our service, API quirks) |
| 3 Relay (bought or built) | ✓ replies only | ✓ replies only | ✓ | ✓ | ✓ | ✗ | none / high ([RELAY.md](RELAY.md)) | $0–120/yr | **rejected for the plan: rewriter on the inbound path (R7); replies only** |
| 4 Gmail app IMAP identity | ~ (+2 taps) | ✗ | ~ | ✓ | ~ | ✓ | none | $0 | **unknown until E2** |
| 5 Extension alone | ✗ | ✓ | ~ | ✓ | ✓ | ✓ | low–medium | ~$0 | medium (Gmail DOM) |
| 7 FairEmail + Thunderbird | ~ (2nd app) | ~ (desktop) | ✗ | ✓ | ✗ in those apps | ✓ | none | $0–small | low |
| 8 Own client | ~ | ~ | ✗ | ✓ | ✗ | ✓ | very high | ~$0 | high |
| 9 Domain IMAP in Gmail app | ✓ | ✗ | ~ | ✗ | ✗ | ~ | none | $36/yr | low |
| 10 Fastmail / self-host | ~ | ~ | ✗ | ✗ | ✗ | — | migration | $60+/yr | low / high |

## 6. Recommendation and decision tree

**This week:**

1. **Freeze** the current Send-as config. Write down each address, display name, SMTP
   server/port/user, and the Reply-To. Don't rotate the smtp2go or Forward Email passwords
   before January.
2. **E0 — inventory** (§7). Find out how much mail actually goes out as the domain, to whom,
   and which recipients are sender-matched systems. This decides how much Option 0's
   downsides really bite.
3. **E2 — Gmail app IMAP identity test.** 15 minutes. If it passes, it's the interim
   Android path, with no code.
4. **E3 — Gmail API spikes.** One or two evenings, to de-risk Option 2 before building it.

**Then:**

```
E2 passes (Gmail app sends as the domain via an "Other" IMAP account)?
├─ yes → Android: Option 4, native inside the Gmail app (workflow D).
│        Web: mmail in web-only mode — the extension plus a small dispatch
│        endpoint; no draft watching, no notifications (DESIGN.md §3).
│        Until that ships: Thunderbird or Forward Email webmail.
└─ no  → Android: full mmail Outbox with notifications (workflow B).
         Web: the same backend plus the extension.
         Until it ships: FairEmail (Android) + Thunderbird (desktop),
         i.e. workflow E.

Decide not to run anything? Workflow E (hybrid) or F (Reply-To), using the
E0 inventory to update the services that check your sender address.
```

**Why this order.**

- Workspace is ruled out (R6) and relays are ruled out (R7). That leaves no Google-native or
  zero-tap way to keep the domain as a sending identity.
- Option 4 is the least janky Android path, if Gmail allows it: two taps, all inside the
  Gmail app.
- The Outbox is the fallback for Android and the way to make the web feel native. Neither
  touches inbound mail. If either fails, the draft stays in Gmail and Gmail's own Send still
  works.

## 7. Experiments to run before January 2027

| ID | Experiment | Steps | Pass criteria |
|---|---|---|---|
| **E0** | Inventory of domain sending | Google Takeout → Mail → only the *Sent* label (mbox), then a local script (no OAuth needed). Alternatively Gmail search `in:sent from:(*@example.net)`. Count messages and recipients per domain over 1/3/10 years; flag helpdesk domains (zendesk.com, freshdesk.com, helpscoutapp.com, service-now.com, atlassian.net…), list traffic (List-Id on received mail) and ingest addresses. | A CSV of "who knows me only as @example.net" and "who checks my sender address". |
| ~~E1~~ | ~~Consumer → Workspace Send-as~~ | Dropped: Option 1 is rejected (R6). Don't verify the domain with Workspace even for a trial. | — |
| **E2** | Gmail app IMAP identity | On Android, set up as in Option 4. Reply to a thread in the Google account, switch From, Send. | The app accepts `imap.gmail.com`. The reply is delivered from `me@example.net` with DKIM pass. The sent copy shows in the *same Gmail thread* and in Gmail's Sent. Record exactly where it lands: the real Sent, another label (fixable by the Sent fixer), or nowhere (Option 4 then has no Sent copy). |
| **E3** | Gmail API behaviours (for Option 2) | With a throwaway OAuth client: (a) `drafts.get format=raw` on a reply drafted in the Android app — does it carry In-Reply-To/References? (b) the Sent filing strategies S1–S5 in DESIGN.md §4.7 (`mmail doctor`): which ones give `SENT`, thread, and show as "me"? Run now and again after the January cutover. (c) `watch` with `labelIds:[DRAFT]` — latency after leaving compose on Android; how often web autosave fires. (d) Does smtp2go keep our Message-ID? (e) Does `messages.send` rewrite a non-alias From? | (a)–(d) behave as DESIGN.md assumes; any deviation is noted there. |
| ~~E4~~ | ~~Reverse-alias trial~~ | Dropped: Option 3 is rejected (R7). | — |

## 8. Open questions

1. Will the `Reply-To` setting on `me@gmail.com` survive January 2027 (Option 0, workflow F)?
2. After January, will Gmail still recognise `me@example.net` as "me" in thread display and
   in the "to me" search operators once it is no longer a Send-as address? Will copies we file
   still get `SENT`? This affects Option 2's sent copies; see DESIGN.md §4.7. The alternate
   email on the Google account (R6) may be enough.
3. Will the Gmail Android app keep allowing an "Other" IMAP account that points at
   `imap.gmail.com`? (E2; Google could close this.)
4. App passwords: still offered but "not recommended". If they disappear, Option 4 breaks;
   Option 2 is unaffected (it uses OAuth).

## 9. Sources

Google:
- [Learn about changes to third-party email account support in Gmail (17101213)][g-17101213]
- [Send emails from a different address or alias (22370)][g-22370]
- [Upcoming changes to Gmailify & POP (16604719)](https://support.google.com/mail/answer/16604719)
- [Add another email account in the Gmail app (6078445)](https://support.google.com/mail/answer/6078445)
- [Sign in with app passwords (185833)](https://support.google.com/accounts/answer/185833)
- [Gmail IMAP is always on (7126229)](https://support.google.com/mail/answer/7126229)
- [OAuth app testing/publishing status (15549945)](https://support.google.com/cloud/answer/15549945),
  [unverified apps (13463817)](https://support.google.com/cloud/answer/13463817),
  [restricted scopes (13807380)](https://support.google.com/cloud/answer/13807380)
- [Gmail API push notifications](https://developers.google.com/workspace/gmail/api/guides/push),
  [messages.insert](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/insert),
  [messages.import](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/import),
  [threads guide](https://developers.google.com/workspace/gmail/api/guides/threads)
- [Workspace add-ons on mobile](https://developers.google.com/workspace/add-ons/guides/using-addons)
- [Gemini in Gmail by edition (13952129)](https://support.google.com/mail/answer/13952129)

Vendors and projects:
- [Forward Email FAQ (Send Mail As deprecation, webhooks, API)][fe-faq]
- [addy.io: sending from an alias](https://github.com/anonaddy/addy.io/blob/master/source/_articles/aliases/sending-email-from-an-alias.md),
  [SimpleLogin reverse alias](https://github.com/simple-login/documentation/blob/master/docs/getting-started/reverse-alias.md)
- [smtp2go API features](https://developers.smtp2go.com/docs/api-features-guide),
  [smtp2go free plan limits](https://support.smtp2go.com/hc/en-gb/articles/223087947)
- [FairEmail FAQ](https://github.com/M66B/FairEmail/blob/master/FAQ.md),
  [Thunderbird Android per-identity SMTP request #1117](https://github.com/thunderbird/thunderbird-android/issues/1117)
- [InboxSDK docs](https://github.com/InboxSDK/inboxsdk-docs), [Plume](https://github.com/CalvinKirs/plume)
- [MailBridge: Gmail Send As deprecated](https://mailbridge.app/posts/gmail-send-as-deprecated-january-2027/)
- [Cloudflare Email Service limits](https://developers.cloudflare.com/email-service/platform/limits/)

Discussion:
- Hacker News [49565693](https://news.ycombinator.com/item?id=49565693),
  [49162481](https://news.ycombinator.com/item?id=49162481),
  [49956092](https://news.ycombinator.com/item?id=49956092)
- [Proton: Gmail ends Send As for third-party addresses](https://proton.me/blog/gmail-ends-send-as-third-party)

[g-17101213]: https://support.google.com/mail/answer/17101213
[g-22370]: https://support.google.com/mail/answer/22370
[fe-faq]: https://forwardemail.net/en/faq
