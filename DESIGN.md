# mmail: design

Status: draft, 2026-10-15. Read [OPTIONS.md](OPTIONS.md) first for why we build this. This
document covers **Option 2 (mmail Outbox)** in detail, including a smaller web-only mode for
when the Gmail app handles Android (Option 4), the optional Sent fixer, and the inventory
tool (experiment E0).

Placeholders: `me@gmail.com` is the primary consumer Gmail account and mail store;
`me@example.net` is a custom-domain identity, one of several.

## 1. Goals and non-goals

Goals:

- **G1** Send mail `From: me@example.net`, DKIM/DMARC-aligned for `example.net`, through an
  external provider (smtp2go or Forward Email), after Gmail drops third-party Send-as in
  January 2027.
- **G2** Keep the **Gmail apps as the compose UI** on Android and web, so Gemini "Help me
  write", smart compose, contacts, attachments and signatures all keep working, and home and
  work share one interface.
- **G3** Keep **Gmail as the store**: the sent copy lands in the original thread, in Sent,
  searchable, visible to Gemini.
- **G4** Pick the right domain identity automatically when replying to mail that was sent to a
  domain address.
- **G5** Be safe by construction:
  - never send anything that isn't one of your drafts, confirmed by you;
  - never send twice;
  - never lose a draft;
  - never sit on the inbound path. If mmail is down, mail still arrives and Gmail's own Send
    still works (OPTIONS.md R7).
- **G6** Cost ≈ $0/month and need no routine babysitting.

Non-goals:

- A mail client: no reading, searching or rendering mail.
- Anything on the inbound path. Inbound mail keeps flowing Forward Email → Gmail untouched.
- Multi-user support, or publishing the OAuth app.
- Sending through Gmail's SMTP or API with a spoofed From. Gmail rewrites a non-alias From
  to the account address (reported), so it can't be the transport anyway.

## 2. User experience

### 2.1 Android: reply as the domain

1. Open the thread in the Gmail app and tap **Reply**. Write it, optionally with Gemini.
2. **Arm it:** add `send@example.net` to Bcc. Autocomplete makes this a couple of
   keystrokes. This is how you say "this draft is finished" (§4.3).
3. Instead of tapping Gmail's Send, tap **←** (back). Gmail shows "Draft saved".
4. Within a few seconds a notification appears:

   > **Send as me@example.net?**
   > To: Alice Smith (+1) · Re: Dinner on Saturday
   > "Sounds good, see you at 7 … Cheers"
   > [ **Send** ] [ Review… ]

5. Tap **Send**. The notification becomes "Sending as me@example.net… [Undo]" for 10 seconds
   (configurable), still showing the body's first and last lines, then "Sent ✓".
6. Back in Gmail, the draft is gone and the reply sits in the thread as sent by
   `me@example.net`. The arming address is stripped and never sent.

**If you skip step 2,** the notification only offers **Review…**. That opens the mmail PWA on
the draft, showing the full body, attachments, recipients and identity picker
(`me@example.net`, `shop@example.net`, …, or "leave it, I'll send from Gmail"), with a Send
button there. A paused, half-written draft can never be sent from a notification you
glanced at.

### 2.2 Android: new message

Same flow. Compose a new message in Gmail, Bcc `send@example.net`, go back, tap the
notification. The identity comes from recipient rules or the default (§4.4). By default mmail only offers notifications for
drafts it thinks belong to the domain (§4.3), so new messages also need either policy `all`
or a tap on the PWA's Outbox home-screen shortcut.

### 2.3 Desktop web

Phase 3 adds an extension (§4.9). It puts a **Send as ▾** button next to Gmail's Send in every
compose window, preselected with the resolved identity, and it intercepts Gmail's own Send on
domain threads:

> This conversation is with me@example.net. **Send as me@example.net** · Send from
> me@gmail.com anyway

Until then, web drafts trigger the same Web Push notifications in desktop Chrome, and the PWA
Outbox works in a tab.

### 2.4 Sending from @gmail.com as usual

Just press Gmail's Send. The draft disappears, mmail sees the deletion and withdraws its
notification. Nothing else happens.

### 2.5 Pressing Gmail's Send on a domain thread by habit

- **Android:** mmail can't stop it. After the fact it notices a `SENT` message from
  `me@gmail.com` in a thread whose identity resolves to the domain, or one carrying the arming
  address in Bcc (an armed draft sent the wrong way). It then pushes a heads-up so you can
  follow up.
- **Web:** the extension's guard prevents it.

## 3. Architecture

```
  Gmail app (Android)        Gmail web (+ mmail extension)
          │ drafts                    │ drafts           │ "Send as" / guard
          ▼                           ▼                  │
  ┌──────────────────────────────────────────┐           │
  │      Gmail mailbox (me@gmail.com)        │           │
  └──────────────────────────────────────────┘           │
     │ users.watch(DRAFT, SENT)   ▲ drafts.get/delete    │
     ▼                            │ threads.get(metadata)│
  Cloud Pub/Sub                   │ messages.insert      │
     │ push (OIDC)                │                      │
     ▼                            │                      ▼
  ┌──────────────────── mmail server (Cloud Run) ────────────────────┐
  │ change feed → draft tracker → identity resolver → notifier       │
  │ /dispatch → state machine → transformer → transport → filer      │
  │ Firestore (state) · Cloud Tasks (debounce, undo) · Secret Mgr    │
  └──────────────────────────────────────────────────────────────────┘
     │ Web Push                         │ SMTP submission (587/465)
     ▼                                  ▼
  mmail PWA + notification       smtp2go / Forward Email ──► recipients
  actions (Android, desktop)       (DKIM d=example.net)
```

**Everything runs in a Google Cloud project owned by `me@gmail.com`.** The only external
parties are the SMTP provider, which already sees the outbound mail today, and the browser's
push service, which carries only an encrypted, minimal payload.

**Web-only mode.** If E2 shows the Gmail Android app can send as the domain by itself
(OPTIONS.md Option 4), mmail only has to serve Gmail web. Then the change feed (§4.2), the
draft tracker and notifications (§4.3, §4.8), Pub/Sub and Web Push all go away. The extension
calls `/dispatch` with a draft ID, and everything from the identity resolver to the Sent copy
works unchanged.

The **durable execution parts stay**:
- **Cloud Tasks** runs the undo delay and each lifecycle step, so nothing depends on the
  HTTP request that started it.
- **Cloud Scheduler** runs the recovery sweep (§4.10) every few minutes. It enqueues the
  overdue next step of every non-terminal dispatch: a `CONFIRMED` submit, and filing steps
  driven by the filing status (`PENDING`, `FILING`, `FILED`). It also expires `SENDING`
  leases into `UNCERTAIN` and raises the unresolved-dispatch alerts.

With min instances at zero, a request that ends or a process that dies never strands a
dispatch. That's still a much smaller service: Cloud Run, Firestore, Secret Manager, Cloud
Storage/KMS, Cloud Tasks and Cloud Scheduler.

## 4. Components

### 4.1 Gmail access

- **OAuth client:** a Google Cloud project with the OAuth consent screen set to *External*
  and published "In production", unverified.
  - "Testing" status would expire refresh tokens after 7 days (doc).
  - Unverified production apps show a warning screen, are capped at 100 lifetime users, and
    need no verification or CASA assessment for personal use (doc).
  - Production refresh tokens don't expire on a timer. They are revoked by a password change
    (because of the Gmail scopes), by 6 months unused, or after 100 newer tokens (doc).
  - OAuth clients unused for 6 months are deleted automatically (doc). Ours is used daily.
- **Scopes:** the narrowest set that works.

  | Scope | Tier | Used for |
  |---|---|---|
  | `gmail.compose` | restricted | `drafts.list`, `drafts.get(format=raw)`, `drafts.delete` |
  | `gmail.metadata` | restricted | `users.watch`, `history.list`, `threads.get(format=metadata)` for identity and threading headers; reading back a filed copy's labels |
  | `gmail.insert` | restricted | `messages.insert` / `messages.import` for the sent copy (§4.7) |
  | `gmail.labels` | non-sensitive | create the `mmail/…` audit labels |
  | `gmail.modify` | restricted | **optional Sent fixer only**, as a separate grant held only by the fixer job (§4.7) |

  - We avoid `gmail.modify`/`gmail.readonly`, because mmail never needs to read message
    *bodies* other than drafts.
  - Caveat: `gmail.compose` also allows sending via Gmail. It's unavoidable and unused.
  - That `drafts.get raw` works alongside `gmail.metadata` is verified in E3.
- **Token storage:** the refresh token lives in Secret Manager, readable only by the
  service's runtime service account. A failed refresh raises an alert (§9) and a re-auth link
  in the PWA.
- **Why not IMAP?** IMAP with an app password would avoid OAuth setup (IDLE on
  `[Gmail]/Drafts`, APPEND to `[Gmail]/Sent Mail`). But it grants full mailbox access, needs a
  persistent connection, loses `threadId`, and app passwords are "not recommended" and missing
  under Advanced Protection. Keep it as a fallback transport for the filer only.

### 4.2 Change feed

- **Watch:** `users.watch` with `labelIds: [DRAFT, SENT]` and `labelFilterBehavior: include`,
  publishing to a Pub/Sub topic. The topic grants `gmail-api-push@system.gserviceaccount.com`
  the Publisher role.
  - A push subscription delivers to `POST /pubsub` with an OIDC token, which we verify.
  - Cloud Scheduler renews the watch daily. It must happen at least every 7 days; daily is
    what Google recommends (doc).
- **Push handling:** the notification only carries a new `historyId`.
  1. Call `history.list(startHistoryId=last)` and collect touched message IDs.
  2. Then call `drafts.list` and diff it against the known `{draftId → messageId}` map.
  3. Every draft edit replaces the draft's message (`messageDeleted` + `messageAdded`), so
     **drafts are tracked by draft ID, never by message ID** (reported).
- **Reconcile:** Cloud Scheduler runs the same diff every 10 minutes, and the PWA runs it on
  open. Push is "typically within a few seconds" but can be delayed or dropped (doc).
- **Lost history:** if `history.list` returns 404 because the history ID is too old, do a
  full resync from `drafts.list`.
- **Quota:** watch costs 100 units and history/drafts calls cost single-digit units, against
  a 15k units/min/user limit (doc). One user won't come close. (Google introduced quota tiers
  in May 2026; re-check the numbers, but personal volume is far below any threshold.)

### 4.3 Draft tracker and notification policy

- **Debounce.** A draft is *settled* when its message ID hasn't changed for `settle_seconds`
  (default 6).
  - Each change enqueues a Cloud Task `evaluate(draftId, messageId)` with that delay.
  - When the task runs, it's dropped if the draft has moved on.
  - Gmail web autosaves while you type, so without this, notifications would flicker
    mid-compose.
- **When to offer** a settled draft. The draft needs at least one recipient, and the policy
  must match:
  - `domain` (default): the resolved identity (§4.4) came from the *thread* or a recipient
    rule, not the fallback default.
  - `all`: every draft with recipients.
  - `none`: never; use the PWA or extension explicitly.
- **Notifications:** at most one per draft (Web Push `tag = draftId`). A newer settled version
  replaces it, and it is closed when the draft is deleted or sent from Gmail.
- **Opting out:** you can silence a specific draft ("Not this one") or a thread.

**Settled is not ready.** A draft that hasn't changed for a few seconds might be finished, or
you might be reading, looking something up, or offline mid-sentence. Gmail's autosave makes
both look the same. So **one-tap Send is offered only for *armed* drafts**. A settled
draft that isn't armed gets a notification with only **Review…**, which opens the full draft in
the PWA before anything can be sent.

Arming signals, explicit and per platform:

- **Web:** clicking the extension's **Send as** button (§4.9) is itself the signal; it
  dispatches directly, with no notification. The extension also reports which drafts are open
  in a compose window, and those are never offered by notification on any device.
- **Android:** adding the **arming address** (`send@example.net`, configurable) to To/Cc/Bcc.
  - It's an ordinary Forward Email alias that delivers to Gmail under a filter that archives
    it, so nothing new sits on the inbound path.
  - The transformer strips it from the headers and the envelope (§4.5).
  - If Gmail's own Send is pressed on an armed draft by mistake, the resulting `SENT` message
    from `me@gmail.com` carries the arming address in Bcc. mmail spots that and warns
    (§2.5).
- **Opt-in `trust_settle`:** for anyone who accepts the risk, a long settle (e.g. 60 s) counts
  as armed. It's off by default.

Even an armed draft is re-checked at send time: its message ID must equal the one you saw
(§4.10). The Send and undo notifications show the body's first and last lines and the
attachment count, so a truncated body is visible before the undo window closes.

### 4.4 Identity resolver

Configuration:

```yaml
identities:
  - address: me@example.net
    name: "Your Name"
    default: true
  - address: shop@example.net
    name: "Your Name"
catch_all: example.net        # any *@example.net we receive at is "ours"
rules:                        # recipient-based, first match wins
  - recipient_domain: [zendesk.com, freshdesk.com]
    identity: me@example.net
fallback: ask                 # ask | default
arming_address: send@example.net
trust_settle: false           # true: a 60 s settle counts as armed (risky)
```

Resolution order:

1. **Explicit choice**: from the PWA, the extension, or the action token.
2. **Thread continuity**: walk the thread newest-first (`threads.get format=metadata`, with
   `metadataHeaders` including `From`, `To`, `Cc`, `Delivered-To` and `X-Mmail-Sent-As`).
   - If one of *our* earlier messages names a domain identity, use that identity. That means
     either `From:` a domain identity, or carrying `X-Mmail-Sent-As` because it was filed
     under strategy S4 (§4.7), where the archived `From` is `me@gmail.com`.
   - Otherwise take the first address in `To`/`Cc`/`Delivered-To` of inbound messages that
     matches an identity or the catch-all domain. With catch-all, that's the exact address the
     sender used, e.g. `amazon@example.net`.
   - Forward Email forwarding keeps the original `To`.
3. **Recipient rules.**
4. **Fallback**: `ask` shows the notification with no preselection (only *Review…*); `default`
   preselects the default identity.

### 4.5 Message transformer

Input: the draft's raw RFC 5322 bytes from `drafts.get(format=raw)`, plus `threadId`.

Rule: **only the header block is rewritten. Body bytes, including MIME structure,
attachments and inline `cid:` images, pass through untouched.**

1. Split at the first `CRLF CRLF`. Parse header fields (unfold, keep the original order) with
   a real RFC 5322/2047 address parser.
2. **Remove** `From`, `Sender`, `Bcc`, `Message-ID`, `Date`, `Return-Path`, `Received`,
   `X-Gm-*`, `X-Google-*`, `DKIM-Signature`, `ARC-*`, `Authentication-Results`. Drafts
   shouldn't carry most of these; removing them is defensive.
3. **Set** these headers:
   - `From: "<name>" <identity>` (RFC 2047-encode the name if needed)
   - `Date: <now>`
   - `Message-ID: <ULID@example.net>`

   The Message-ID is generated once per dispatch and persisted *before* submission.
4. **Keep** `To`, `Cc`, `Subject`, `Reply-To` (if you set one), `In-Reply-To`, `References`,
   `MIME-Version`, `Content-*` and anything else.
5. **Reply or new message?** Gmail gives *every* draft a `threadId`, even a brand-new one.
   The dispatch is classified as a **reply** only if both of these hold:
   - the thread contains a non-draft parent message;
   - the draft's `Subject` matches the thread's subject, once leading **`Re:`** prefixes and
     whitespace are stripped and case is ignored. **`Fwd:`/`Fw:` are *not* stripped**, so
     a Gmail forward (`Fwd: Original`) never matches its source thread and is always new
     for filing. Gmail's threading contract requires matching subjects as well as
     `threadId` and reply headers.

   Otherwise it is **new for filing purposes**. That includes a forward or reply whose
   subject you edited. Its outbound threading headers are kept as Gmail wrote them, so
   recipients still see the relationship, but the sent copy is filed without `threadId`. If
   an insert *with* `threadId` is still rejected as a threading mismatch, the filer
   reconciles (§4.10), commits the switch (`filingMode: unthreaded`), and then re-files
   without `threadId`, rather than leaving it `SENT_UNFILED`.
   - **New messages** get no threading headers. Their sent copy is filed **without**
     `threadId` (§4.7), so Gmail starts a fresh thread. Gmail's threading rules need a
     parent for `In-Reply-To`/`References`, so filing a new message into its draft thread
     would fail after the mail had already gone out.
   - **Threading repair, replies only.** If `In-Reply-To`/`References` are missing (E3a
     checks whether Gmail drafts carry them):
     - find the newest non-draft message in the thread;
     - set `In-Reply-To` to its Message-ID;
     - set `References` to its References plus its Message-ID.
6. **Envelope:**
   - `MAIL FROM` is the identity address. The provider rewrites Return-Path to its bounce
     domain or our custom return-path CNAME.
   - The arming address (§4.3) is removed from `To`/`Cc`/`Bcc`, from the envelope, and from
     the sent copy. A draft whose *only* recipient was the arming address is refused.
   - `RCPT TO` is the de-duplicated set of addresses in `To` + `Cc` + `Bcc`. Group syntax and
     encoded words are handled by the parser.
7. **Limits:** refuse anything over 25 MB. Gmail already converts large attachments to Drive
   links.
8. **Outputs:**
   - the *outbound message*: the headers above plus the body. Each submission attempt adds
     its own `X-Mmail-Attempt: <dispatchId>/<n>` (§4.6);
   - the *sent copy*: the outbound message plus the original `Bcc` (so you can still see
     whom you Bcc'd) plus `X-Mmail-Dispatch: <dispatchId>`.

### 4.6 Transport

- **Primary: SMTP submission** with an explicit envelope:
  - smtp2go `mail.smtp2go.com:587` (STARTTLS); or
  - Forward Email `smtp.forwardemail.net:465`.

  This is provider-neutral, sends raw bytes verbatim and keeps Bcc private. Cloud Run allows
  egress on 587 and 465 (only 25 is blocked).
- **HTTP alternatives**, for runtimes without TCP such as Workers or Apps Script:
  - smtp2go `POST /email/mime` (base64 `mime_email`);
  - Forward Email `POST /v1/emails` with `raw` (From must be an alias on the domain) (doc).

  How each derives envelope recipients from `raw` is undocumented, so test Bcc before relying
  on them.
- **Result handling.** The outcomes below are **per attempt** (§4.10). The dispatch's
  delivery status *aggregates* them:
  - **`FAILED`** only when no attempt has ever had a recipient accept the message.
  - **Once any recipient has accepted,** a later attempt's failure (pre-`DATA`, auth,
    connection) is recorded on that attempt only. The dispatch stays **`SENT_PARTIAL`**
    with its rejected set unchanged, so the warning and "retry rejected only" remain
    available.
  - **An `UNCERTAIN` retry attempt** also leaves the dispatch `SENT_PARTIAL`, but blocks
    further retries to those recipients until that attempt is resolved.

  Per attempt:
  - 2xx with **every** recipient accepted → `SENT`.
  - 2xx with **some** recipients rejected → **`SENT_PARTIAL`**. A `RCPT TO` can fail for one
    address while the rest are accepted and delivered. Nodemailer resolves successfully and
    lists the failures in `info.rejected`; the HTTP APIs report them per recipient.
    - The accepted and rejected sets are recorded. The dispatch files and cleans up like
      `SENT`.
    - The sent copy gets an `X-Mmail-Rejected` header and an `mmail/partial` label.
    - You're told who didn't get it and why.
    - It is **never retried as a whole**, which would duplicate delivery to the accepted
      recipients. "Retry rejected only" re-submits the same bytes and Message-ID with an
      envelope of just the rejected addresses, as another attempt of the same dispatch (§4.10).
  - Every recipient rejected (no `DATA` phase) → `FAILED`, and the draft is kept.
  - **Failures before `DATA`** (connection refused, TLS or auth errors, a temporary 4xx at
    `MAIL`/`RCPT` for every recipient) are safe to retry, because nothing was accepted. They
    get a few automatic retries with backoff, then `FAILED`. "Retry" is the next attempt of
    the same dispatch (§4.10).
  - 5xx before `DATA` → `FAILED`, and the draft is kept.
  - Timeout or connection loss after `DATA` → **`UNCERTAIN`**. Never auto-retry; ask the user,
    because a duplicate is worse than a delay.
  - The process dies mid-submission. The provider may already have answered 2xx, but `SENT`
    was never committed. When the `SENDING` lease expires with no result recorded, recovery
    moves the dispatch to **`UNCERTAIN`**, never back to sending. This works because
    `SENDING` and the outbound Message-ID are committed *before* the first byte goes to the
    provider, so recovery always knows a submission may have happened.
  - **Resolving `UNCERTAIN`:** if the provider exposes a searchable send log (smtp2go's
    activity search, Forward Email's emails API; untested), recovery looks up the **current
    attempt's marker**, not just the Message-ID. Retries reuse the Message-ID (§4.10), so a
    Message-ID match can belong to an earlier attempt.
    - Each attempt carries `X-Mmail-Attempt: <dispatchId>/<n>`, or the provider's own
      tag/metadata field where it has one. Its provider reference is recorded the moment the
      submission returns.
    - **Only a record carrying the current attempt's marker counts.** Records that can't be
      uniquely attributed to the current attempt, such as an earlier attempt's result for
      the same recipient, are ignored, and the attempt stays `UNCERTAIN`.
    - **A positive, attributable record with per-recipient results for every recipient in
      that attempt's envelope** resolves it automatically:
      - all accepted → `SENT`;
      - some rejected → `SENT_PARTIAL`, with the accepted and rejected sets, the warning and
        "retry rejected only".
    - **A positive record without per-recipient detail** only proves the message was
      accepted for *someone*. It stays `UNCERTAIN`, now marked "accepted, recipients
      unknown". It never resolves to full `SENT`, so a lost rejection can't masquerade as
      success.
    - **No record, an unavailable log, or anything inconclusive leaves it `UNCERTAIN`.**
      Logs can lag or be incomplete even after acceptance, so a missing record never becomes
      `FAILED` and never triggers a resend. You decide.
    - **An `UNCERTAIN` attempt keeps polling.** Its track keeps a `nextStepAt` on a widening
      schedule (e.g. 5 min, 15 min, 1 h, then every 6 h). When it's due, the sweep re-runs
      the lookup, so a record that turns up late still resolves it as above. Polling stops
      once the provider's log retention has passed. The attempt is then parked with no
      `nextStepAt`, and only you can resolve it. Polling only ever reads the log; it never
      resubmits.
- **DNS:** unchanged from today's Send-as setup: provider DKIM, and SPF or a custom
  return-path. Once mmail is the only outbound path, consider moving DMARC to `p=quarantine`.
- **Provider limits:** smtp2go free is 1,000/month and 200/day (25/hour until the domain is
  verified). Forward Email starts at 300/day (doc).

### 4.7 Sent copies (special handling)

When Gmail sends a message itself, it files the copy, applies the system `SENT` label,
threads it and shows it as "me". When anything else sends, none of that happens on its own.
`SENT` is also a system label that can't be added to an existing message later: the labels
guide says it is applied automatically and can't be applied manually (doc). So whatever we do
has to happen **when the copy is created**, and a wrong first attempt can't be patched with
our narrow scopes.

#### What depends on the copy being a proper Sent message

| Gmail behaviour | Needs `SENT` | Needs From recognised as yours | Needs the delivered Message-ID |
|---|---|---|---|
| Sent view in web and apps; `in:sent` | ✓ | | |
| The reply shows as "me" in the thread; `from:me`; Gemini attributes it to you | | ✓ | |
| Recipients' replies thread with your copy | | | ✓ (plus `References`, Subject) |
| An inbound echo of your own message (Cc to yourself, mailing-list copy) is collapsed rather than shown as new mail | probably | | ✓ |
| Nudges ("Received 3 days ago. Reply?") stop once you've replied | probably | probably | |
| Importance and "Other contacts" autocomplete learn from people you write to | probably | probably | |

"Probably" means inferred from how these features must work, not documented. Whether Gmail
still recognises `me@example.net` as yours once it stops being a Send-as alias in January is
unknown. The alternate email on the Google account (OPTIONS.md R6) may be enough.

#### Filing strategies

Ranked by preference. Each one gets tested in E3b, now while `me@example.net` is still a
Send-as alias **and again after the January cutover**. Wherever a strategy passes
`threadId`, it does so only for **replies** (§4.5 step 5). New messages are always filed
without one.

| # | Method | Gets `SENT`? | Keeps true From? | Threads? | Cost / risk |
|---|---|---|---|---|---|
| **S1** | `messages.insert` with `labelIds: [SENT]`, `threadId` (replies only), `internalDateSource=dateHeader` | Docs say `SENT` is auto-applied only when From is the user's address. Explicit `SENT` on insert is common practice but unverified for a non-alias From. | ✓ | ✓ via `threadId` | `gmail.insert` only |
| **S2** | `messages.import`, same labels, `neverMarkSpam=true`, `processForCalendar=false` | GYB restores sent mail by passing `SENT` on import (reported; those copies have your own From) | ✓ | ✓ | Same scope. Runs Gmail's classifier. |
| **S3** | IMAP `APPEND` to `[Gmail]/Sent Mail` | ✓ This is how Thunderbird and FairEmail file external-SMTP sends | ✓ | by headers only (no `threadId`) | Needs an app password (full mailbox access) or the `mail.google.com` scope; a second auth path |
| **S4** | `messages.insert` with From rewritten to `me@gmail.com`, plus `X-Mmail-Sent-As: me@example.net` and label `mmail/as/me@example.net` | ✓ (documented auto-apply) | ✗ the archive shows the Gmail address | ✓ | Your archive differs from what recipients saw. The header and label keep the truth and let identity resolution (§4.4) find it. |
| **S5** | `messages.insert` with user label `mmail/sent` only | ✗ | ✓ | ✓ | Nothing in the Sent view; nudges and echoes misbehave |

**Choosing a strategy.** We can't silently fall back at runtime: if S1 "succeeds" without
`SENT`, the bad copy is already in the mailbox, and removing it needs `gmail.modify` or more.
So:

- **`mmail doctor`** files a test message into a throwaway thread with each strategy and
  reads back `labelIds` (`messages.get format=minimal`, allowed under `gmail.metadata`). It
  reports which strategies give `SENT`, thread correctly and show as "me". Run it at setup and
  again after the January cutover.
- **The winner is pinned in config** as `sent_strategy`.
- **After every real filing**, the filer reads back the copy's labels. If `SENT` is missing it
  raises an alert to re-run `mmail doctor`. It never retries with another strategy, which would
  file a duplicate.

#### Rules for the copy itself

- **File only after the provider accepts the message (2xx), never before.** A copy filed
  first would be a phantom if submission fails, and we can't delete it with our scopes.
- **Same bytes as delivered**, plus the original `Bcc` (so you can see whom you Bcc'd) and
  `X-Mmail-Dispatch`. The `Date` is stamped at submission time, after the undo window, so the
  copy sorts correctly in the thread.
- **Same Message-ID as delivered** (A4). If the provider replaces it, read the provider's
  value back and use that. Otherwise recipients' replies won't thread with your copy, and
  echoes won't be collapsed.
- **No `INBOX` and no `UNREAD`** on the copy, so sending doesn't resurface the thread as
  unread. An optional `archive_on_send` setting emulates Gmail's "Send & Archive", but it needs
  `gmail.modify` to remove `INBOX` from the thread, so it's off by default.
- **File the copy, then delete the draft.** A leftover draft is better than a missing copy.
  But a leftover draft is also a **footgun**: pressing Gmail's Send on it sends a duplicate
  from `me@gmail.com`. So the delete is retried on every reconcile, and the notification
  says "Sent — remove the leftover draft" until it's gone.
  - `drafts.delete` runs only if the draft's message ID still equals the one we sent. If you
    edited the draft after confirming, it is kept and you're told.
- **Echoes.** If you Cc yourself (either address) or post to a list you're on, an inbound copy
  with the same Message-ID arrives via Forward Email. Gmail collapsing it is expected only
  with an S1–S3 copy (A8). Otherwise mmail knows the outbound Message-ID and can flag the
  echo. Hiding it from the inbox would need `gmail.modify`.

#### Sent copies from the other paths

These matter if Option 4 or the interim workflows (OPTIONS.md D/E) are in use:

- **Gmail app "Other" IMAP identity (Option 4):** the app APPENDs the copy itself. E2 tells us
  where it lands:
  - in `[Gmail]/Sent Mail` (fine);
  - in some other label, e.g. a "Sent" folder the app created (fixable, below);
  - nowhere, if Gmail refuses the folder (not fixable).
- **FairEmail / Thunderbird (Option 7):** point the identity's Sent folder at
  `[Gmail]/Sent Mail`. In Thunderbird that's the identity's Copies & Folders setting; FairEmail
  does it automatically.

#### Optional: Sent fixer

Only needed if E2 shows Option 4's copies landing in the wrong label. It's a background
tidy-up, **off the critical path**: the mail has already gone out, and the fixer only
corrects where your copy of it is filed.

1. **Find** messages that are in the label the app uses (from E2), are `From:` one of your
   domain identities, and don't carry the system `SENT` label. Triggered by `users.watch` on
   that label, or a run every few minutes.
2. **Re-file** a copy with the pinned strategy (S1/S2): same bytes, same Message-ID, same
   thread.
3. **Verify** that the new copy carries `SENT` and sits in the same thread. Only then
   **move the original to Trash**, which is recoverable for 30 days. It never hard-deletes.
4. **Log** each fix and label the new copy `mmail/fixed`.

Safeguards (OPTIONS.md R7):

- opt-in, with a dry-run mode that only reports what it would do;
- matches only that narrow pattern;
- idempotent and resumable. If a verified `SENT` copy with the same Message-ID already
  exists in the same thread (e.g. a run crashed after re-filing but before trashing), it
  doesn't re-file. It **resumes the remaining steps**: trash the misfiled original, then
  label and log. Nothing is skipped, so no duplicate is left behind;
- runs under its **own** OAuth grant with `gmail.modify`, so the dispatcher keeps its narrow
  scopes;
- if Gmail ever starts filing the copies correctly, it finds nothing and does nothing.

Limits:

- it can't help if no strategy yields `SENT` (then S4/S5 only), or if the app saves no copy at
  all;
- a copy sits in the wrong label until the next run.

### 4.8 Notifier and PWA

- **Web Push (VAPID)** from the server to the PWA's service worker, which works in Android
  Chrome and desktop Chrome.
  - The payload holds `draftId`, identity, a recipient summary, the subject, the body's
    first and last lines, and the attachment count. For **armed** drafts only (§4.3), it also
    carries a signed **action token**:
    `HMAC(key, action | target | draftId | draftMessageId | identity | generation | confirmationId | expiry)`,
    valid for 15 minutes and usable once (§4.10). Replaying it only ever returns the same
    dispatch (§4.10).
  - **Expired tokens are refreshed on tap.** A notification can sit untouched for longer
    than 15 minutes. If the embedded token has expired when you tap Send, the service
    worker uses its device credential to fetch a fresh one for the **same draft version,
    generation and identity** (`GET /dispatch/{draftId}/token?identity=…`), then posts that.
    - If the draft or generation changed meanwhile, the server returns the current state
      and the notification is updated instead. You're never asked to confirm a version you
      didn't see.
    - A daily sweep also withdraws notifications for drafts untouched for over 24 hours.
  - Actions:
    - armed drafts: **Send** (posts the token to `/dispatch` in the background, refreshing
      it first if expired) and **Review…** (opens the PWA);
    - unarmed drafts: **Review…** only.

    Android shows two action buttons comfortably.
- **PWA screens:**
  - *Outbox*: settled drafts with identity picker, preview and Send.
  - *History*: dispatches with their state, plus "Open in Gmail" links.
  - *Settings*: identities, rules, policy, undo delay, devices.
- **Auth:** Google Sign-In (OIDC) restricted to `me@gmail.com`, and a session cookie. Each
  device registers its push subscription.
- **Quick-prototype alternative:** ntfy, with `http` actions (up to 3, sent in the
  background by the Android app). Use a self-hosted ntfy or an access-controlled topic; never
  put subjects on a public topic.

### 4.9 Gmail web extension (phase 3)

- **Build:** Chrome MV3 with `@inboxsdk/core` (MV3-compatible, no remote code, actively
  released). It needs a free InboxSDK AppId (doc).
- **Compose button.** Add **Send as ▾** to each compose view, preselected via
  `GET /identity?threadId=…`. On click:
  1. make sure the draft is saved (wait for InboxSDK's saved state, then `getDraftID()`);
  2. fetch a confirmation token for the saved draft version **and the identity chosen in
     the dropdown** (`GET /dispatch/{draftId}/token?identity=…`, authenticated with the
     device token), then `POST /dispatch {token}` (§4.10). The token carries the identity,
     so there's no separate identity field for the server to trust;
  3. close the compose view without re-saving.
- **Guard.** On `presending`, if the thread resolves to a domain identity, call
  `event.cancel()` and show the choice dialog (§2.3) (doc for `presending`/`cancel`).
- **Pairing:** the extension gets a device token from the PWA via a one-time code, kept in
  `chrome.storage.local`. The extension never holds Gmail OAuth tokens.
- **Spike items:**
  - whether the InboxSDK draft ID equals the API draft ID (E3);
  - how to force or await a save;
  - closing without resurrecting the deleted draft.
- **Prior art:** Plume, an alpha extension that hands drafts to a local SMTP helper.

### 4.10 Dispatch state machine

The diagram shows the common path as one line. In storage (§5), a dispatch's **delivery
status** (`CONFIRMED` … `SENT` / `SENT_PARTIAL` / `FAILED` / `UNCERTAIN`) and its **filing
status** (`PENDING` / `FILING` / `FILED` / `DONE` / `UNFILED`) are separate fields. That
way, filing a copy and deleting the draft never erase a pending partial-delivery decision.

```
OFFERED ──confirm──► CONFIRMED ──undo window──► SENDING ──2xx──► SENT ──insert──► FILED ──delete──► DONE
   │                    │ undo                     │ 5xx            │ insert fails         │ draft edited
   ▼                    ▼                          ▼                ▼ (retry w/ backoff)   ▼
WITHDRAWN           CANCELLED                   FAILED          SENT_UNFILED          DONE_KEPT_DRAFT
                                                   │ timeout after DATA, or
                                                   │ lease expired with no result
                                                   ▼
                                               UNCERTAIN (provider log lookup, else user decides)
```

`WITHDRAWN` is terminal. A draft version reaches it when the draft is deleted, sent from
Gmail or edited before mmail sends it, including from `CONFIRMED` via the pre-send check
(below). A transient failure before `DATA` parks the attempt in `RETRY_WAIT` and returns to
`SENDING` when its backoff is due (not drawn). The diagram's `SENT_UNFILED` is delivery
`SENT` (or `SENT_PARTIAL`) with filing `UNFILED`. Its `DONE_KEPT_DRAFT` is filing `DONE`
with `draftKept` set.

- **One dispatch per draft version, for its whole life.** The dispatch document's ID is
  `hash(draftId, draftMessageId)`, created with create-if-absent in the confirming
  transaction.
  - Every later `/dispatch` for that version returns the *existing* dispatch and its
    current state, rather than starting a new one. That covers a repeated tap, a lost HTTP
    response, a replayed action token, or the notification and extension both firing.
  - This holds whatever state it's in: `SENDING`, `SENT`, `FILING` or `DONE`.
  - A new *dispatch* of the same draft requires an edited draft, which means a new message
    ID and a new confirmation. **Retries are *attempts* within the same dispatch**, keyed
    `(dispatchId, attempt)` so each is idempotent too:
    - after `FAILED`: nothing was accepted, so "Retry" (or "Retry via secondary provider")
      starts the next attempt, with no edit needed;
    - while delivery is `SENT_PARTIAL` with rejected recipients outstanding: "retry rejected
      only", with an envelope of just those addresses. This works **whatever the filing
      status** (`FILING`, `FILED`, `DONE` or `UNFILED`). It never files a copy itself: the
      Sent copy is the filing track's job, and a filed one is reused;
    - after `UNCERTAIN`: **never automatically**. Only your explicit "I've checked, resend
      anyway", which warns that it may duplicate.

    Every attempt submits the stored outbound MIME (below), with the same body and the same
    Message-ID. Only the `X-Mmail-Attempt` header differs, so the provider log can tell
    attempts apart (§4.6). The lease allows one in-flight attempt per dispatch at a time.
- **Outbound MIME is kept until the dispatch is resolved.** Retries, partial retries and
  filing recovery all need the exact bytes the recipients got. The draft may be deleted
  (after `SENT`) or edited by then, and `gmail.compose` + `gmail.metadata` can't read the
  filed Sent copy's body.
  - **When:** the outbound message and the sent-copy bytes are written at `SENDING`,
    *before* submission.
  - **Where:** a Cloud Storage object with a customer-managed key (Cloud KMS), one per
    dispatch.
  - **Deleted:** once the dispatch is resolved: `DONE` with no partial retry pending, or
    explicitly closed by you.
    - **Unresolved objects are kept indefinitely,** however long an `UNCERTAIN` or
      partial-retry decision takes. The daily health job alerts on any dispatch unresolved for
      more than 3 days.
    - **Resolution** moves the object under a `resolved/` prefix before deleting it. The
      bucket's age-based lifecycle rule (e.g. 14 days) matches **only that prefix**. It's a
      backstop for failed deletes, so it can never remove bytes an open dispatch still needs.
  - **If it's gone:** recovery **never reconstructs from the current draft**. It alerts
    instead.
- **Lease:** each transition is a Firestore transaction. `SENDING` takes the attempt's
  lease, so the worker that holds it is the only one that may submit. The same
  transaction refuses to start an attempt while another is `SENDING` or `RETRY_WAIT`, so
  only one attempt is in flight per dispatch. Filing has its own lease (below), so it neither blocks nor is
  blocked by a retry. The lease decides *who* may send. It is never used to *retry*: an
  expired `SENDING` lease becomes `UNCERTAIN` (§4.6).
- **Partial sends:** `SENT_PARTIAL` (§4.6) runs filing and draft deletion just like `SENT`,
  but its delivery status stays `SENT_PARTIAL` until every rejected recipient has been
  retried successfully or you dismiss them.
  - Each "retry rejected only" action creates a **new attempt** keyed
    `(dispatchId, attempt n)`, where `n` increments with every retry action. It submits the
    stored outbound MIME to the rejected set.
    - Replays of that *one* action are de-duplicated by its server-issued one-time token
      (the same token model as confirmations, bound to this action and attempt `n`). A
      repeated tap, or a retried request, returns attempt `n` rather than starting another.
    - After an attempt fails before `DATA`, the next retry action gets attempt `n+1` and
      really submits again.
  - A successful retry updates `acceptedRcpts`/`rejectedRcpts` and the delivery status,
    without touching the filed Sent copy.
  - **"Dismiss"** (`dismiss-rejected`, bound to the current rejected-set hash) moves the
    outstanding rejected recipients to `dismissedRcpts`.
    - It's refused while an attempt is `SENDING` or `RETRY_WAIT`.
    - Delivery stays `SENT_PARTIAL`, since that's still the truth, but nothing remains
      outstanding.
    - If the rejected set has changed since the token was minted, e.g. a retry landed in
      between, the hash no longer matches and you're shown the new state.
  - The dispatch counts as resolved, so its stored MIME can be deleted, only when filing is
    `DONE` *and* no rejected recipient is outstanding (`rejectedRcpts` is empty, because each
    was either retried successfully or dismissed). Whichever of filing `DONE`, a successful retry or a dismissal comes last
    resolves it: its stored MIME is deleted and the unresolved-dispatch alerts stop.
- **Write-ahead:** `SENDING`, with the outbound Message-ID, is committed before submission,
  and `SENT` right after the 2xx. The only window where the outcome is unknown is between
  those two commits, and it always resolves to `UNCERTAIN`. So "never send twice" (G5) holds
  by construction, at the cost of occasionally asking you.
- **Filing is write-ahead too.** `messages.insert`/`import` has no idempotency key, so a lost
  response or a crash before `FILED` is committed could otherwise file a second copy.
  1. Before filing, commit `FILING` together with the current mailbox `historyId`.
  2. On recovery from `FILING`, or after an ambiguous error, **reconcile before retrying**.
     Look for a message whose `X-Mmail-Dispatch` header is ours **among all `messageAdded`
     entries since the recorded `historyId`** (`history.list`). That finds the copy
     whichever thread it landed in, including an unthreaded fallback whose response was
     lost. When `filingMode` is `threaded`, the draft's thread is checked first as a fast
     path (`threads.get` with metadata headers). Both work under `gmail.metadata`.
  3. If one is found, move to `FILED` with that message ID. If none is found, retry the
     insert **with the stored sent-copy bytes**, never the current draft, which may have been
     edited or deleted. If history has expired, so reconciliation can't be done, or the
     stored bytes are gone, alert rather than guess. A duplicate Sent copy is cosmetic, but
     it's still avoided.
- **Pre-send check:** before committing `SENDING`, re-fetch the draft and require
  `messageId == confirmed messageId`. If it differs, or the draft is gone, nothing has been
  stored or submitted yet:
  - This dispatch moves to the **terminal** `WITHDRAWN`. It's keyed to a draft version that
    no longer exists, so nothing could ever confirm it again, and it must not sit in a
    non-terminal state for the sweep to chase.
  - An edited draft is a **new version**. It's offered afresh, with a "Draft changed —
    review" notification, and gets its own dispatch (keyed by the new message ID) only when
    you confirm it.
- **Undo window:** confirming commits `CONFIRMED` together with `sendAfter` (now +
  `undo_seconds`) and a `confirmation` number. It then creates a Cloud Task for the
  attempt, named as described below.
  - The task handler acts only if the dispatch is still `CONFIRMED` with the same
    `confirmation` number. Stale tasks do nothing.
  - Undo moves the dispatch to `CANCELLED` and deletes the task. If the delete fails, the
    stale-task check covers it.
  - **Undo is a token action too** (`undo`, below). Its target is the `confirmation`
    generation it cancels. The confirming transaction mints it, and the "Sending… [Undo]"
    notification and the `/dispatch` response carry it. It acts only if the dispatch is
    still `CONFIRMED` at that generation, and it consumes its nonce in the same
    transaction. A retried or delayed Undo for confirmation N can't cancel a later
    reconfirmation N+1. It only returns the current state.
- **Cancelled isn't final.** Nothing was submitted, so a later confirmation of the same
  draft version (a new tap or click) moves the existing dispatch from `CANCELLED` back to
  `CONFIRMED` with a fresh undo window and the next `confirmation` number. No draft edit is
  needed. It's still one dispatch per draft version, so the uniqueness guard holds.
- **Every confirmation, and every other action on a dispatch, uses a server-issued,
  generation-bound, one-time token.** Clients never mint their own.
  - Notification actions carry one, bound to the identity the notification shows.
  - The PWA and extension fetch one for the identity currently selected
    (`GET /dispatch/{draftId}/token?identity=…`) when they show the Send button. They fetch
    a new one whenever the dispatch **or the selected identity** changes, so a token is
    only ever minted after the choice is made.
  - Each token binds `draftId`, `draftMessageId`, identity, the current `confirmation`
    generation, a server nonce and an expiry (§4.8).
  - **Each token is bound to one action and its target.** The action is one of `confirm`,
    `undo`, `retry`, `retry-rejected`, `dismiss-rejected`, `resend-uncertain`,
    `retry-filing` or `close`. The target is action-specific:
    - for `undo`, the `confirmation` generation it cancels;
    - for a retry, the attempt number it creates, plus the hash of the recipient set for
      `retry-rejected`;
    - for `dismiss-rejected`, that recipient-set hash. The server stores the action and target with the
    nonce, and each endpoint accepts only tokens for its own action and the current
    target. A mismatch is rejected without consuming the nonce. So when "Retry filing" and
    "retry rejected only" are both on offer, neither token can authorize the other.
  - **The server mints a token only for an identity the resolver could produce** (§4.4):
    a listed identity, or any address at the `catch_all` domain, such as
    `amazon@example.net`, which uses the default identity's display name.
  - **The identity comes only from the token.** `/dispatch` takes no separate identity
    field. The confirming transition records the token's identity on the dispatch, so a
    reconfirmation after Undo may pick a different one.

  `/dispatch` applies it in one transaction:
  - **Wrong generation:** if the token's generation isn't the dispatch's *current* one, the
    request only returns the current state. This covers anything issued before an Undo,
    from any client, however many clicks were in flight.
  - **Already used:** if the token's nonce is already consumed (a delayed duplicate, a
    retried call, a replayed token), it also only returns the current state.
  - **Otherwise** the transition happens and the nonce is consumed.

  Undo increments the generation. So **reconfirming a `CANCELLED` dispatch needs a token
  the server issued after the cancellation**: the Undo notification's fresh "Send anyway"
  action, or a new tap in a PWA or extension view that has fetched a new token.
- **Delivery and filing keep separate clocks.** A dispatch has independent **tracks**:
  - one **delivery track per attempt** (`CONFIRMED` → `SENDING` → result, with
    `RETRY_WAIT` between submissions, below);
  - one **filing track** (`PENDING` → `FILING` → `FILED` → `DONE`).

  Under `SENT_PARTIAL` both can be active at once, e.g. a rejected-only retry while filing
  is backing off. So each track has its own `nextStepAt`, `scheduleGen`, `lastEnqueuedAt`
  and lease. A transition on one track never writes another track's fields, so neither
  can move the other's deadline.
- **Safe retries wait outside `SENDING`.** A transient failure **before `DATA`**
  (connection error, 4xx at `MAIL`/`RCPT` for every recipient) means nothing was accepted.
  - The attempt commits `RETRY_WAIT` with the backed-off `nextStepAt` and **releases its
    lease**. Only `SENDING` with an expired lease means "may have been accepted"
    (→ `UNCERTAIN`). A parked retry is never mistaken for a crashed submission.
  - When due, `RETRY_WAIT` → `SENDING` (taking the lease again) re-submits the same stored
    MIME. Once its retry budget is spent, the attempt becomes `FAILED`.
- **Every transition persists its next step.** Each state commit also writes its track's
  `nextStepAt`, so a process can die between the commit and task creation without
  stranding the dispatch. So does every retry decision: a transient error commits a later
  `nextStepAt` for that track (the backoff) together with the re-enqueue.
- **Task names include a scheduling generation.** Each Cloud Task is named
  `dispatchId-track-state-scheduleGen`, where the track is `attempt-n` or `filing`.
  - The track's `scheduleGen` is incremented in the same transaction that decides to
    (re)enqueue.
  - Cloud Tasks refuses to reuse a deleted or completed task's name for up to ~24 hours.
    A fixed name could therefore leave a re-enqueue blocked after a task exhausts its
    retries or is deleted, stalling a `SENT`, `FILING` or `FILED` dispatch, and its
    sendable draft, for a day.
  - **Idempotency comes from the transactional guard, not the name.** Every handler
    checks its track's **lifecycle guard** (for an attempt: its state, attempt number and
    `confirmation`; for filing: the filing status), checks that the step is **due**
    (`now ≥` the track's `nextStepAt`), and takes the track's lease before acting.
    - `scheduleGen` exists only to make names unique. It is **not** part of the guard.
    - Any queued task whose lifecycle guard still matches may perform the step, however
      old its `scheduleGen`, **once the step is due**. The first to run does the work and
      moves the state on, and the rest find the state changed and do nothing.
    - A task that arrives before `nextStepAt` does nothing. So after a transient error has
      pushed `nextStepAt` out, an older duplicate can't run the step early and skip the
      backoff; the task enqueued for the new `nextStepAt`, or the sweep, runs it on time.
    - So a backlog in Cloud Tasks can delay a step but never starve it.
- **Recovery sweep** (Cloud Scheduler, every few minutes, in both modes). It enqueues the
  next step for **any non-terminal track whose `nextStepAt` is more than a minute
  overdue**:
  - `CONFIRMED` → submit (after `sendAfter`);
  - `RETRY_WAIT` → submit again;
  - filing steps are driven by the **filing status alone**:
    - **`PENDING`**, when delivery is `SENT`/`SENT_PARTIAL` → start `FILING`, recording
      `filingHistoryId` once. A delivered message never misses its Sent copy and draft
      deletion, so a leftover draft can't be sent again by hand.
    - **`FILING`** → reconcile against the *original* `filingHistoryId`, then file. It is
      never re-recorded on later runs.
    - **`FILED`** → delete the draft (retried).
    - **`DONE`** → nothing. A `SENT_PARTIAL` dispatch stays non-terminal while rejections
      are outstanding, but that never triggers filing again; only its retry actions remain;
    - **`UNFILED`** → nothing. Filing moves here when its retries are exhausted. The track
      is parked (no `nextStepAt`), the draft is kept (file first, then delete), and you're
      alerted.
      - **"Retry filing"** (a one-time token action like the others) moves it back to
        `FILING` with a fresh filing-track schedule (`nextStepAt` = now, next
        `scheduleGen`). It keeps the *original* `filingHistoryId`, so it still reconciles
        before inserting.
      - **"Close"** resolves the dispatch without a copy (see stored MIME above);
  - **any attempt** that is `UNCERTAIN` and due → re-run the provider-log lookup (§4.6);
  - **any attempt** that is `SENDING` with an expired lease → that attempt becomes
    `UNCERTAIN` and starts polling. This is checked per attempt, **before and independently of** the
    dispatch's aggregate delivery and filing status. A crashed rejected-only retry under a
    `SENT_PARTIAL` dispatch whose filing is already `DONE` is still caught, and can't hold
    its lease forever.

  It also raises the unresolved-dispatch alerts.
  - Each re-enqueue bumps that track's `scheduleGen`, so it never collides with a
    remembered task name. The lifecycle guard keeps the effect idempotent.
  - To avoid piling up duplicates during a backlog, the sweep re-enqueues a step only if
    the track's last enqueue (`lastEnqueuedAt`) is older than the task's dispatch deadline,
    or Cloud Tasks reports that task gone.

## 5. Data model

Firestore (Native mode), single user:

| Collection / doc | Fields |
|---|---|
| `sync/gmail` | `historyId`, `watchExpiration`, `lastReconcile` |
| `drafts/{draftId}` | `messageId`, `threadId`, `toSummary`, `subject`, `identity`, `identitySource`, `settledAt`, `notifiedAt`, `silenced` |
| `dispatches/{hash(draftId, draftMessageId)}` | `draftId`, `draftMessageId`, `identity`, `outboundMessageId`, `confirmation`, `consumedConfirmationIds`, `sendAfter`, `attempt` (per attempt: envelope, result, and its track's `nextStepAt`, `scheduleGen`, `lastEnqueuedAt` and lease), `filingTrack` (`nextStepAt`, `scheduleGen`, `lastEnqueuedAt`, lease), `acceptedRcpts`, `rejectedRcpts`, `dismissedRcpts`, `mimeObject` (Cloud Storage path of the encrypted outbound MIME, cleared on resolution), `provider`, `providerRef`, `delivery` (delivery status), `filing` (filing status), `draftKept`, `filingMode` (`threaded`/`unthreaded`), `error`, `sentCopyId`, `sentStrategy`, `sentLabelsOk`, `filingHistoryId`, timestamps. The ID is the uniqueness guard (§4.10). |
| `devices/{id}` | push subscription, token hash, name, lastSeen |
| `config/main` | identities, rules, policy, `settle_seconds`, `undo_seconds`, `sent_strategy`, `archive_on_send` |

**Message bodies are stored only as the encrypted outbound MIME of unresolved dispatches**
(Cloud Storage with a customer-managed key). They're kept until resolution however long that
takes, then deleted, with a lifecycle backstop that applies only to resolved objects
(§4.10). They're never stored in Firestore or logs. Logs carry IDs and states, never subjects
or addresses.

## 6. Deployment

- **Default: Google Cloud in `me@gmail.com`'s own project.**
  - Cloud Run (min instances 0), Pub/Sub, Cloud Tasks, Cloud Scheduler (3 jobs: watch renew,
    reconcile, daily health), Firestore, Secret Manager, and a Cloud Storage bucket with a
    Cloud KMS key for in-flight outbound MIME. Its lifecycle rule matches only the
    `resolved/` prefix.
  - For one user this all sits inside the free tiers (inferred; check the billing report after
    a month).
  - The PWA is served from the service's default `*.run.app` URL. Nothing here should
    register `example.net` with Google as an organisation (Workspace, Cloud Identity); see
    OPTIONS.md R6.
- **Stack: TypeScript end to end**, so the server, PWA and extension share `core` (MIME,
  identity, types).
  - `googleapis`
  - `nodemailer` (raw SMTP send with explicit envelope)
  - an RFC 5322 header/address parser (e.g. `postal-mime` or `mailparser` + `addressparser`)
  - `web-push`
  - `@inboxsdk/core`
- **Alternatives:**
  - *Cloudflare Workers* + D1 + Queues, with HTTP provider APIs. Fine, but Pub/Sub push and
    OAuth live more naturally in GCP.
  - *A home server or VPS* with SQLite and polling every 15–30 s. Simplest, but it's now a box
    to keep up.
  - *Apps Script*: no SMTP, minute-granularity triggers and runtime quotas. OK for a
    throwaway prototype, not for the real thing.
- **Proposed repo layout:**

  ```
  packages/core        MIME transform, identity resolver, shared types (+ fixtures)
  packages/server      Cloud Run service: /pubsub, /dispatch, /api, push, filer
  packages/web         PWA + service worker
  packages/extension   Gmail web extension (phase 3)
  tools/inventory      E0: Takeout mbox → CSV
  infra/               gcloud/terraform for the project, topic, queues, secrets
  ```

## 7. Testing

- **Unit tests for `core` against real fixtures** captured with `drafts.get raw` from both
  clients:
  - Android reply, web reply, new message, forward with attachments;
  - inline image, Bcc, non-ASCII display names (RFC 2047), group syntax;
  - long folded headers, an existing `Reply-To`.
- **Golden tests:**
  - body bytes are identical before and after the transform;
  - the envelope equals the parsed To ∪ Cc ∪ Bcc;
  - the outbound copy carries no `Bcc` or `X-Gm-*`;
  - the sent copy keeps `Bcc`;
  - a brand-new draft (no non-draft parent) is classified as new and filed without
    `threadId`, while a reply keeps its `threadId` and threading headers.
- **Integration**, with a second throwaway Gmail account and a test recipient mailbox:
  - DKIM/DMARC pass at the recipient;
  - recipient-side threading;
  - the sender-side sent copy lands in the right thread and in Sent;
  - the draft is deleted.
- **Chaos:**
  - kill the server between `SENT` and `FILED`, then restart, and confirm there's no
    duplicate send;
  - kill the server after the provider's 2xx but before `SENT` is committed, and confirm the
    dispatch ends in `UNCERTAIN` (or is resolved from the provider log), never resent;
  - edit the draft during the undo window;
  - replay the same action token, and tap Send twice, both after the first dispatch has
    reached `SENT`. Confirm both return the existing dispatch and nothing is resubmitted;
  - send to a mix of valid and rejected recipients. Confirm `SENT_PARTIAL` with the right
    accepted/rejected sets, and that "retry rejected only" reaches just those addresses;
  - drop Pub/Sub messages and confirm reconcile catches up.

## 8. Security and privacy

- **Google's warning** in its help article is about handing *third parties* your credentials
  or tokens. mmail is first-party: your project, your token, your provider account. Only you
  can trigger a send, and only for your own drafts.
- **Blast radius of a server compromise.** An attacker could:
  - **read any draft in full**, bodies and attachments included. `gmail.compose` allows
    `drafts.get(format=raw)`, which mmail itself uses (§4.5);
  - read the *headers* of every other message (`gmail.metadata`);
  - read the stored outbound MIME of unresolved dispatches, which is content it could already
    read as drafts;
  - insert messages into your mailbox;
  - send mail as `me@example.net` (provider credential) or as `me@gmail.com`
    (`gmail.compose`).

  It can't read the bodies of non-draft messages. The exception is the optional Sent fixer:
  its `gmail.modify` grant can read and trash any message, so that token lives in its own
  secret, readable only by the fixer job (§4.7). Mitigations:
  - narrow scopes;
  - Secret Manager with a dedicated service account;
  - no public endpoints except `/pubsub` (OIDC-verified), the authenticated API and the
    static PWA.
- **The endpoint can only send existing drafts.** `/dispatch` takes a draft ID, never message
  content. Combined with the HMAC action token and session auth, a stolen notification token
  can only send *that* draft version, as *that* identity, within 15 minutes, and at most once
  (§4.10).
- **Limits:** a rate limit (e.g. 30 dispatches/hour, 200/day) and a kill switch (a config
  flag, checked on every dispatch).
- **Provider credential:** a dedicated smtp2go/Forward Email credential for mmail, separate
  from today's Send-as credential (which we don't touch until January).
- **Privacy:**
  - the push payload is end-to-end encrypted by Web Push;
  - nothing else leaves Google Cloud except the outbound mail itself, which goes to the same
    provider as today.

## 9. Failure modes

| Failure | Detection | Behaviour |
|---|---|---|
| Push delayed or dropped | reconcile every 10 min; PWA pull-to-refresh | drafts get offered late, never lost |
| Watch expired | daily renew; health job alerts if expiry < 2 days | reconcile keeps working meanwhile |
| `historyId` too old (404) | API error | full resync from `drafts.list` |
| Refresh token revoked (password change) | token refresh fails | health alert via Web Push; re-auth link in PWA; Gmail's own Send still works |
| Provider down or rejecting before `DATA` | connection error, 4xx/5xx at `MAIL`/`RCPT` | automatic retries with backoff (attempt parked in `RETRY_WAIT`, lease released), then `FAILED`; draft kept; "Retry" / "Retry via secondary provider" start the next attempt of the same dispatch from the stored MIME, with no edit needed |
| Timeout after DATA | socket error | `UNCERTAIN`; never auto-retried; user checks recipient/provider log |
| Process dies mid-submission (the provider may have accepted) | `SENDING` lease expires with no result | `UNCERTAIN`; resolved from the provider's send log only by a record **uniquely attributable to the current attempt** (its `X-Mmail-Attempt` marker or provider tag, never Message-ID alone) that gives per-recipient results for that attempt's whole envelope (→ `SENT` or `SENT_PARTIAL`); otherwise you decide; never resent automatically |
| Insert fails, or its outcome is unknown (lost response, crash before `FILED`) | API error or a dispatch left in `FILING` | reconcile by `X-Mmail-Dispatch` (thread or history since the recorded `historyId`) before any retry (§4.10); then retry with backoff → filing `UNFILED` (`SENT_UNFILED`) + alert; "Retry filing" reactivates it; mail *was* sent |
| Copy filed but without `SENT` (Gmail behaviour changed, e.g. at the January cutover) | label read-back after filing | alert: re-run `mmail doctor` and re-pin `sent_strategy`; no automatic second filing |
| Draft delete fails after a successful send | API error | leftover-draft warning in the notification; retried on every reconcile |
| Draft edited after confirm | message ID mismatch | before send: this dispatch becomes `WITHDRAWN` (terminal) and the edited version is offered afresh, getting its own dispatch when confirmed; after send: draft kept |
| Repeated tap, lost `/dispatch` response, replayed token | dispatch for `(draftId, draftMessageId)` already exists | returns the existing dispatch and its state; never a second send |
| Some recipients rejected at `RCPT TO` | rejected set in the transport result | `SENT_PARTIAL`; you're told who and why; "retry rejected only"; never a whole-message resend |
| Half-written draft looks settled (you paused, or are offline mid-edit) | not armed (§4.3) | notification offers **Review…** only; no one-tap Send |
| Gmail app re-uploads a stale offline draft after we deleted it | the same draft content reappears | offered again with a "looks already sent" warning |
| Gmail Send pressed on a domain thread | `SENT` from `me@gmail.com` in a domain-resolved thread | heads-up push (Android); prevented by guard (web) |
| Sent copy doesn't thread | E3 / user report | file without `threadId`; still in Sent and search |
| Server down entirely | health check from outside (e.g. a free uptime monitor) | drafts just wait; Gmail's Send is the manual fallback |

## 10. Rejected: reverse-alias relay

An earlier draft had an optional relay here. It rewrote inbound mail so that Gmail's normal
Reply would route through a relay address that sends as the domain. It's rejected because it
puts a rewriter on the inbound critical path, where any fault means mail that silently never
arrives (OPTIONS.md R7, Option 3). It also only helps replies, not typed recipients. A
self-hosted version built to fail open is explored separately in [RELAY.md](RELAY.md).

## 11. Inventory tool (E0)

`tools/inventory` answers "how much does Option 0 (send from @gmail.com) actually hurt?"
using a Takeout export, with no OAuth needed.

- **Input:** Takeout mbox of the *Sent* label, and optionally *Inbox/All Mail* for List-Id
  and inbound volume; plus the identities config.
- **Output** `recipients.csv`, one row per recipient address:

  | Column | Meaning |
  |---|---|
  | `address`, `domain` | the recipient |
  | `sent_as_domain_count` | messages sent to them as the domain |
  | `sent_as_gmail_count` | messages sent to them as `me@gmail.com` |
  | `first`, `last` | date range |
  | `received_count` | inbound messages from them |
  | `only_knows_domain` | they have never seen `me@gmail.com` |
  | `flags` | `helpdesk`, `list`, `ingest`, `noreply` |

  Flags come from a domain/pattern list (zendesk.com, freshdesk.com, helpscoutapp.com,
  service-now.com, atlassian.net, `reply+…@github.com`, `List-Id`, kindle.com, tripit.com, …).
- **Summary:**
  - how many active correspondents (last 24 months) only know the domain;
  - how many are sender-matched systems you'd have to update;
  - a per-month volume of domain sends.

## 12. Plan

The January 2027 cutover date is fixed. Keep today's Send-as running in parallel until it
stops, and use mmail for real domain sends from M2 onward to burn it in while the fallback
still exists.

| Milestone | Target | Contents |
|---|---|---|
| **M0** | 2026-10-31 | E0 inventory; E2 Gmail-app IMAP test (the interim Android path); E3 API spikes to check assumptions A1–A7. |
| **M1** | 2026-11-21 | Prototype: CLI `mmail dispatch <draftId>` + `mmail doctor` (§4.7) + `core` transform + filer, run locally against real drafts; ntfy notification from a polling loop. Verifies threading, sent copy strategy, DKIM. |
| **M2** | 2026-12-12 | *If E2 failed:* Cloud Run + Pub/Sub + Firestore + Cloud Tasks; PWA with Web Push actions; identity resolver; undo; accidental-send heads-up; health alerts. Daily driver on Android. *If E2 passed:* web-only mode (§3) instead, plus the Sent fixer if E2 showed copies in the wrong label. |
| **M3** | 2027-01-09 | Gmail web extension (button + guard). Until then, desktop uses Thunderbird, Forward Email webmail, or (full mode) notifications and the PWA. |

Interim fallback for any slippage: FairEmail on Android and Thunderbird on desktop, both with a
`me@example.net` identity on smtp2go (OPTIONS.md workflows D/E).

## 13. Assumptions to verify (E3)

| # | Assumption | If false |
|---|---|---|
| A1 | Drafts written in the Gmail Android app reach the server within seconds of leaving compose. | Notify on the PWA instead; add a "sync now" hint. |
| A2 | `drafts.get(raw)` for a reply includes `In-Reply-To`/`References`. | Threading repair from the thread (§4.5 step 5). |
| A3 | At least one of S1–S3 (§4.7) files a `SENT` copy with the true From that threads and shows as "me". | S4 (Gmail From in the archive); S5 as a last resort. |
| A4 | The provider preserves our `Message-ID`. | Read back the provider's ID and use it for the sent copy; or switch provider. |
| A5 | `gmail.compose` + `gmail.metadata` together allow `drafts.get(raw)` and `threads.get(metadata)`. | Use `gmail.modify` instead (broader). |
| A6 | The InboxSDK draft ID equals the Gmail API draft ID, or can be mapped to it directly. | **The extension never dispatches by heuristic.** Thread, recipients, subject and timing can't prove identity: the clicked draft may not yet be visible to the API while another draft matches exactly. Without a direct ID mapping, the Send-as button opens the PWA's Review screen for that thread, where you see the full content and choose explicitly. The guard against Gmail's own Send still works. |
| A7 | After January 2027 Gmail still renders our inserted sent copies sensibly even though `me@example.net` is no longer a Send-as alias. | Re-run `mmail doctor`; switch to S4 if "me" or `SENT` breaks. |
| A8 | Gmail collapses inbound echoes (Cc to yourself, list copies) of a message whose `SENT` copy exists with the same Message-ID. | Echoes show as new mail; mmail flags them. |
