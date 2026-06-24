---
name: gsuite-account-routing
description: Decide which of Forrest's three Google accounts to search or write for ANY Google Workspace task — contacts, Gmail, Calendar, Google Voice — so there is no per-task guesswork. The accounts are forrest@nlma.io (primary/canonical), admin@fidumcompany.com (Fidum business), and forrest.surprenant@gmail.com (personal/iPhone). Trigger phrases "my contacts", "add to my contacts", "save X to contacts", "look up X", "find X's number", "find X's email", "is X in my contacts", "do I have X saved", "email X", "draft an email", "check my calendar", "am I free", "text X", "which account". Load this BEFORE searching or writing any contact, email, or calendar event whenever the user did not name a specific account.
---

# Google Workspace account routing (Forrest)

Forrest has **three** Google accounts behind one Workspace MCP. "My contacts / my email / my calendar" does not name one — this skill removes the guesswork so reads and writes go to the right place the first time.

## The accounts
- **forrest@nlma.io** — PRIMARY working account. **Canonical contacts store** plus main mail/calendar. Business contacts, vendors, contractors, deal parties, and clients all live here (e.g. Patrick & Philip Gomez).
- **admin@fidumcompany.com** — Fidum Company business / Workspace-admin account. Fidum-internal mail, calendar, Drive.
- **forrest.surprenant@gmail.com** — personal Gmail. **Assumed to be the account synced to Forrest's iPhone Contacts app** (CONFIRM — this drives the mirror rule below).

## READ rule — finding a person or data
1. **Search all three accounts, in parallel, every time.** Never conclude from a single account, and never trust a prior chat's summary that says "already saved / already exists" — verify live. (On 2026-06-24 a past summary claimed Matthias Gomez's contact was updated; he was in **none** of the three.)
2. Use **`<acct>:search_contacts`** (People API). Do **NOT** use `search_voice_contacts` for lookups — see Gotchas.
3. Only after all three come back empty do you tell Forrest it is "not saved."

## WRITE rule — adding or updating a person
1. **Dedup first** (run the READ rule). If the person exists in any account, **UPDATE** that record (merge mode) — do not create a duplicate. Match across spelling variants (Matthias / Mathias / Matthew; Gomez / Gomes).
2. **Default create target = `forrest@nlma.io`** (canonical store) unless the contact is clearly personal or Forrest names an account.
3. **Mirror to the device-sync account** (`forrest.surprenant@gmail.com`, pending confirm) for anyone Forrest will call or text from his phone — i.e. basically all operational contacts (vendors, contractors, tenants, agents, deal parties). The mirror is what makes the number show up on his iPhone.
4. Formats: phone in **E.164** (`+1XXXXXXXXXX`), type `mobile`; email type `other`; role/company in `organizations`; a one-line provenance note (source + date) in `notes`.

## Tools (exact)
- Search: `<acct>:search_contacts` — `query`, `page_size` (≤30).
- Create one: `<acct>:manage_contact` — `action="create"`, `given_name`, `family_name`, `phones=[{number,type}]`, `emails=[{address,type}]`, `organizations`, `notes`.
- Create several / most reliable create: `<acct>:manage_contacts_batch` — `action="create"`, `contacts=[…]`.
- Update: `<acct>:manage_contact` `action="update"` with `phones_mode="merge"` / `emails_mode="merge"` (read-modify-write; dedups fields).
- `<acct>` is one of the three emails above — the Workspace MCP namespaces its tools per account.

## Gotchas
- **Voice tools need a live Google Voice cookie session.** `search_voice_contacts`, `send_voice_sms`, and the call tools fail on `admin@fidumcompany.com` and `forrest.surprenant@gmail.com` with "Voice sign-in still pending" and return a patchright takeover URL (stale auth, observed 2026-06-24). **People API (`search_contacts` / `manage_contact`) needs no Voice auth** — use it for all contact reads/writes. Only reach for Voice tools when actually sending an SMS or placing a call, and expect a re-auth step.
- **Tool discoverability is noisy.** `tool_search` may not surface the contact tools by description; query the **exact** names (`search_contacts`, `manage_contact`, `manage_contacts_batch`).
- **Past-session summaries are not ground truth.** They have falsely reported contacts as saved. Always verify against the live account before telling Forrest something exists or does not.

## Worked example — Matthias Gomez (2026-06-24)
Ask: "Did you save Matthias to my contacts? If not, do it."
- `search_voice_contacts` → failed on admin + personal (Voice pending).
- `search_contacts` (People API) across all three accounts → absent everywhere (only Patrick / Philip / Lilana Gomez existed).
- Created in **forrest@nlma.io** via `manage_contacts_batch`: `Matthias Gomez`, mobile `+15715645763`, email `matthiasgomez26@gmail.com` (other), org "Roofing Canvasser at NuRoof", note "DoorLoop vendor 6a3b4136c5bb94f9030438e8; added 2026-06-24."
- New Contact ID **c8603297592630407547**.
- Open item: not yet mirrored to the iPhone-sync account (see Config).

## Config — confirm once, then this skill is zero-guesswork
- `CANONICAL_CONTACTS_ACCOUNT = forrest@nlma.io`  ✓ confident
- `DEVICE_SYNC_ACCOUNT = forrest.surprenant@gmail.com`  ← **assumption**; confirm which account backs the iPhone Contacts app. If it is actually `forrest@nlma.io`, the mirror step becomes a no-op (canonical already equals device).
