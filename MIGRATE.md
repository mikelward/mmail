# Moving mail off Gmail to another provider

Status: draft, 2026-10-15. Companion to [OPTIONS.md](OPTIONS.md) (Option 10).

Scope: only **mail** moves. The `me@gmail.com` Google Account stays for Calendar, Drive,
Photos, Gemini and Android, and work stays on Workspace Gmail. Custom-domain sending becomes
native again, because the new provider hosts `me@example.net`. The question is what that
costs you.

Confidence tags as in OPTIONS.md: **(doc)** vendor documentation, **(reported)** third-party
or user reports, **(untested)** inference. Several vendor sites could only be read through
search snippets.

## TL;DR

- **Easiest operationally:** no code, no workarounds; the provider just sends as your domain.
- **But it gives up most of what you listed:**
  - Gemini over your mail: Gemini only reads Gmail.
  - Calendar's auto-added flights and reservations.
  - Gmail-grade spam filtering: probably a step down, even at Fastmail.
  - One interface on the web: Gmail web can't show other accounts any more.
- **Only Fastmail fits your criteria at all.**
  - Proton: search limits, which matches your memory.
  - Outlook.com: worse spam filtering, and no new custom domains since 2023.
  - iCloud+: no Android app.
  - HEY and Tuta: no IMAP.
- **If you do move, use the hybrid** (§4): Fastmail as primary, with every inbound and sent
  message copied into Gmail, so Gemini and Gmail search still see your mail.
- **Cheap way to find out:** a two-week Fastmail trial on a secondary address before moving
  the primary domain.

## 1. Your criteria, provider by provider

✓ good · ~ partial · ✗ poor or missing

| Criterion | Gmail (today) | Fastmail | Proton | Outlook.com / M365 | iCloud+ |
|---|---|---|---|---|---|
| **Google Calendar integration** | ✓ native invites; auto-added events | ~ two-way sync of Google calendars; Google-organiser invites must be answered in Google; no auto-added events (doc) | ✗ read-only subscription or one-time import; no CalDAV (doc) | ~ the Outlook app can show a Google account; otherwise separate (untested) | ✗ Apple Calendar |
| **Gemini** | ✓ in Gmail, the Gemini app, Deep Research | ✗ no AI. Has an official MCP server for Claude/ChatGPT; Gemini's custom MCP connector (US only) might reach it (untested) | ✗ | ✗ Copilot instead | ✗ Apple Intelligence instead |
| **Spam filtering** | ✓ best in class | ~ score-based plus global Bayes, then personal Bayes after ~200 spam/200 ham trained. Reports vary from "on par, fewer false positives" to "more missed spam" | ~ mixed reports | ✗ your experience; ongoing complaints in 2026 | ~ (untested) |
| **Android** | ✓ | ~ a web-wrapper app with native push; or use the Gmail app over IMAP | ✓ own app; no IMAP, so it can't go in the Gmail app | ✓ | ✗ no app; IMAP in the Gmail app, or the web |
| **Web** | ✓ | ✓ fast | ✓ | ✓ | ~ icloud.com |
| **Speed and reliability** | ✓ | ✓ fast. Outages: a 19 h partial in 2023; a few 2–7 h incidents in 2024–26 (reported) | ~ slower; encryption overhead | ✓ | ~ |
| **Search** | ✓ | ✓ server-side, fast, operators, searches inside attachments | ✗ body search is opt-in with a local index per device; Android likely metadata-only (untested) | ~ | ~ |
| **One interface with work Gmail** | ✓ | Android ✓ via the Gmail app (without Gmail features); **web ✗** | ✗ | ✗ | Android ~ (IMAP); web ✗ |
| **Custom-domain sending** | ✗ after January 2027 | ✓ wildcard, any address | ✓ (paid) | ✗ no new domains since Nov 2023 | ✓ |
| **Price** | free / Google One | ~$60/yr | ~$48–60/yr | ~$100/yr | from ~$12/yr |

Also ruled out: **HEY** and **Tuta** (no IMAP, own apps only); **Zoho, Migadu, Purelymail**
(budget IMAP hosts with weaker apps and spam filtering; Migadu Micro allows only 20 outgoing
messages a day).

## 2. What you lose by moving mail out of Gmail

Even with the Google Account kept, these features read *Gmail's* mailbox, so they stop
seeing mail that lives elsewhere.

| Feature | Why it needs Gmail | Survives the hybrid (§4)? |
|---|---|---|
| Gemini in Gmail: Help me write, AI Overviews, AI Inbox, summaries | runs inside Gmail on Gmail mail (doc) | Summaries and search work on the copies (untested). You'd *write* in Fastmail, so Help me write is gone. |
| Gemini app, Personal Intelligence, Deep Research over your mail | reads Gmail through the Workspace connected app; no consumer connector for other mail (doc) | ✓ on the copies (untested) |
| **Calendar "Events from Gmail"** (flights, hotels, tickets auto-added) | Gmail only. Google lists "the email redirects from another email account" as a case where it doesn't work (doc) | ✗ probably. **It may not work for your domain mail even today**, since that is forwarded too. Check whether recent bookings made with `me@example.net` show up in your Calendar. |
| Wallet passes from Gmail, Maps reservations, package tracking, travel summary cards | Gmail smart features (doc) | untested on forwarded copies |
| Invitations from non-Google organisers | arrive as email at the new provider. Google-organiser invites to your alternate email still land in Google Calendar directly (doc) | You'd RSVP in Fastmail, or forward to Gmail |
| **Spam filtering** | Gmail's classifier at Gmail's scale | ✗ replaced by the provider's |
| Categories/tabs, Important, nudges, Smart Compose/Reply, Drive-link attachments, Meet links, Calendar/Keep/Tasks side panel | Gmail UI features | ✗ |
| One web interface for home and work | Gmail web can't add third-party accounts (doc) | ✗ two tabs or apps on desktop |
| One place for history | Old mail is either imported into the provider or left in Gmail | The hybrid keeps a full copy in Gmail |

**Android can still be one app.** The Gmail app can add Fastmail (or any IMAP provider) as
an "Other" account, up to 5 (doc). But with Gmailify gone, that account gets none of Gmail's
spam filtering, categories or smart features; it's a plain IMAP client inside Gmail's UI.

## 3. What you gain

- **Native custom-domain sending everywhere.** Any address at the domain, no Outbox, no
  extension, nothing to run. This is the "easiest" part, and it is real.
- **Independence from Gmail policy changes** for your domain mail.
- **Fastmail specifics:**
  - fast web UI and good search;
  - masked email aliases;
  - an Auto Bcc and forwarding toolkit (used in §4);
  - an official MCP server (since April 2026) if you want an AI assistant over your mail;
  - a Gmail importer that brings over labels and filters.

## 4. The hybrid: Fastmail primary, everything copied into Gmail

If you move, keep Gmail as a *shadow archive* so the Gemini and search half of your wish list
mostly survives.

- **Inbound:** a Fastmail rule sends a copy of everything to `me@gmail.com`, with SRS.
  Mail Fastmail classifies as spam isn't forwarded (doc).
- **Sent:** Fastmail's per-address **Auto Bcc** to `me@gmail.com`, optionally including mail
  sent from other clients (doc).
- **In Gmail:** a filter so the copies skip the inbox and get a `fastmail` label (and
  `fastmail/sent` for the Bcc'd ones). Gmail then isn't a second inbox to triage.
  - The sent copies arrive as *incoming* mail, because Gmail no longer knows
    `me@example.net` is yours.
- **What you get:** Gemini, Personal Intelligence and Deep Research over all your mail, plus
  Gmail search as a backup (untested, but these read the whole mailbox).
- **What you still lose:**
  - Calendar auto-events: probably, per the "redirects from another account" exclusion;
  - Help me write;
  - Gmail spam filtering;
  - one web UI.
- **Cost:** Fastmail ~$60/yr, plus the copies count against your Google storage.

## 5. Provider notes

- **Fastmail:**
  - Google Calendar sync caveats: invitations must be opened in Google, event attachments are
    stripped, and Fastmail recommends migrating calendars rather than syncing.
  - Google Contacts is a one-time import with no ongoing sync.
  - The Android app wraps the web app, with native push (FCM). A desktop app launched in
    October 2025.
- **Proton:**
  - Content search builds an encrypted local index per browser and device; advanced
    operators work on metadata only. That's what felt limiting.
  - The calendar has no CalDAV.
  - Bridge (IMAP) is desktop-only, so Android means Proton's app.
- **Outlook.com / M365:** no new custom domains for consumers since 30 November 2023, so it's
  effectively out regardless of spam filtering. Copilot in Outlook is the AI analogue.
- **iCloud+:** custom domains work, but on Android you're on IMAP with an app-specific
  password or the website, and custom-domain IMAP glitches are reported.

## 6. Verdict

- **Easiest, but not best for you.** Moving to Fastmail is the least engineering. It meets
  custom-domain sending, mobile, web, speed and search. It loses Gemini-in-Gmail and
  auto-events, is likely a step down on spam, and splits the desktop from work.
- **With the hybrid,** Gemini over your mail mostly survives, at the cost of a duplicate
  archive.
- **Compared with the plan** (Option 4 and/or the mmail Outbox): those keep every Gmail
  feature and the single interface, but need a 15-minute test and possibly some building.
- **Before deciding, try it cheaply.** A Fastmail trial with a *secondary* domain address,
  or a forwarded copy of your domain mail, used for two weeks. Watch spam, the Android app,
  and how often you miss Gemini. Meanwhile, check whether Calendar auto-events work today
  for bookings made with `me@example.net`; if they don't, moving loses nothing there.

## Sources

- Gmail and Gemini:
  - [Gemini in Gmail features](https://support.google.com/mail/answer/16831098)
  - [Gemini app with Gmail](https://support.google.com/gemini/answer/13695044)
  - [Personal Intelligence](https://support.google.com/gemini/answer/16598406)
  - [Gemini custom MCP connectors](https://support.google.com/gemini/answer/17209137)
  - [Smart features settings](https://support.google.com/mail/answer/10079371)
- Calendar:
  - [Events from Gmail](https://support.google.com/calendar/answer/6084018)
  - [Invitations to alternate addresses](https://support.google.com/calendar/answer/37135)
- [Third-party accounts in the Gmail app](https://support.google.com/mail/answer/6078445),
  [Gmailify & POP changes](https://support.google.com/mail/answer/16604719)
- Fastmail:
  - [pricing](https://www.fastmail.com/pricing/us/)
  - [spam filtering](https://www.fastmail.help/hc/en-us/articles/1500000278142)
  - [search](https://www.fastmail.help/hc/en-us/articles/360060591213)
  - [Google Calendar sync](https://www.fastmail.help/hc/en-us/articles/360058752754)
  - [Gmail import](https://www.fastmail.help/hc/en-us/articles/360058752414)
  - [Auto Bcc](https://www.fastmail.help/hc/en-us/articles/1500000280401)
  - [forwarding](https://www.fastmail.help/hc/en-us/articles/360058753434)
  - [MCP server](https://www.fastmail.com/blog/an-mcp-server-for-fastmail/)
  - [outage history (aggregator)](https://isdown.app/status/fastmail/outage-history)
- Fastmail user reports (Hacker News):
  - [30998400](https://news.ycombinator.com/item?id=30998400)
  - [41456641](https://news.ycombinator.com/item?id=41456641)
  - [29760616](https://news.ycombinator.com/item?id=29760616)
- Proton: [search](https://proton.me/support/search),
  [message content search](https://proton.me/support/search-message-content),
  [external calendars](https://proton.me/support/subscribe-to-external-calendar)
- [Outlook.com personalized domains](https://support.microsoft.com/office/e888d746-61e5-49e3-9bd1-94b88e9be988),
  [Copilot in Outlook](https://support.microsoft.com/en-us/office/35b88a54-fe01-49f5-8789-b74e239d8dc0)
