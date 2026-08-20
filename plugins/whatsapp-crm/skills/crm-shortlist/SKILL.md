---
name: crm-shortlist
description: Read a CodeYogi export — WhatsApp chats or Gmail threads — and shortlist who should be added to the CRM. Use when the user says /crm-shortlist, asks who from WhatsApp or from their mail should be in the CRM, or points at a chats-*.ndjson or mail-export-*.ndjson export.
---

# CRM Shortlist

Reads an **export** — WhatsApp chats from the CodeYogi WhatsApp adapter, or Gmail threads from
the ingestor's mail export — and writes a JSON shortlist of the people worth adding to the CRM.
Writes nothing else: no CRM call, no network. Adding a contact stays a human action.

Two sources, ONE judgement. The mail line is deliberately shaped like the chat line so the same
standard of evidence applies to both — do not develop a looser bar for mail because there is
more of it.

**Which source am I looking at?** A `chats-*.ndjson` is a chat export and every rule below about
chats applies. A `mail-export-*.ndjson` is a mail export: read *Find the mail export* and *A mail
shortlist is per PERSON* as well, and skip the WhatsApp-specific fields (`isMyContact`,
`isBusiness`, `verifiedName`, `members`).

## Find the export

The app writes to `chat-exports/` inside its own Electron data folder. **That folder has two
possible names — check BOTH on every platform and use whichever exists:**

- `whatsapp-local-adapter` — package.json's `name` field, which is what `app.getName()` returns
  when the app is run from source (`npm run app`). Verified on Linux.
- `CodeYogi WhatsApp` — the packaged build's `productName` (electron-builder.yml), which
  electron-builder writes into the bundle's `CFBundleName`, and `app.getName()` reads that.
  Expected on a packaged macOS build; likely on packaged Windows too.

So the same app has different data folders depending on how it was installed, and assuming one
name reports "No export found" forever on the other. Look under both parents:

- Linux: `~/.config/<name>/chat-exports/`
- macOS: `~/Library/Application Support/<name>/chat-exports/`
- Windows: `%APPDATA%\<name>\chat-exports\`

If both exist, take the newest export across the two and say which folder it came from.

Take the newest `chats-*.ndjson` unless the user gave a path. **Say which file you took and how
old it is** before judging anything: `reading chats-2026-08-12.ndjson (2 hours old, 137 chats)`.
A file older than 7 days gets a warning, not a refusal — chats move, and overriding that is the
operator's call.

If the folder is missing or holds no export, say **"No export found — open CodeYogi WhatsApp and
click Export chats"** and stop. Never try to reach WhatsApp yourself: the app owns that session,
and a second browser would fight it for the profile lock.

A leftover `chats-*.ndjson.part` means an export died before writing anything. Ignore it.

## Find the mail export

A mail export is downloaded by the ingestor frontend, so it lands wherever that browser puts
downloads — `~/Downloads/mail-export-YYYY-MM-DD.ndjson` on Linux and macOS,
`%USERPROFILE%\Downloads\` on Windows. Take the newest `mail-export-*.ndjson` unless the user
gave a path, and say which file you took and how old it is, exactly as for a chat export.

If there is none, say **"No mail export found — open the ingestor and press Export my mail"** and
stop. Never try to read the mailbox yourself: the export is the only sanctioned path, and it is
the thing that applies the filters.

**That file is unmanaged plaintext holding mail from people who are not CRM contacts.** Read it,
judge it, write the shortlist beside it — never copy it anywhere else, and never paste thread
bodies into chat beyond the one or two quoted lines an entry's `evidence` needs.

## What each line looks like

### A chat export line

One JSON object per line:

```json
{"chatId":"9111…@lid","name":"Rahul Verma","phone":"919111000000","chatType":"direct",
 "isMyContact":true,"isBusiness":false,"verifiedName":null,"lastTs":"2026-08-11T09:14:41.000Z",
 "members":null,"memberSource":null,
 "msgs":[{"ts":"…","me":false,"sender":{"phone":"919111000000","inCrm":false},"body":"…"}]}
```

- `me: true` is the operator speaking.
- `sender.inCrm` is whether that person is already a CRM contact.
- Groups carry `members` and `memberSource`: `"roster"` is real membership, `"senders"` means
  only the people who spoke are known — say "membership unknown" rather than treating four
  talkers as the whole group.
- Chats already in the CRM, and archived chats, are already excluded by the export.

### A mail export line

One JSON object per THREAD:

```json
{"threadId":"18f0…","subject":"About the fellowship","lastTs":"2026-08-11T09:14:41.000Z",
 "participants":[{"address":"anita@ngo.org","inCrm":false},{"address":"me@codeyogi.org","inCrm":false}],
 "msgs":[{"ts":"…","me":false,"sender":{"address":"anita@ngo.org","inCrm":false},"body":"…"}]}
```

- `me: true` is the operator speaking; `sender.inCrm` is whether that person is already a CRM
  contact — the same two fields the chat line carries, and they mean the same thing.
- The export has ALREADY dropped: threads only one side spoke in, threads whose every outside
  participant is a colleague, threads where everyone is already a contact, and robot or desk
  addresses (`noreply@`, `invoices@`, `careers@` and kin). Do not re-implement those filters —
  but do note that role addresses like `info@`, `contact@`, `hello@`, `admin@` and `office@` are
  KEPT on purpose, because that is how a great many small nonprofits correspond. Judge them on
  what was said, never on the local part.
- A thread that survived because it mixes a known contact with an unknown person is the single
  best row this file can produce — an introduction. Look for them.
- There is no `phone`, no `isMyContact` and no group roster. The identifier is the ADDRESS.

## Who to shortlist

**Include:**

- **Partner / funder / institute** — a school, college, NGO, CSR arm, foundation or government
  body discussing running, hosting, or funding CodeYogi programs.
- **Vendor / service provider** — someone the org buys from or contracts: hosting, printing,
  logistics, freelancers, agencies.

**Exclude:**

- Students and course enquiries (hour-of-ai, ai-basics, fees, certificates). They would swamp
  the list and are handled elsewhere.
- Personal and family chats.
- Volume alone. A long back-and-forth about nothing is not a signal; a three-message thread
  naming a funding commitment is.

**How to judge:**

- A **direct** chat is judged on what was said. The messages are the evidence.
- A **group** is judged on who is in it: how many members are already contacts, and who the
  unknown ones are.
- A **mail person** is judged on everything they and the operator said across every thread they
  share. Thread count is context, not a verdict: one thread naming a partnership beats nine
  rounds of scheduling.
- `isMyContact`, `isBusiness` and `verifiedName` are supporting evidence, never the verdict on
  their own. Saved-but-not-in-the-CRM is a stronger candidate than a stranger who messaged once;
  a verified business name supports a vendor read. All three absent means enrichment failed —
  judge on messages alone, do not read it as `false`.

## A mail shortlist is per PERSON

A chat entry is a CHAT. A mail entry is a **PERSON**, because someone spread across a dozen
threads is one strong signal rather than a dozen weak ones, and what the operator is deciding is
whether to create one Person.

So for a mail export: group the threads by counter-party address — every participant who is not
the operator and not on the org's own domain — and emit at most one entry per address. Its
evidence is drawn from across their threads, and the entry carries the threads behind it so the
operator can see why you think this person matters.

Keep the same `entries[]` shape (see below), with these mail-specific fields:

- `name` — the display name from the From header if the export carries one, else `null`. NOT
  `nameFromWhatsApp`, which names a WhatsApp address-book nickname and would be a lie here.
- `identifiers` — `[{ "type": "email", "value": "anita@ngo.org", "isPrimary": true }]`.
- `threadCount`, `messageCount`, `firstSeen`, `lastSeen` — the aggregate that made this a signal.
- `threads` — `[{ "threadId": "…", "subject": "…" }]`, the threads behind the judgement.

Everything else — `reason` as free text, no verdict enum, `evidence` as one or two quoted
messages, dropped candidates as a COUNT not a list, strongest first — is unchanged. A person-level
entry passes through the existing flattening step untouched, which is the point of keeping the
shape.

## Write the shortlist

`shortlist-YYYY-MM-DD.json`, beside the export (for a mail export that means the downloads
folder — the ingestor's checklist is where you drag it afterwards). **JSON only — no markdown file.** The app is
expected to render this later, and prose is not a format a window can lay out.

```json
{
  "schema": 1,
  "generatedAt": "2026-08-12T12:30:00.000Z",
  "source": "chats-2026-08-12.ndjson",
  "counts": { "chatsRead": 137, "shortlisted": 11, "dropped": 126 },
  "entries": [
    {
      "chatId": "9111…@lid",
      "chatType": "direct",
      "nameFromWhatsApp": "Ramesh printing bhaiya",
      "identifiers": [{ "type": "phone", "value": "919111000000", "isPrimary": true }],
      "reason": "prints our workbooks — quoted per-unit rates and delivery dates",
      "evidence": [{ "ts": "2026-08-10T11:02:00.000Z", "me": false, "body": "…" }],
      "members": null
    }
  ]
}
```

Rules for that file:

- **`schema: 1`.** A window rendering this is written against a shape. Do not change the shape.
- **`reason` is free text, and there is NO verdict enum.** The CRM forbids role enums
  (`crm_feats/04-tag.md`: *"a `donor` tag is fine as a vibe; it is not a role, and nothing may
  branch on it"*). Never emit `verdict: "funder"` or similar.
- **`identifiers` mirrors the CRM's Person shape** (`crm_feats/01-person.md`), so whoever adds
  them is not re-deriving it. A chat with `phone: null` gets `"identifiers": []` and the reason
  must say the number is unknown, naming the `chatId`.
- **`nameFromWhatsApp`, named for what it is.** These are saved nicknames; a Person created from
  one carries "Ramesh printing bhaiya" as its canonical name forever. Prefer `verifiedName` in
  the `reason` when there is one.
- **`evidence` is quoted messages** — one or two, enough for a human to disagree with you.
- For a group entry, `members` carries the marked member list you judged on; otherwise `null`.
- **A group entry's `reason` MUST name, as bare digits, the member or members you judge
  strongest** — "918108431006 directs the blog content and is the strongest add here", not "the
  person who directs blog content". `members` is `{phone, inCrm}` and carries no names or
  ranking, so this mention is the ONLY per-member signal `schema: 1` can carry. The app's
  Add-to-CRM checklist matches those digits to mark and sort the member rows; phrase the reason
  without them and a five-person roster arrives as five identical-looking rows for a human to
  guess between. Free text elsewhere, load-bearing here.
- **Dropped chats are a count, not a list.**
- Rank strongest first.

Then tell the user, in the chat, how many you shortlisted and where the file is. Do not paste the
whole shortlist as prose.

**A run that shortlists nobody is a real answer.** Say so plainly — "read 214 threads, nobody
worth adding" — and still write the file with `entries: []`. Silence, or an unexplained empty
file, reads as a failure the operator then goes hunting for.
