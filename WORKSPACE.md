# Google Workspace for personal use: pros and cons

Status: draft, 2026-10-15. Companion to [OPTIONS.md](OPTIONS.md) (Option 1).

Workspace is the one Google-supported way to keep sending as `me@example.net` from Gmail
after January 2027: Workspace "Send as" is explicitly unaffected. OPTIONS.md rejects
a Workspace seat *alongside* `me@gmail.com`, because it splits your Google identity (R6). This
doc looks at the bigger moves that avoid the split by making a Workspace account your *one*
Google identity, and at what that costs in consumer features.

Tags: **(doc)** Google documentation; **(doc, snippet)** Google documentation seen only via
search, because Workspace admin help pages were unreachable from the research sandbox;
**(reported)**; **(untested)**.

## TL;DR

- **What Workspace gets you:**
  - native custom-domain sending in the Gmail app and web, with aliases and catch-all;
  - all of Gmail's spam filtering and search;
  - Gemini in Gmail, with business data terms;
  - the same apps as work;
  - your domain address becomes your Google identity for Calendar and Drive.
- **What it costs: a Workspace account is a second-class consumer account.**
  - **Family:** it can't create, join or manage a Google family group, or supervise a child
    with Family Link (doc).
  - **Subscriptions:** no Google One or AI Pro, no YouTube Premium (individual or family), no
    Play Pass (doc). Play purchases can't move between accounts (doc).
  - **Devices:** no Fitbit / Google Health linking, and no Android backup and restore for
    managed accounts (doc).
  - **Home and phone:** Google Home works with limits, and the free Google Voice plan isn't
    available (doc).
  - **Gemini:** Business Starter is roughly the *free* consumer tier with business data
    terms. No Personal Intelligence, and no connected apps for Home, Maps or YouTube Music
    (doc).
  - **Lifecycle:** miss payment for 30 days and the account is suspended; after 60 days you
    "might lose associated user data" (doc, snippet). Admin recovery depends on keeping the
    domain.
- **Verdict:** if you manage a family group or supervise a child, the family restriction
  alone means keeping a personal account for that. That brings back the two-identity split
  Workspace was meant to avoid, so Workspace isn't a fit.

## 1. Variants

| Variant | What | One identity (R6)? | Notes |
|---|---|---|---|
| **1a Send-as seat** | Workspace user `me@example.net`; consumer Gmail sends as it | ✗ | Invites and shares to the domain go to the Workspace account. Rejected in OPTIONS.md. |
| **1b New Workspace account as main** | Create `me@example.net` on Workspace, migrate, retire `me@gmail.com` over time | ✗ for years, maybe forever | Family, Play purchases and Google One can't follow, so `me@gmail.com` lives on. |
| **1c Convert the existing account** | Buy Workspace *on* `me@gmail.com` and verify `example.net` on it (doc, snippet) | ✓ | "All of your emails… Drive files and other data… are retained." The Gmail address keeps receiving. You "can no longer use Google Family, Google Fit, or Google Nest." Effect on Photos, Play, YouTube and Google One is undocumented; Google's own pages conflict on whether it can be undone. |
| **1d Hybrid** | Workspace for mail, Calendar and Drive; `me@gmail.com` for family, purchases, Fitbit and Home | ✗ by design | Two accounts in every app. |

1c is the only variant that keeps a single identity, but it's also the one that switches off
family and the other consumer features *on your main account*, possibly irreversibly.

## 2. Pros

- **Solves the original problem natively.** "Send as" for Workspace addresses and aliases
  keeps working everywhere: up to 30 aliases, domain aliases, catch-all routing. No code, no
  workaround.
- **It's still Gmail:** spam filtering, search, speed, the Android app and web, Calendar
  integration including Events from Gmail.
- **One interface with work.** Both accounts are Workspace Gmail in the same apps.
- **Identity on your domain.** Calendar invites and Drive shares follow `me@example.net`,
  which you control, rather than a Gmail address.
- **Gemini data terms.** Chats aren't human-reviewed or used for training (doc).
- **Photos retention.** Workspace photos aren't deleted for inactivity (doc).
- **Admin control.** You set the policies yourself, including the "Additional Google services"
  switches.

## 3. Cons: consumer features on a Workspace account

| Feature | On a Workspace account | Source |
|---|---|---|
| **Family group** (manager, member, sharing Google One/AI plan, Play Family Library, YouTube family, family calendar) | ✗ can't create, join or be added | (doc) [families 7103337](https://support.google.com/families/answer/7103337), [6317858](https://support.google.com/families/answer/6317858), [Play 7007852](https://support.google.com/googleplay/answer/7007852) |
| **Family Link parent/supervisor** | ✗ "cannot be used to… manage supervision via Family Link" | (doc, snippet) [Family Link FAQ](https://families.google/familylink/faq/) |
| **Google One / AI Pro / AI Ultra** | ✗ "Workspace accounts are ineligible for Google One" | (doc) [googleone 9080668](https://support.google.com/googleone/answer/9080668) |
| What that costs: AI Pro's bundle (5–10 TB, YouTube Premium Lite, Google Home Premium, Google Health Premium, Flow, Jules, family sharing) | ✗ | (doc) [googleone 14534406](https://support.google.com/googleone/answer/14534406) |
| **YouTube Premium** | ✗ individual or family (student is allowed) | (doc) [youtube 13187688](https://support.google.com/youtube/answer/13187688) |
| **Play Pass** | ✗ | (doc) [googleplay 9473027](https://support.google.com/googleplay/answer/9473027) |
| **Play purchases** (apps, movies, books) | ✗ can't be transferred between accounts | (doc) [googleplay 14274288](https://support.google.com/googleplay/answer/14274288) |
| **Fitbit / Google Health** | ✗ "You can't link your Google Health account to a Google Workspace account" | (doc) [googlehealth 14237024](https://support.google.com/googlehealth/answer/14237024) |
| **Pixel Watch** | ~ only with Device Management off; Gemini on the watch is personal-only | (doc) [12651780](https://support.google.com/googlepixelwatch/answer/12651780), [16393978](https://support.google.com/googlehealth/answer/16393978) |
| **Google Home / Nest** | ~ once the admin enables it. No personal results, calls to contacts, personal music or Sleep Sensing. Docs conflict on inviting members. | (doc) [googlehome 7571892](https://support.google.com/googlehome/answer/7571892), [9155535](https://support.google.com/googlehome/answer/9155535) |
| **Android backup and restore** | ✗ "not available for managed Google accounts on Android devices" | (doc) [work/android 10384040](https://support.google.com/work/android/answer/10384040) |
| **Google Voice** | ✗ free and standalone plans; ~ paid Workspace add-on ($10–30/user) | (doc) [voice 9249103](https://support.google.com/voice/answer/9249103) |
| **Google Fi** | ~ after the admin enables Fi and Payments | (doc) [fi 6201840](https://support.google.com/fi/answer/6201840) |
| **Wallet tap-to-pay** | ? Google Pay is an admin switch; tap-to-pay unconfirmed | (untested) |
| **Photos** | ~ admin switch; partner sharing undocumented | (doc) [photos 10100180](https://support.google.com/photos/answer/10100180) |
| **YouTube** | ~ admin switch; new accounts restricted at first | (doc) [youtube 1074205](https://support.google.com/youtube/answer/1074205) |
| **Maps Timeline / Location Sharing** | ~ admin switch; sharing "may not be available" | (doc) [accounts 4388034](https://support.google.com/accounts/answer/4388034), [maps 7326816](https://support.google.com/maps/answer/7326816) |
| **Maps reservations / Wallet passes from Gmail; Personal Intelligence** | ✗ "smart features in other Google products" are personal-account only | (doc) [mail 10079371](https://support.google.com/mail/answer/10079371), [gemini 16598406](https://support.google.com/gemini/answer/16598406) |
| **Chromebook** | ~ Play Store, Android apps and Phone Hub may be blocked by policy, which you'd set yourself | (doc) [chromebook 7021273](https://support.google.com/chromebook/answer/7021273) |

Services switched on under "Additional Google services" (Photos, YouTube, Play, Google Pay,
Takeout, Timeline, Home…) aren't covered by Workspace support or the SLA (doc, snippet).

## 4. Gemini: Business Starter vs consumer plans

| | Consumer free | Consumer AI Pro (~$19.99/mo, US) | Workspace Business Starter | Business Standard |
|---|---|---|---|---|
| Gemini app models | standard limits, 32K context | 4× limits, 1M context | "basic access" to Pro and Thinking, 32K context | Pro 25 prompts/4 h, Thinking 300/day, 1M context |
| Deep Research | limited | higher | 5 per month | 20 per day |
| Video generation, scheduled actions | limited / ✓ | ✓ | ✗ | ~ (3 videos/day) |
| Gemini in Gmail | some features | full, including AI Inbox | Help me write, proofread, suggested replies, summaries; **no AI Inbox** | more |
| Personal Intelligence; Tasks in Gemini | ✓ | ✓ | ✗ | ✗ |
| Connected apps | Google Home, Maps, YouTube Music, Spotify, WhatsApp, Workspace | same | Workspace apps, Classroom, GitHub only | same as Starter |
| Gemini Live connections | all | all | Calendar, Keep and Tasks only | same |
| Data terms | consumer | consumer | business: no human review, no training | business |

Sources:
- Gemini app limits by edition: [gemini 14620100](https://support.google.com/gemini/answer/14620100)
- consumer plan limits: [gemini 16275805](https://support.google.com/gemini/answer/16275805)
- Gemini in Gmail by edition: [mail 13952129](https://support.google.com/mail/answer/13952129?co=DASHER._Family%3DBusiness-Enterprise)
- connected apps: [gemini 14959807](https://support.google.com/gemini/answer/14959807) vs [13695044](https://support.google.com/gemini/answer/13695044)
- Gemini Live: [gemini 15274899](https://support.google.com/gemini/answer/15274899)

Also: converting an account that already pays for a Google One AI plan (variant 1c) "can
hide some benefits or lower Gemini limits", and Personal Intelligence turns off
(doc, snippet).

## 5. Account lifecycle and admin risk

- **Unpaid subscription.** 30 days unpaid leads to suspension. Subscriptions "suspended for
  more than 60 days might be cancelled, and you might lose associated user data" (doc,
  snippet). Compare a consumer account, which is deleted only after 2 years over quota,
  with 3 months' notice ([googleone 9312312](https://support.google.com/googleone/answer/9312312)).
- **Sole super admin.** Google recommends at least two super admins run by different people,
  plus a separate daily-use account (doc, snippet).
  - Self-recovery is on by default for most editions.
  - The last-resort recovery is proving ownership of the domain in DNS. **Lose the domain
    and you lose the account.**
- **Ownership and terms.** Domain-verified accounts are "organization owned", and Gemini use
  falls under the Workspace terms, not the consumer terms. No explicit ban on personal use was
  found.

## 6. Migration: what moves

For 1b, a separate new Workspace account:

| Data | Moves? |
|---|---|
| Gmail | ✓ Data migration service, one Gmail account per import (doc, snippet) |
| Calendar, Contacts | ~ ICS/CSV import. Ownership of events you organised doesn't transfer (doc) |
| Drive | ✗ "You can't transfer a file from your personal Google account to someone with a work or school account." Copies get new links, so collaborators must re-share (doc, [drive 2494892](https://support.google.com/drive/answer/2494892)) |
| Photos | ~ no official transfer; partner sharing with "save to your account" as a workaround (doc) |
| YouTube | ~ the channel moves via a Brand Account; history and subscriptions don't (doc / untested) |
| Play purchases, Google One, family | ✗ |

For 1c (converting `me@gmail.com`): data is retained under the domain address (doc, snippet).
Family, Fit and Nest stop. What happens to Photos, Play, YouTube and Google One isn't
documented, and reversibility is unclear.

## 7. The alternate-email trap

Verifying `example.net` in *any* Workspace or Cloud Identity tenant affects your current
personal account. If `me@example.net` is an alternate email on `me@gmail.com`, it is
**removed** from the personal account once a managed account claims it, and you're told at
next sign-in (doc, snippet). From then on, new invites and shares addressed to the domain go
to the Workspace account (untested, but that is the point of the claim). That's the R6 split
mechanism, and why OPTIONS.md says not to verify the domain even for a trial.

## 8. Pricing

| Plan | Annual | Flexible | Storage |
|---|---|---|---|
| Business Starter | $7/user/mo | $8.40 | 30 GB pooled per user |
| Business Standard | $14 | $16.80 | 2 TB pooled per user |
| New-customer promo, 28 Oct 2026 – 28 Jan 2027 | Starter $5.60, Standard $9.80 | | |
| Workspace Individual | sold on a personal account; 1 TB; **no custom domain** | | |

Compare consumer AI Pro at ~$19.99/mo with 5 TB, family sharing and the consumer Gemini
features.

## 9. Verdict

- **1a** and **1d** split your identity (R6).
- **1b** can't take family, purchases or Google One with it, so it ends up as 1d.
- **1c** keeps one identity but turns off family, Fit and Nest on your main account, likely
  costs Google One and AI Pro, and is possibly irreversible.
- **All variants** add the unpaid-subscription and domain-loss risks to the account that
  holds your whole Google life.

If you manage a family group or supervise a child's account, as your question implies,
Workspace can't be your only account, because Workspace accounts can do neither. Workspace
stays rejected. The remaining plan is unchanged: Option 4 and/or the mmail Outbox
(OPTIONS.md).
