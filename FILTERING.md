# Filtering on top of Gmail (exploratory)

Status: exploratory, 2026-10-26. **Not part of the January plan** ([OPTIONS.md](OPTIONS.md),
[DESIGN.md](DESIGN.md)). This covers better filtering for the personal account
`me@gmail.com`: what Gmail's own filters can do when managed as code, and a **watcher** that
sits on top of Gmail after delivery and labels, archives or trashes mail. Political mail is
the worked example. Read it against OPTIONS.md **R7**: nothing invasive on a critical path.

## TL;DR

- **Two layers.**
  - **Layer 0:** Gmail's own filters, managed as code with gmailctl, plus catch-all aliases
    routed to plus addresses so filters can see them. Nothing to run.
  - **Layer 1:** a watcher on mmail's change feed (DESIGN.md §4.2) for what filters can't
    express: time, your history with a sender, any header, per-alias state.
- **The watcher stays off the inbound path.** It acts after Gmail has delivered the mail, so
  if it's down, mail stays in the Inbox unsorted (R7).
- **Political mail** carries legally required footers ("Paid for by …", the FEC "best
  efforts" notice). These make a high-precision native filter whose matches go to Trash.
  Keyword rules ("Trump", "Republican") go to a review label, not Trash. This needs no mmail
  code (§5).
- **No Google review is needed.** "Personal use (fewer than 100 users)" is an explicit
  exemption from OAuth verification (doc). Restricted-scope verification and the annual
  security assessment apply to apps offered to more people than that. That's what makes a
  public TripIt-style product hard, not a personal tool (§2).
- **The real cost of Layer 1 is access.** Moving mail needs `gmail.modify`, which can also
  read every message. It gets its own token, like the Sent fixer (DESIGN.md §4.7).
- **Order:** the political filter and the gmailctl port now; alias routing after F1; the
  watcher after M3.

## 1. What Gmail filters can and can't do

**Can** (doc, gmailctl):
- match `from`, `to`, `subject`, words anywhere in the message including the body, `list:`,
  `deliveredto:`, size, attachments, and any other search operator;
- skip the Inbox, mark read, star, apply a label, send to Trash, set "never spam", set
  importance, set a category, or forward;
- run on arrival, **before** the phone is notified, whether or not anything of ours is up.

**Can't:**
- **send mail to Spam.** The API takes `SPAM` only as "never mark as spam"
  (`removeLabelIds`). gmailctl notes that `markSpam: true` "is not supported by Gmail" (doc);
- act on **time** ("after 3 days", "if still unread");
- use **history** ("I've written to them", "I replied in this thread", "in my contacts");
- read **arbitrary headers** (`Authentication-Results`, `Auto-Submitted`, `X-Original-To`) or
  match a regex;
- keep **per-alias state** ("first mail to this alias from a new domain");
- re-run over existing mail automatically. gmailctl deliberately never does. A one-off
  XML import in Gmail's settings with "also apply to matching conversations" does.

## 2. Access and Google review

Checked 2026-10-26.

- **Personal use is exempt from verification.** Google lists "If the app is for your
  personal use (fewer than 100 users)" as needing no verification. Users "click through
  'unverified app' warning screens", and apps "will need to complete a verification if they
  want to grow their user base beyond 100" (doc).
- **The 100-user cap** counts "100 new users in total" and "cannot be reset or changed"
  (doc). One user is fine.
- **Publish "In production".** In Testing, "authorizations by a test user will expire seven
  days from the time of consent" (doc). gmailctl's README gives the same advice, and adds
  that Google "does not require verification for personal-use apps".
- **What the review is.** Apps that want restricted scopes for more than 100 users need
  verification plus an annual security assessment (doc).
  - The assessment (CASA) is now done by an authorized lab at an assurance level (AL1/AL2).
    Self-scans are no longer accepted (reported).
  - Reported costs range from hundreds to tens of thousands of dollars a year (reported).
  - **This is the TripIt problem:** a product that reads other people's Gmail. Reading your
    own mail is the exemption above.
- **Scope tiers.**
  - `gmail.labels` is non-sensitive (doc).
  - `gmail.modify`, `gmail.readonly`, `gmail.metadata`, `gmail.compose`, `gmail.insert` and
    `gmail.settings.basic` are restricted; `gmail.send` is sensitive (reported, from
    third-party scope tables; Google's scope page couldn't be fetched from here).
- **No scope moves mail without being able to read it.** `messages.modify` needs
  `gmail.modify` or full mail access. `gmail.labels` only edits label definitions. And
  `gmail.modify` also permits sending through Gmail, as `me@gmail.com` (doc, not re-fetched).
- **Token gotchas** (reported; same as DESIGN.md §4.1):
  - a password change revokes refresh tokens that carry Gmail scopes;
  - six months unused revokes a token;
  - 100 live refresh tokens per client per account, after which the oldest is silently
    revoked.
- **Apps Script** follows the same rules for your own script. Consumer quotas are 90 minutes
  a day of trigger runtime and 20,000 Gmail reads/writes a day (doc). That's fine for a
  5-minute trigger, tight for a 1-minute one.
- **Work account:** a Workspace admin decides which unverified apps may touch Gmail. Nothing
  here targets it.

## 3. Layer 0: Gmail's own filters, managed better

### 3.1 Filters as code

- [gmailctl](https://github.com/mbrt/gmailctl): Jsonnet config, `diff`, `apply`, and unit
  tests with message bodies. It uses your own OAuth client (`gmail.settings.basic`, plus
  `gmail.labels` if it manages labels).
- **Port:** run `gmailctl download`, then check that `gmailctl diff` is empty. Imported rules
  that come back with `isEscaped: true` are fine to leave.
- **Gotchas** (doc):
  - `apply` replaces *all* filters. Anything else that wants a native filter, such as the
    watcher's generated block lists (§4), must go through the same config as an imported
    `.libsonnet` file, never the API directly.
  - With label management on, removing the last filter that uses a label deletes the label.
  - The API allows one user label per filter.

### 3.2 Make catch-all aliases visible to filters

Forward Email supports regex aliases that substitute into the forwarding address. Its own
example is `/^(support|info)$/:user+$1@gmail.com` (doc). For the catch-all:

```
forward-email=/^(.+)$/:me+$1@gmail.com
```

- Mail to `amazon@example.net` arrives with `Delivered-To: me+amazon@gmail.com`. Then
  `deliveredto:me+amazon@gmail.com` matches even list mail and Bcc'd mail, where the alias
  isn't in `To`.
- It's a forwarding-address change at the provider, not a rewriter on the path (R7).
- **Check first (F1):**
  - exact aliases, especially the arming address `send@` (DESIGN.md §4.3), still win over
    the regex;
  - what a message sent to two aliases does;
  - the identity resolver (DESIGN.md §4.4) is unaffected. It reads `To`/`Cc`, which keep
    the domain address.
- Forward Email also adds `X-Original-To`, but its docs tie that to "IMAP and/or masked
  forwarding" (doc), so don't count on it for plain forwarding.

### 3.3 Burn an alias at the edge

Prefix the alias with `!` (quiet accept-and-drop, 250), `!!` (soft reject, 421) or `!!!`
(hard reject, 550) (doc). The API does the same with `is_enabled: false` and
`error_code_if_disabled` (doc), so a "Burn" button in the PWA is one call (§6). A hard reject
also tells well-run senders to drop the address.

## 4. Layer 1: the watcher

### 4.1 Where it sits

```
 mail ─► Forward Email ─► Gmail ─► native filters (Layer 0) ─► Inbox / labels / Trash
                                     │ users.watch (DESIGN.md §4.2, same feed)
                                     ▼
                              mmail change feed ──► Outbox (drafts, SENT)
                                     └────────────► watcher: messageAdded, not
                                                    DRAFT/SENT/SPAM ─► rules ─► messages.modify
```

- It's a second consumer of the existing unfiltered watch, which today drops "ordinary
  inbound mail". **Web-only mode** (DESIGN.md §3) removes the change feed, so the watcher
  would be the reason to keep it.
- **Native filters run first.** The watcher sees the labels they applied and can refine them:
  rescue, promote or classify.
- Rules can use a header predicate, a Gmail query, or a computed fact (correspondents,
  alias history, time).
  - **Query trick (untested, F6):** `messages.list(q="rfc822msgid:<id> (<query>)")` asks
    Gmail's own search whether one message matches. Rules can then share the Layer 0 query
    language, body text included, without fetching bodies. It needs `gmail.modify` or
    `gmail.readonly`, since `q` isn't allowed under `gmail.metadata`, and may lag
    indexing.

### 4.2 Ground rules

Mirroring G5 and the Sent fixer (DESIGN.md §4.7):

- **Reversible actions only:** labels, archive, read, star, Trash. Never hard delete. Never
  take anything out of Spam. Never change content (§7).
- **Act once per message and never fight the user.** Journal each action at
  `filtered/{messageId}`. If you move a message back, the watcher leaves it alone.
- **Every action is visible and undoable:** an `mmail/rule/<name>` label, plus a "what the
  watcher did today" list in the PWA with Undo.
- **Each rule climbs a ladder: shadow → label → archive → Trash.**
  - Shadow mode only adds `mmail/shadow/<rule>`.
  - A rule moves up a rung after a review period with no false positives.
  - The same ladder works for Layer 0 rules.
- **Brakes:** a rate limit on destructive actions (e.g. more than 20 trashed an hour pauses
  every rule that trashes, and alerts) and a kill switch.
- **Fail open:** if the watcher is down, mail stays where Gmail and the native filters put it.

### 4.3 The notification race

The Gmail app notifies when mail arrives, seconds before the watcher can act, so mail the
watcher archives still buzzes. Native filters don't have this problem. Options:

- **Accept it** for low-volume rules.
- **Push the rule down into Layer 0** where the filter language can express it. For example,
  the watcher writes a generated sender list that gmailctl applies.
- **Notify by label:**
  - turn off Inbox notifications on Android;
  - turn on notifications for a `notify` label, which a native filter applies broadly and
    the watcher adds when it holds something back.
  - This setup still exists in the 2025 app (reported).
  - Unknown: does a label added *after arrival* through the API notify? (F2)
  - It turns "mmail down" into "fewer notifications" for anything that depends on the
    watcher, so it needs a health alert.

### 4.4 Access split

Many useful features only read and notify. They fit the `gmail.metadata` token the Outbox
already holds. Only features that move mail need `gmail.modify`.

| Token | Can | Used for (§6) |
|---|---|---|
| Outbox, `gmail.metadata` (existing) | read headers, labels and history; no bodies, no search | alias leak alerts, VIP alerts, waiting-for-reply, filter suggestions, filter health, phishing alerts |
| **Watcher, `gmail.modify`** (new, own secret) | read everything, change labels, Trash; can also send as `me@gmail.com` through Gmail | anything that moves mail; body and query rules |

- The watcher holds **no SMTP credential and no `gmail.compose`**, so a buggy rule can never
  send as the domain. A buggy dispatch can never reshuffle the mailbox.
- Rules that need a model (§6) call one on Vertex AI **in the same project**. That keeps mail
  inside Google Cloud, as DESIGN.md §8 promises.

### 4.5 Prototype first

Before any of this goes into Cloud Run, an **Apps Script** with a 5-minute time trigger
(`GmailApp.search` → label) can test which watcher rules are worth having. It has no push,
needs nothing from the Cloud project, and fits the consumer quotas (§2).

## 5. Political mail

### 5.1 Signals

| Signal | Where | Precision | Notes |
|---|---|---|---|
| "best efforts to collect and report" / "best efforts to obtain" | footer | high | The FEC notice on contribution solicitations by political committees (doc). Wording varies, so include both forms. |
| "not authorized by any candidate" | footer | high | The FEC disclaimer wording for committees not authorized by a candidate (11 CFR 110.11; not re-checked here, since fec.gov was unreachable). |
| "paid for by" | footer | medium | Required on committees' bulk email: more than 500 substantially similar messages (doc). State rules vary. Also appears on **political ads inside news newsletters**. |
| "not deductible" / "not tax deductible" | footer | medium | IRC §6113 statement on fundraising by political organizations and 501(c)(4)/(5)/(6) groups (doc). Also unions, trade groups and chambers of commerce. |
| ActBlue, WinRed, Anedot | donation links, footer | high with "paid for by" | The main donation processors. Whether Gmail search matches a word that appears only in a link URL is untested (F5). |
| Trump, Republican(s), Democrat(s)/Democratic, GOP, MAGA, DNC/RNC/DCCC/NRCC/DSCC/NRSC | subject, sender name | low | Catches news and friends. Whether Gmail search matches word variants is untested, so list each form. |
| Fake `Re:`/`Fwd:` subjects, a sender name posing as a person | subject, From | — | A common dark pattern (reported, Princeton study). Don't exempt `Fwd:`. |

**DMARC and other authentication don't help:** this is authenticated bulk mail from the
committees' own domains.

### 5.2 Native rules (gmailctl, sketch)

```jsonnet
// Keep these even when they match: newsletters you read, committees you
// support (and their receipts).
local politicalAllow = {
  or: [
    { list: 'newsletter.example.org' },
    { from: 'committee-you-support.example' },
  ],
};

// Legally required fundraising footers (§5.1). High precision.
local politicalSolicitation = {
  or: [
    { has: 'best efforts to collect and report' },
    { has: 'best efforts to obtain' },
    { has: 'not authorized by any candidate' },
    {
      and: [
        { has: 'paid for by' },
        { or: [{ has: 'actblue' }, { has: 'winred' }, { has: 'anedot' },
               { has: 'not deductible' }] },
      ],
    },
  ],
};

// Keywords. Low precision: review, don't trash.
local politicalWords = {
  query: '(subject:(trump OR maga OR gop OR republican OR republicans OR democrat ' +
         'OR democrats OR democratic) OR from:(dnc OR rnc OR dccc OR nrcc OR dscc OR nrsc))',
};

{
  version: 'v1alpha3',
  rules: [
    {
      // Shadow mode: change archive to delete only after F4 passes.
      filter: { and: [politicalSolicitation, { not: politicalAllow }] },
      actions: { archive: true, labels: ['political/solicitation'] },
    },
    {
      filter: { and: [politicalWords, { has: 'unsubscribe' },
                      { not: politicalSolicitation }, { not: politicalAllow }] },
      actions: { archive: true, labels: ['political/maybe'] },
    },
  ],
  tests: [
    {
      name: 'a newsletter carrying a political ad is kept',
      messages: [{ lists: ['newsletter.example.org'],
                   body: 'Paid for by Example PAC. Donate via ActBlue.' }],
      actions: {},
    },
  ],
}
```

- **"Drop" means Trash.** It's recoverable for 30 days and searchable with
  `in:trash label:political/solicitation`. That's what R7 allows for background tidying.
- **The keyword rule needs a bulk signal** (`has: 'unsubscribe'`), so a friend writing about
  politics isn't caught. gmailctl tests skip raw `query` filters, so test that rule by hand
  (F4).
- **Before turning on Trash (F4):**
  - paste the solicitation query into Gmail search over All Mail and read years of matches;
  - then run both rules in shadow mode for two weeks.

### 5.3 What the watcher adds (optional)

- **Rescue:** a message the native rule trashed whose sender is someone you've written to
  goes back to the Inbox with `political/rescued`. This covers forwards from friends, which
  carry the footer.
- **Promote the maybes:** a model call on headers plus the first part of the text sorts
  `political/maybe` into Trash or the Inbox.
- **Learn senders:** a sending domain that hits the solicitation rule N times goes into a
  generated `political-senders.libsonnet` that gmailctl applies. Over time Layer 0 catches
  more on its own.
- **Burn the alias:** if political mail arrives at a catch-all alias you gave one campaign,
  offer **Burn** (§3.3). Committees share and rent lists; the Princeton study found
  addresses "disclosed between senders" (reported). So burning the alias beats unsubscribing
  from each sender.

### 5.4 Spam, unsubscribing and blocking

- **Spam instead of Trash:** native filters can't do it (§1). The watcher can add `SPAM`,
  which the labels guide lists as manually applicable (doc). Whether that trains Gmail like
  "Report spam" is undocumented (F7). Trash stays the default. You can still press Report
  spam yourself.
- **Unsubscribing helps less than it seems,** except on platforms that enforce it themselves,
  such as Substack (§5.5). CAN-SPAM covers commercial mail, so political mail needn't honor
  unsubscribes (reported, FTC via Vice). Gmail's one-click unsubscribe
  rule is written for commercial bulk mail too. Unsubscribe per sender only through Gmail's
  own Manage subscriptions (July 2025), and lean on the rules above.
- **Gmail's Block** sends a single address to Spam. Committees rotate addresses, so it's
  whack-a-mole.
- **Context:** Gmail's treatment of political mail is contested.
  - The FTC chairman warned Alphabet in August 2025 about "partisan" spam filtering.
  - Krebs reported that the difference tracked sending practices.
  - An earlier RNC lawsuit was dismissed.
  - Don't expect Gmail to get stricter; this rule is your own call.

### 5.5 Political newsletters on Substack

Politicians and campaigns also mail through Substack, sometimes from imported lists. A former
DNC chair mass-subscribed his old campaign list in 2025. Substack requires imported lists to
be people who "explicitly opted in", but says it "doesn't have a way to verify" how they
were collected (reported, Politico).

- **No FEC footer:** these are newsletters, not committee solicitations, so the §5.2
  solicitation rule misses them. The keyword rule catches some.
- **Unsubscribe works here.** Substack runs the switch itself, per publication, rather than
  relying on the sender's goodwill. One writer found that unsubscribed readers stay marked
  as unsubscribed in the database (reported), so a re-import shouldn't resubscribe you.
- **Report the publication to Substack** as well as unsubscribing. Its policy lets it
  restrict publishers that import readers who didn't opt in (reported).
- **A "missing" footer is often Gmail clipping.** Gmail cuts long messages at "[Message
  clipped] View entire message", which hides the footer (reported). Gmail's own Unsubscribe
  link by the sender's name works from the `List-Unsubscribe` header either way. Also, as
  above, CAN-SPAM probably doesn't cover non-commercial political newsletters.
- **Filter Substack as its own screener,** since you can't predict which publication comes
  next:

  ```jsonnet
  local substackRead = { or: [{ from: 'one-you-read@substack.com' }] };
  // rule:
  {
    filter: { and: [{ from: 'substack.com' }, { not: substackRead }] },
    actions: { archive: true, labels: ['substack'] },
  },
  ```

  Publications with a custom domain may send from it instead (untested). Check one
  message's `From` and `List-Id`.
- **Find who leaked the address:** check `To`/`Delivered-To`. With per-sender catch-all
  aliases (and F1), the alias names the source of the import, and §3.3 can burn it.

## 6. More ideas

| Idea | What it does | Needs | Notes |
|---|---|---|---|
| **Political mail** | §5 | Layer 0; watcher for rescue/promote | Start now |
| **Filter health** | Weekly: hits per rule (from labels), rules with no hits in 6 months, a sample of what went to Trash | metadata | Catches silent false positives in every layer |
| **Alias leak alerts** | Record sender domains per catch-all alias; on the first mail to an established alias from a new domain, push "amazon@ got mail from 3 new domains [Burn] [Allow]" | metadata + Forward Email API key | Best after F1 |
| **Waiting for reply** | Threads where you sent the last message (Outbox or Gmail) and nobody answered in N days → push, or a `waiting` label cleared on reply | metadata (modify for the label) | The Outbox already records sends |
| **Expiry** | Archive login codes after an hour, shipping notices after a week, unread newsletters after N days | modify | The time dimension filters lack |
| **VIP alerts** | mmail Web Push for chosen senders or words (school, bank fraud desk) whatever Gmail's notification settings | metadata | A deterministic "High priority only" |
| **Phishing alerts** | Display-name impersonation of contacts ("Mom" from a stranger's address), lookalike domains of correspondents, DMARC failure from domains you deal with → `suspicious` + push | metadata | Where authentication does help |
| **Filter suggestions** | From history: senders whose mail you always archive or trash unread → a proposed gmailctl rule in the weekly report | metadata (history `labelsRemoved`) | Learning without a model |
| **Screener** | Mail from a sender you've never corresponded with goes to `screener`, archived; approve or deny each sender once in the PWA; denials compile into a Layer 0 block list | modify | HEY-style; mind the notification race |
| **Daily digest** | Archive chosen bulk mail on arrival; insert one digest per day into the Inbox with links | modify + `gmail.insert` | Fewer pings |
| **Identity labels** | Label domain threads `mmail/as/<identity>` using the resolver (DESIGN.md §4.4) | modify | Shows the reply address on the phone |
| **Retired aliases** | Mail to a retired alias gets a label and a drafted "my address changed" reply, sent as the current identity after you confirm in the Outbox | modify + `gmail.compose` | A draft, never an auto-send (§7) |
| **Trips** (personal TripIt) | Parse confirmations (the schema.org reservations airlines and hotels embed for Gmail's cards, else a model) into a trip page and a Calendar or ICS feed; share with family | **No Gmail scope** if confirmations are forwarded to `trips@example.net`, by a Gmail filter or by hand. Forward Email posts the raw message and attachments to a signed webhook (doc). Scanning the mailbox instead needs `gmail.readonly` | Personal use needs no review either way (§2); Calendar's "Events from Gmail" already covers part |
| **Bills to Drive** | Save statement PDFs from chosen senders into a Drive folder; due dates to Calendar | modify + `drive.file` + Calendar | |
| **Unsubscribe assist** | After you trash a sender N times, offer one-click unsubscribe (RFC 8058) | modify | Gmail's Manage subscriptions covers the manual case |
| **Model triage** | Sort the `*/maybe` buckets and unknown bulk mail into your own categories; never trashes directly | modify + Vertex AI | Opt-in per rule; stays in Google Cloud |

**Do first:** political mail, filter health, alias leak alerts, waiting for reply, expiry.

## 7. Changing messages and sending

- **Don't change messages.** Gmail messages are immutable, so "changing" one means
  inserting a rewritten copy and trashing the original. The copy gets new IDs and loses the
  original's authentication results, and a crash between the two steps leaves a duplicate.
  Labels can carry any annotation.
- **Draft, don't send.** Auto-sending breaks G5 ("never send anything that isn't one of your
  drafts, confirmed by you"). A rule that wants to send creates a **draft** as the right
  identity: forward invoices to the accountant, or answer mail to a retired alias. The
  Outbox then offers it like any other (DESIGN.md §4.3). If a truly automatic send is ever
  wanted, it needs:
  - an explicit per-rule allowlist and fixed recipients;
  - a rate limit;
  - no replies to `Auto-Submitted` mail (RFC 3834).

## 8. Experiments

| ID | Experiment | Pass criteria |
|---|---|---|
| **F1** | Plus-address routing at Forward Email (§3.2) on a test alias, then the catch-all | `Delivered-To` carries the tag and `deliveredto:` matches. Exact aliases, including `send@`, still win. A message to two aliases behaves acceptably. The identity resolver still picks the alias. |
| **F2** | Does a label added via `messages.modify` after arrival fire a Gmail Android label notification? | Notification within a few seconds, or a clear "no". |
| **F3** | Watcher latency: arrival to action | P50/P95 recorded. Decides how much the notification race matters. |
| **F4** | Political precision: run the §5.2 queries in Gmail search over All Mail, then two weeks of shadow mode | No false positives in the solicitation rule before it trashes. |
| **F5** | Does Gmail search match a word that appears only in a link URL (`actblue` in an `href`)? | Decides whether link-only signals need the watcher. |
| **F6** | Per-message query trick (§4.1) on a just-arrived message | Matches within seconds, or a measured indexing lag. |
| **F7** | Adding `SPAM` via `messages.modify` | Where it lands, and whether later mail from that sender goes to Spam unaided. Low priority. |

## 9. Where this fits

- **Now, independent of January:** port filters to gmailctl; add the political rules in
  shadow mode; run F4. No mmail code.
- **After F1:** plus-address routing, then alias-based rules.
- **After M3:** the watcher. First the metadata-only features (they reuse the Outbox's
  token and feed), then a separate `gmail.modify` worker with its own secret, dry-run and
  brakes. Prototype rules in Apps Script before porting them.

## Sources

- Google OAuth:
  - [Exceptions to verification (personal use)](https://support.google.com/cloud/answer/13464323)
  - [Unverified apps, user cap, Testing status](https://support.google.com/cloud/answer/15549945)
  - [Restricted scope verification](https://support.google.com/cloud/answer/13464321)
  - [Unverified apps on Workspace accounts](https://support.google.com/a/answer/9352843)
- Gmail API:
  - [Managing filters](https://developers.google.com/workspace/gmail/api/guides/filter_settings)
  - [Managing labels](https://developers.google.com/workspace/gmail/api/guides/labels)
  - [Apps Script quotas](https://developers.google.com/apps-script/guides/services/quotas)
- Scope tiers and CASA (reported):
  - [EmailEngine scope reference](https://learn.emailengine.app/docs/accounts/gmail/gmail-api-scopes)
  - [Nylas scope reference](https://developer.nylas.com/docs/cookbook/use-cases/build/google-oauth-scopes/)
  - [Validic CASA FAQ](https://help.validic.com/space/VCS/5539037190)
  - [Agentic Fabriq on CASA](https://www.agenticfabriq.com/blog/google-oauth-verification-casa)
  - [Nango on refresh token revocation](https://nango.dev/blog/google-oauth-invalid-grant-token-has-been-expired-or-revoked)
- [gmailctl](https://github.com/mbrt/gmailctl)
- Forward Email:
  - [regex aliases](https://forwardemail.net/en/faq#do-you-support-regular-expressions-or-regex)
  - [disabling aliases](https://forwardemail.net/en/faq#can-i-disable-specific-aliases)
  - [API](https://forwardemail.net/en/email-api)
- Gmail app:
  - [label notifications (Computerworld, 2025)](https://www.computerworld.com/article/3831854/android-email-upgrade.html)
  - [label notifications (PCWorld)](https://www.pcworld.com/article/412173/how-to-get-notifications-of-specific-gmail-messages-on-your-android-phone.html)
  - [Manage subscriptions (9to5Google)](https://9to5google.com/2025/07/08/gmail-manage-subscriptions/)
- Political mail:
  - [FEC: notices on solicitations](https://www.fec.gov/help-candidates-and-committees/making-disbursements-pac/notices-required-nonconnected-solicitations/)
  - [FEC MUR 6799 (email disclaimers)](https://www.fec.gov/files/legal/murs/6799/15044381639.pdf)
  - [26 U.S.C. §6113](https://www.law.cornell.edu/uscode/text/26/6113)
  - [Princeton study of 2020 political email](https://bstewart.scholar.princeton.edu/publications/manipulative-tactics-are-norm-political-emails-evidence-100k-emails-2020-us)
  - [CAN-SPAM and political mail (Vice)](https://motherboard.vice.com/en/article/xywjx3/i-cant-unsubscribe-from-ted-cruzs-mailing-list-and-the-ftc-says-thats-ok)
  - [FTC letter (TechCrunch)](https://techcrunch.com/2025/08/31/ftc-chair-warns-alphabet-about-gmails-partisan-spam-filters)
  - [Krebs on the spam-filter dispute](https://krebsonsecurity.com/2025/09/gop-cries-censorship-over-spam-filters-that-work/)
- Substack:
  - [Politico via Yahoo: Jaime Harrison's Substack import](https://www.yahoo.com/news/articles/suddenly-got-added-jaime-harrison-142919005.html)
  - [How email list imports work](https://support.substack.com/hc/en-us/articles/360044079591-How-do-email-list-imports-work)
  - [A writer on imports and unsubscribed readers](https://unstackit.substack.com/p/importing-email-list-to-substack)
