# mmail: design

Status: draft, 2026-10-15. Read [OPTIONS.md](OPTIONS.md) first for why we build this. This
document covers **Option 2 (mmail Outbox)** in detail, plus the optional reverse-alias relay
(Option 3b) and the inventory tool (experiment E0).

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
  - never lose a draft.
- **G6** Cost ≈ $0/month and need no routine babysitting.

Non-goals:

- A mail client: no reading, searching or rendering mail.
- Changing inbound delivery (except in the optional relay, §10).
- Multi-user support, or publishing the OAuth app.
- Sending through Gmail's SMTP or API with a spoofed From. Gmail rewrites a non-alias From
  to the account address (reported), so it can't be the transport anyway.

## 2. User experience

### 2.1 Android: reply as the domain

1. Open the thread in the Gmail app and tap **Reply**. Write it, optionally with Gemini.
2. Instead of tapping Gmail's Send, tap **←** (back). Gmail shows "Draft saved".
3. Within a few seconds a notification appears:

   > **Send as me@example.net?**
   > To: Alice Smith (+1) · Re: Dinner on Saturday
   > [ **Send** ] [ More… ]

4. Tap **Send**. The notification becomes "Sending as me@example.net… [Undo]" for 10 seconds
   (configurable), then "Sent ✓".
5. Back in Gmail, the draft is gone and the reply sits in the thread as sent by
   `me@example.net`.

*More…* opens the mmail PWA on that draft. There you can see the recipients, subject and a
text preview, switch identity (`me@example.net`, `shop@example.net`, …, or "leave it, I'll
send from Gmail"), and send.

### 2.2 Android: new message

Same flow. Compose a new message in Gmail, go back, tap the notification. The identity comes
from recipient rules or the default (§4.4). By default mmail only offers notifications for
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
  `me@gmail.com` in a thread whose identity resolves to the domain, and it pushes a heads-up
  so you can follow up.
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
  | `gmail.metadata` | restricted | `users.watch`, `history.list`, `threads.get(format=metadata)` for identity and threading headers |
  | `gmail.insert` | restricted | `messages.insert` for the sent copy |
  | `gmail.labels` | non-sensitive | create the `mmail/…` audit labels |

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
```

Resolution order:

1. **Explicit choice**: from the PWA, the extension, or the action token.
2. **Thread continuity**: walk the thread newest-first (`threads.get format=metadata`, headers
   only).
   - If one of *our* earlier messages was `From:` a domain identity, use that identity.
   - Otherwise take the first address in `To`/`Cc`/`Delivered-To` of inbound messages that
     matches an identity or the catch-all domain. With catch-all, that's the exact address the
     sender used, e.g. `amazon@example.net`.
   - Forward Email forwarding keeps the original `To`.
3. **Recipient rules.**
4. **Fallback**: `ask` shows the notification with no preselection (only *More…*); `default`
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
5. **Threading repair.** If `threadId` is set but `In-Reply-To`/`References` are missing (E3a
   checks whether Gmail drafts carry them):
   - find the newest non-draft message in the thread;
   - set `In-Reply-To` to its Message-ID;
   - set `References` to its References plus its Message-ID.
6. **Envelope:**
   - `MAIL FROM` is the identity address. The provider rewrites Return-Path to its bounce
     domain or our custom return-path CNAME.
   - `RCPT TO` is the de-duplicated set of addresses in `To` + `Cc` + `Bcc`. Group syntax and
     encoded words are handled by the parser.
7. **Limits:** refuse anything over 25 MB. Gmail already converts large attachments to Drive
   links.
8. **Outputs:**
   - the *outbound message*: the headers above plus the body;
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
- **Result handling:**
  - 2xx → `SENT`.
  - 5xx before `DATA` → `FAILED`, and the draft is kept.
  - Timeout or connection loss after `DATA` → **`UNCERTAIN`**. Never auto-retry; ask the user,
    because a duplicate is worse than a delay.
- **DNS:** unchanged from today's Send-as setup: provider DKIM, and SPF or a custom
  return-path. Once mmail is the only outbound path, consider moving DMARC to `p=quarantine`.
- **Provider limits:** smtp2go free is 1,000/month and 200/day (25/hour until the domain is
  verified). Forward Email starts at 300/day (doc).

### 4.7 Sent-copy filer

- **Insert:** `messages.insert` with the sent-copy bytes, `threadId` (when the draft had one),
  `internalDateSource=dateHeader`, and `labelIds: [SENT, <mmail/sent>]`.
  - Gmail threads it if the threadId is given, `In-Reply-To`/`References` are valid, and the
    Subject matches (doc).
  - Gmail only auto-applies `SENT` when the From is the user's own address, so we pass the
    label explicitly. Whether `insert` honours an explicit `SENT` is verified in E3b.
  - If it doesn't, fall back to IMAP APPEND into `[Gmail]/Sent Mail` (filer-only, app
    password), or keep `mmail/sent` as the visible label.
- **Delete:** then `drafts.delete(draftId)`, but only if the draft's message ID still equals
  the one we sent. If you edited the draft after confirming, it is kept and you're told.

### 4.8 Notifier and PWA

- **Web Push (VAPID)** from the server to the PWA's service worker, which works in Android
  Chrome and desktop Chrome.
  - The payload holds `draftId`, identity, a recipient summary, the subject, and a signed
    **action token**: `HMAC(key, draftId | draftMessageId | identity | expiry)`, valid for
    15 minutes.
  - Actions: **Send** (posts the token to `/dispatch` in the background) and **More…** (opens
    the PWA). Android shows two action buttons comfortably.
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
  2. `POST /dispatch {draftId, identity}` with the device token;
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

```
OFFERED ──confirm──► CONFIRMED ──undo window──► SENDING ──2xx──► SENT ──insert──► FILED ──delete──► DONE
   │                    │ undo                     │ 5xx            │ insert fails         │ draft edited
   ▼                    ▼                          ▼                ▼ (retry w/ backoff)   ▼
WITHDRAWN           CANCELLED                   FAILED          SENT_UNFILED          DONE_KEPT_DRAFT
                                                   │ timeout after DATA
                                                   ▼
                                               UNCERTAIN (user decides)
```

- **Lease:** each transition is a Firestore transaction. `SENDING` takes a lease keyed by
  `draftId`, so two taps (notification plus extension) can't both send.
- **Pre-send check:** at the start of `SENDING`, re-fetch the draft and require
  `messageId == confirmed messageId`. Otherwise go back to `OFFERED` and re-notify with "Draft
  changed — review".
- **Undo window:** a Cloud Task scheduled `undo_seconds` ahead. Undo deletes the task.

## 5. Data model

Firestore (Native mode), single user:

| Collection / doc | Fields |
|---|---|
| `sync/gmail` | `historyId`, `watchExpiration`, `lastReconcile` |
| `drafts/{draftId}` | `messageId`, `threadId`, `toSummary`, `subject`, `identity`, `identitySource`, `settledAt`, `notifiedAt`, `silenced` |
| `dispatches/{id}` | `draftId`, `draftMessageId`, `identity`, `outboundMessageId`, `rcptCount`, `provider`, `providerRef`, `state`, `error`, `sentCopyId`, timestamps |
| `devices/{id}` | push subscription, token hash, name, lastSeen |
| `config/main` | identities, rules, policy, `settle_seconds`, `undo_seconds` |

**No message bodies are stored. Logs carry IDs and states, never subjects or addresses.**

## 6. Deployment

- **Default: Google Cloud in `me@gmail.com`'s own project.**
  - Cloud Run (min instances 0), Pub/Sub, Cloud Tasks, Cloud Scheduler (3 jobs: watch renew,
    reconcile, daily health), Firestore and Secret Manager.
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
  packages/relay       optional reverse-alias relay (phase 4)
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
  - the sent copy keeps `Bcc`.
- **Integration**, with a second throwaway Gmail account and a test recipient mailbox:
  - DKIM/DMARC pass at the recipient;
  - recipient-side threading;
  - the sender-side sent copy lands in the right thread and in Sent;
  - the draft is deleted.
- **Chaos:**
  - kill the server between `SENT` and `FILED`, then restart, and confirm there's no
    duplicate send;
  - edit the draft during the undo window;
  - drop Pub/Sub messages and confirm reconcile catches up.

## 8. Security and privacy

- **Google's warning** in its help article is about handing *third parties* your credentials
  or tokens. mmail is first-party: your project, your token, your provider account. Only you
  can trigger a send, and only for your own drafts.
- **Blast radius of a server compromise:** read your drafts and message *headers*, insert
  messages, and send mail as `me@example.net` (provider credential) or as `me@gmail.com`
  (`gmail.compose`). It can't read message bodies. Mitigations:
  - narrow scopes;
  - Secret Manager with a dedicated service account;
  - no public endpoints except `/pubsub` (OIDC-verified), the authenticated API and the
    static PWA.
- **The endpoint can only send existing drafts.** `/dispatch` takes a draft ID, never message
  content. Combined with the HMAC action token and session auth, a stolen notification token
  can only send *that* draft version, as *that* identity, within 15 minutes.
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
| Provider down or rejecting | 4xx/5xx | `FAILED`; draft kept; "Retry" / "Retry via secondary provider" |
| Timeout after DATA | socket error | `UNCERTAIN`; never auto-retried; user checks recipient/provider log |
| Insert fails | API error | retry with backoff → `SENT_UNFILED` + alert; mail *was* sent |
| Draft edited after confirm | message ID mismatch | send is blocked and re-offered; if after send, draft kept |
| Gmail app re-uploads a stale offline draft after we deleted it | the same draft content reappears | offered again with a "looks already sent" warning |
| Gmail Send pressed on a domain thread | `SENT` from `me@gmail.com` in a domain-resolved thread | heads-up push (Android); prevented by guard (web) |
| Sent copy doesn't thread | E3 / user report | file without `threadId`; still in Sent and search |
| Server down entirely | health check from outside (e.g. a free uptime monitor) | drafts just wait; Gmail's Send is the manual fallback |

## 10. Optional phase 4: reverse-alias relay

This gives zero-tap replies through Gmail's native Send. It's built on Forward Email
features we already pay for, with no MX change. Trial a bought one (addy.io) first; see
OPTIONS.md E4.

### Address scheme

Use a dedicated subdomain `r.example.net`, with MX at Forward Email and a catch-all regex alias
that delivers to a webhook. Addresses are readable, e.g. `alice=example.org@r.example.net`,
optionally prefixed with an identity, e.g. `shop+alice=example.org@r.example.net`. That lets
you type new contacts by hand.

### Outbound (Gmail → relay → world)

1. Forward Email POSTs the raw message plus its `dkim`/`spf`/`dmarc` results to the relay,
   with `X-Webhook-Signature` (doc).
2. The relay verifies:
   - the webhook signature;
   - DKIM pass with `d=gmail.com`;
   - `From` exactly `me@gmail.com`;
   - `h=` covers `to` and `cc`;
   - the DKIM `t=` is recent;
   - the Message-ID hasn't been seen before.

   It takes recipients **from the signed headers**, not from RCPT. Don't rely on gmail.com's
   DMARC policy, which is `p=none`.
3. It rewrites `From` to the identity and maps relay addresses back to real addresses. It
   strips Gmail headers, keeps `In-Reply-To`/`References`, sets a fresh Message-ID, and
   submits via the transport (§4.6).
4. Any To/Cc address in the Gmail copy that *wasn't* a relay address already received the
   message straight from `me@gmail.com`. The relay can't prevent that, so it alerts.

### Inbound (world → relay → Gmail)

1. `*@example.net` routes to the relay webhook instead of being forwarded to Gmail.
2. If the original sender authenticates (DMARC pass or aligned DKIM pass), the relay rewrites
   the other parties in `From`, `To`, `Cc` and `Reply-To` into relay addresses, keeping
   display names, and adds `X-Mmail-Original-*` headers.
3. It delivers through Gmail API `messages.import`, which applies Gmail's normal spam
   classification (but no SPF check), with `processForCalendar=true`.
4. Unauthenticated mail is imported *unmodified* with an `mmail/unverified` label, so it has no
   relay reply path.

**Costs:**

- the relay is now in the **inbound critical path**. If it's down, Forward Email retries and
  then defers with 421, and senders' servers keep retrying;
- Gmail's view of senders degrades (original DKIM is gone, no avatars or unsubscribe);
- autocomplete fills up with relay addresses;
- mmail Outbox's identity resolver must map relay addresses back.

**Pilot it on a secondary address before the primary.**

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
| **M1** | 2026-11-21 | Prototype: CLI `mmail dispatch <draftId>` + `core` transform + filer, run locally against real drafts; ntfy notification from a polling loop. Verifies threading, sent copy, DKIM. |
| **M2** | 2026-12-12 | Cloud Run + Pub/Sub + Firestore + Cloud Tasks; PWA with Web Push actions; identity resolver; undo; accidental-send heads-up; health alerts. Daily driver on Android. |
| **M3** | 2027-01-09 | Gmail web extension (button + guard). Until then, desktop uses notifications or the PWA. |
| **M4** | 2027 Q1, optional | Relay pilot on a secondary address (or adopt addy.io). |

Interim fallback for any slippage: FairEmail on Android and Thunderbird on desktop, both with a
`me@example.net` identity on smtp2go (OPTIONS.md workflows D/E).

## 13. Assumptions to verify (E3)

| # | Assumption | If false |
|---|---|---|
| A1 | Drafts written in the Gmail Android app reach the server within seconds of leaving compose. | Notify on the PWA instead; add a "sync now" hint. |
| A2 | `drafts.get(raw)` for a reply includes `In-Reply-To`/`References`. | Threading repair from the thread (§4.5 step 5). |
| A3 | `messages.insert` accepts `labelIds: [SENT]` with a non-alias From, and threads with `threadId`. | IMAP APPEND to `[Gmail]/Sent Mail`, or the `mmail/sent` label only. |
| A4 | The provider preserves our `Message-ID`. | Read back the provider's ID and use it for the sent copy; or switch provider. |
| A5 | `gmail.compose` + `gmail.metadata` together allow `drafts.get(raw)` and `threads.get(metadata)`. | Use `gmail.modify` instead (broader). |
| A6 | The InboxSDK draft ID equals the Gmail API draft ID. | Look the draft up by thread + recipients + recency. |
| A7 | After January 2027 Gmail still renders our inserted sent copies sensibly even though `me@example.net` is no longer a Send-as alias. | Cosmetic; accept. |
