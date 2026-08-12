---
name: crm-shortlist
description: Read a CodeYogi WhatsApp chat export and shortlist which conversations should be added to the CRM. Use when the user says /crm-shortlist, asks who from WhatsApp should be in the CRM, or points at a chats-*.ndjson export.
---

# CRM Shortlist

Reads a chat export produced by the **CodeYogi WhatsApp** adapter and writes a JSON shortlist of
the conversations worth adding to the CRM. Writes nothing else — no CRM call, no network. Adding
a contact stays a human action.

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

## What each line looks like

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
- `isMyContact`, `isBusiness` and `verifiedName` are supporting evidence, never the verdict on
  their own. Saved-but-not-in-the-CRM is a stronger candidate than a stranger who messaged once;
  a verified business name supports a vendor read. All three absent means enrichment failed —
  judge on messages alone, do not read it as `false`.

## Write the shortlist

`shortlist-YYYY-MM-DD.json`, beside the export. **JSON only — no markdown file.** The app is
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
