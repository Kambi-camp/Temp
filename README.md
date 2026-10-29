<p align="center">
  <img src="kambi-logo.svg" alt="KAMBi" width="220">
</p>

# KAMBi x Textify Africa: Technical Documentation

Submission document for the Textify Africa Innovation Competition.
Project: KAMBi (kambi.camp). Prepared October 2026.

This document explains what KAMBi is, where Textify Africa sits inside it, how the
integration is built, how we protect people's privacy and the key, how we tested it, and
what is live today versus ready to switch on. It describes the design and the contracts.
It does not publish source code: the KAMBi repository is private.

## 1. What KAMBi is, in one minute

KAMBi is a field-research and health-outreach platform built for Tanzania. Organizations
run a "camp" (a screening day, a survey, an outreach event). Field staff capture entries
on a phone, often with no signal. The entries sync to a server when the phone is back
online. Headquarters (HQ) staff run the platform from a separate console.

The people KAMBi serves are often reached only by mobile phone, and many do not use
smartphone apps. That is why SMS is not a side feature for us. It is the one channel
that reaches everyone.

## 1a. Why this matters

Most health outreach in Tanzania still runs on paper forms, WhatsApp photos and memory. A
camp screens hundreds of people in a day, and then the trail goes cold: nobody knows who
went to the clinic, who needs a reminder, or what the numbers say. KAMBi fixes the full
loop: record in a minute, work with no signal, protect the person, report in one tap, and
bring people back with a message they can read on any phone. Textify Africa is the part
that closes that loop.

## 1b. What a person can do in KAMBi

This is the user side of the product (staff tools are left out on purpose).

**Get started**
- Create an account with a phone number and a password. No email needed.
- Three personal security questions and a device check keep the account safe and let a
  person recover access without anyone's help.
- Choose Kiswahili or English at any time. The whole app switches.
- Install it like an app: Android download, or add to the home screen. It also works in
  any browser.

**Run a camp**
- A step-by-step camp builder: name, place, dates, what to ask, who may help, and a
  consent screen the volunteer must show.
- Invite helpers with a referral link and see who joined.

**Record people, even with no signal**
- A fast entry form, about a minute per person, with a consent gate before anything is
  saved.
- Entries are saved on the phone first. When the network returns they sync on their own,
  safely: sending twice never creates a duplicate.
- If two phones disagree, nothing is overwritten. A person decides.
- A person can withdraw consent, and their answers are wiped.
- A printed or on-screen slip for the person recorded.

**Report and share**
- One-tap summary report as a print-ready PDF, a Word file or a spreadsheet.
- Small groups are hidden automatically so no one can be identified from the numbers.
- A Branding centre for photo frames, QR posters and the organization logo, so a camp
  looks like itself.

**Stay connected**
- A feed with stories and photo posts, chat between teams, and a library where people
  write and share guides with a simple editor.
- Notifications and a personal "You" area for profile, devices and language.

**Bring people back (where Textify comes in)**
- When a person was referred to a clinic and has not gone, KAMBi can send a short,
  respectful SMS reminder, limited in number and never to someone who opted out.
- A person can reply through a simple follow-up link, with no account needed.

**Private by design**
- Names stay apart from the data used for reports.
- Sensitive text can be redacted on the phone before it leaves the device.

## 2. Why Textify Africa

| Need | Why SMS through Textify fits |
| --- | --- |
| Reach people with no data bundle or smartphone | SMS works on any handset. |
| Tanzanian numbers and operators | Textify normalizes `0712...` and `+255712...` to the `255...` form and bills in local terms. |
| Follow-up after a camp (a referral that was not attended) | Scheduled and idempotent sends, with delivery state we can read back. |
| Account safety | Single-use codes with expiry, from one API. |
| Cost control for a non-profit style budget | Transparent unit billing (160 / 153 characters, 70 / 63 for non-ASCII) that we can calculate before we send. |

## 3. Where Textify sits in the system

```
 Phone / browser (KAMBi app)           HQ console
        |                                   |
        |  HTTPS, signed-in user            |  HTTPS, staff login
        v                                   v
 +---------------------------------------------------------+
 |  Supabase (Postgres + row-level security + auth)        |
 |  - decides WHO may queue a message                      |
 |  - checks opt-out BEFORE anything is queued             |
 |  - stores a masked number, a hash, and a short preview  |
 +---------------------------+-----------------------------+
                             |  queued rows (server side only)
                             v
 +---------------------------------------------------------+
 |  Server worker (holds the Textify key as a secret)      |
 |  - Textify adapter: builds request, reads response      |
 |  - idempotency key per message                          |
 +---------------------------+-----------------------------+
                             |  HTTPS + bearer token
                             v
                     Textify Africa API
                  portal.textify.africa/api/v1
                             |
              delivery status  (poll + webhook)
                             v
                  back into Supabase, then HQ
```

Three rules shape the whole design:

1. **The browser never talks to Textify.** The key exists only on the server worker. Our
   build pipeline runs an automated scan of every built web and HQ bundle and fails the
   deploy if a secret pattern is found.
2. **The database is the gatekeeper.** Opt-out, rationing and "who may send" are enforced
   in Postgres, so a bug in any front end cannot send an unauthorized message.
3. **The provider call is its own small module.** Policy lives in the database. The
   adapter only translates between KAMBi and Textify. Swapping or mocking the provider
   touches one module.

## 4. The Textify endpoints we use

Base URL: `https://portal.textify.africa/api/v1`. Authentication: bearer token.

| Textify endpoint | Used for in KAMBi |
| --- | --- |
| `POST /messages` | Follow-up and reminder SMS, sent atomically with balance check. Supports scheduling. |
| `GET /messages`, `GET /messages/{id}` | Reading delivery state back (queued, sent, delivered, failed). |
| `POST /otps` | Sending a single-use code for phone verification. |
| `POST /otps/verify` | Checking the code the person types in. |
| Webhook (per key) | Receiving delivery updates without polling. |

We use the single response envelope Textify returns (`success`, `status_code`,
`message`, `data`, `error`) with one parser for every call, and we map the documented
error codes (`validation_error`, `creation_error`, `invalid_token`, and so on) into
plain-language results.

## 5. Integration design

### 5.1 The adapter

A dedicated package builds each request and interprets each response. The network call is
injected, which gives us:

- no hidden global state and no network in unit tests;
- one place that formats numbers (to `255...`), counts billable segments, and attaches
  the idempotency key;
- one place to read Textify's error envelope.

### 5.2 Counting cost before sending

Textify bills by segment. Our adapter computes segments with the same rule Textify
documents: up to 160 characters is one unit, longer text bills per 153; text containing
non-ASCII characters (for example some Kiswahili punctuation) bills at 70 then 63 per
part. HQ and field managers therefore see "this message will cost N units" before it is
queued, and the stored row records the segment count.

### 5.3 Safe retries

Every send carries an `Idempotency-Key` (Textify remembers it for 48 hours). If a
request times out and the worker retries, the person is not messaged twice and the
account is not billed twice. This matters on rural networks where timeouts are normal.

### 5.4 Message lifecycle

KAMBi keeps its own small state machine, mapped from Textify's:

| KAMBi state | Meaning | Textify states it covers |
| --- | --- | --- |
| `queued` | Accepted by KAMBi rules, not yet handed over | before send |
| `sent` | Textify accepted it | `sending`, `processing`, `sent` |
| `delivered` | Reached the handset | `delivered` |
| `failed` | Did not arrive | `undelivered`, `failed`, `operator_rejected`, `expired`, `unknown` |
| `opted_out` | Blocked by KAMBi before sending | never sent |

### 5.5 Use case A: closing the loop after a camp

A person is screened and referred to a facility. If the referral is still "waiting" 48
hours later, a scheduled job selects it, subject to these limits, all enforced in the
database:

- at most 5 reminders per referral;
- at least 24 hours between reminders;
- never to a number that has opted out;
- the job stamps the reminder and queues the row first. Only then does the worker call
  Textify.

The result is an automatic, respectful nudge instead of a lost patient.

### 5.6 Use case B: one-time codes

Phone verification uses Textify's OTP endpoints (codes are single-use and expire in 30
minutes). The code is generated and checked by Textify, so KAMBi never stores or sees the
code. Account-verification codes stay on the SMS channel.

### 5.7 Use case C: HQ SMS centre

HQ staff see message volume, segments, delivery state and failures in one list, filtered
by event. Opt-outs are permanent per number.

## 6. Privacy and safety

KAMBi works with sensitive health-adjacent information, so the SMS log is deliberately
thin:

| What we store | What we do not store |
| --- | --- |
| A one-way hash of the number (used for opt-out matching and de-duplication) | The raw phone number in the message log |
| A masked number for display (for example `+255 7** *** 123`) | The full message body (only a short preview) |
| Segment count, status, timestamps, provider id | Any health answer or consent answer |

Further protections:

- **Consent first.** Entries carry a consent record. Withdrawal wipes the answers.
- **Opt-out is final.** A number that opts out is blocked at the database level, before
  the adapter is ever called.
- **Least privilege.** Row-level security limits who can read the SMS log. Only HQ staff
  see it.
- **Key handling.** The Textify key is held as a server secret, never in the repository,
  never in a bundle. Rotation is a documented runbook step.
- **No health details in SMS.** Message templates carry the minimum: who to contact and
  where to go.

## 7. Reliability on poor networks

- The mobile app records entries on the phone first and syncs later, so SMS-triggering
  events (such as a referral) reach the server whenever signal returns.
- Idempotency keys make retries safe.
- Delivery state is reconciled both by webhook and by polling, so one missed webhook
  does not leave a message stuck.
- A failure never blocks the field workflow. It becomes a visible `failed` row for HQ.

## 8. Quality evidence

| Check | Result |
| --- | --- |
| Adapter contract tests (number formats, segment maths, envelope and error handling, idempotency) | 10 of 10 passing |
| Whole platform automated tests | 653 passing across 61 files |
| Database rules (opt-out, reminder limits, masking, consent) tested against a real Postgres 16 | passing |
| Secret scan on every built surface in the deploy pipeline | clean |
| Live key check against the Textify API | authenticated, `200 OK` |

## 9. Honest status: live today and ready to switch on

| Capability | Status |
| --- | --- |
| Textify adapter (send, OTP send and verify, message lookup), tested | Done |
| Live credentials verified against the Textify production API | Done |
| Database rules: SMS log, opt-outs, reminder selection and stamping, HQ queue | Done |
| HQ SMS centre screen and delivery write-back | Built, connected to the log |
| Reminder worker calling the adapter on a schedule | Next step (the data side is done; the scheduled sender is being wired) |
| Phone one-time-code sign-in | Designed on the OTP endpoints. For launch, sign-in uses phone and password with security questions and device checks, so the app works with no SMS cost. OTP is the planned upgrade. |
| Approved sender name | Pending approval from Textify. Until then sends use the default sender. |

We prefer to say this plainly. The integration is real, authenticated and tested, and the
remaining step is operational (scheduling the worker and the sender-name approval), not
architectural.

## 10. Impact and roadmap

- **Fewer lost referrals.** Automated, limited, opt-out-respecting reminders after a camp.
- **Inclusion.** SMS reaches people without smartphones or data.
- **Next:**
  1. Scheduled reminder worker live.
  2. Kiswahili message templates, reviewed by native speakers.
  3. OTP as a second factor on new devices.
  4. WhatsApp channel through the same adapter (Textify supports `channel: whatsapp`).
  5. Delivery-rate dashboard per camp in HQ.

## 11. What we would like from Textify Africa

- Sender-name approval for "KAMBi".
- Guidance on volume pricing for health outreach and non-profit partners.
- A sandbox or test number range for demonstrations.

## 12. Contact

KAMBi team, kambi.camp. Product and technical contact: noelchesco5 (GitHub), via the
competition portal.

## Appendix: terms

- **Camp:** a day or period when an organization collects entries in the field.
- **Entry:** one person's recorded answers, with consent.
- **HQ:** the staff console, separate from the public app.
- **Segment:** one billable SMS part.
- **Idempotency key:** a label that makes a repeated request count only once.
