# Storage Reclaim Playbook (Phase 2)

A second-pass workflow that runs after a successful Phase 1 inbox purge. Phase 1
deletes the easy categories; Phase 2 targets the long tail of attachment-heavy
mail that doesn't fit any single Gmail category.

The skills in this file assume you've already done Phase 1 (the main `gmail_cleanup_worker.py` pattern) and have a working Composio session with Gmail + Drive activated.

## What Phase 1 doesn't catch

After deleting 99% of Promotions, all Spam, all Social, all Forums, and old Updates, you still have ~568 GB of Gmail storage on a typical personal account. That residual lives in messages with **real attachments** that Gmail's category filters don't apply to. Common buckets:

- Old Google Photos auto-backup emails (image attachments)
- Voicemail `.mp3`/`.wav`/`.amr` attachments from carriers
- Bank/PDF statements (often attached, not linked)
- Image attachments from friends/family (no longer in your Drive)
- Group text / MMS messages forwarded to Gmail by carriers
- Old Google Drive notification emails (with thumbnails of Drive files)

## What you CAN do through Composio

| Action | Works? | Notes |
|---|---|---|
| Filter by Gmail search operator (`larger:5M`, `has:attachment`, `filename:mp3`) | ✅ Yes | This is the only size-related filter that works. |
| Get per-message `sizeEstimate` | ❌ No | Always null. See SKILL.md. |
| Get `attachment[].body.size` | ❌ No | Always 0. See SKILL.md. |
| Get `attachment[].filename` | ✅ Yes | Use this to filter by file type. |
| Get message body via `format=raw` | ❌ No | Composio returns an error ("'data'") for `raw` format on the messages I've tested. |
| `BATCH_DELETE_MESSAGES` by ID | ✅ Yes | Same caveats as Phase 1. |
| Upload the message body to Drive as a manifest file | ⚠️ Partial | See SKILL.md "Drive uploads from a worker". |
| Upload the actual attachment bytes to Drive | ❌ No | Requires local file path that Composio's sandbox can read. Gmail attachments aren't accessible as local files. |

## Recommended Phase 2 queries

Run these in order. Each is a separate sweep of the full mailbox; combine with `include_spam_trash=True` so deleted spam also gets cleaned.

```python
# Voicemails (high-confidence deletable, almost always junk)
VOICEMAIL_QUERIES = [
    'has:attachment (filename:mp3 OR filename:wav OR filename:amr OR filename:m4a OR filename:3gp)',
    'subject:(voicemail OR "voice mail" OR "voice message" OR "new voicemail")',
    'from:(tmobile.com OR tmomail.net OR voicemail.net OR vtext.com OR verizon.com OR verizonwireless.com OR att.com OR sprint.com OR messaging.sprintpcs.com)',
]

# Old image attachments (you may want to archive first, not delete)
IMAGE_QUERIES = [
    'has:attachment (filename:jpg OR filename:jpeg OR filename:png OR filename:heic OR filename:gif) larger:2M older_than:2y',
    'has:attachment (filename:mov OR filename:mp4 OR filename:m4v) larger:10M older_than:2y',
]

# Google Photos notification sweep (often the largest single bucket)
PHOTO_QUERIES = [
    'from:googlephotos OR from:photos.google.com',
    'subject:("Google Photos" OR "Backed up" OR "Storage is getting low")',
    '"Your storage" from:googlephotos',
]

# Group/MMS via carriers
MMS_QUERIES = [
    'subject:(MMS OR "group message")',
    'has:attachment (filename:jpg OR filename:jpeg) "Sent from my"',
]
```

## Decision tree per email

When you cannot measure size, you have to decide based on the **category and age** rather than bytes. A useful default rule set:

1. **Voicemails** older than 1 year: **delete**. Carriers keep the originals.
2. **Google Photos notification emails** of any age: **delete** (the photos themselves are in Photos, not the email).
3. **Image attachments** from group texts / forwarded from phones older than 2 years: **delete**.
4. **PDF statements / receipts** older than 7 years: **archive to Drive + delete from Gmail**. The Drive file is searchable, the email is just a duplicate.
5. **Bank / tax / legal attachments** of any age: **NEVER delete without explicit confirmation**. Even if the user says "delete everything," flag these for human review.

## When to give up and use a direct Google API

If the user asks for *exact* per-message byte counts or wants a guaranteed "I freed exactly N GB" report, bypass Composio. The right tool:

- `google-api-python-client` with `users.messages.get(format='full')` — returns real `sizeEstimate` per message.
- `users.messages.attachments.get` — returns the real `data.size` per attachment.
- Set up once: create a Google Cloud OAuth client (Desktop app), download `client_secret.json`, use the `google-workspace` skill's setup.py pattern.

Cost: ~10 minutes of setup. Worth it for any "I want to know exactly" project.

## Drive archive pattern (when you do this)

For messages you're archiving rather than deleting:

1. Find or create the archive folder once at the start of the worker:
   ```
   GOOGLEDRIVE_FIND_FOLDER {"name": "Gmail Archive <YYYY-MM>"}
   ```
   If not found, `GOOGLEDRIVE_CREATE_FOLDER`.
2. For each message to archive, build a small text manifest:
   ```
   Archived from Gmail
   Date: 2024-03-15
   From: chase@alerting.chase.com
   Subject: Your March statement
   Size: <unknown — Composio strips this>
   Message-ID: 19f9...
   ```
3. Upload the manifest as a `.txt` file to the Drive folder (filename = `YYYY-MM-DD_<messageid>_<subject-slug>.txt`).
4. Delete the Gmail message.

You will **not** have the original attachment bytes in Drive. You're archiving *metadata* about what was there, which is usually enough for tax/audit purposes (you can re-find the original from your bank's website using the date + sender).

## A working "second pass" that does work today

The most reliable second pass with Composio's current capabilities is **voicemail delete**. The queries are tight, the false-positive rate is low (almost nobody has non-voicemail `.mp3` attachments), and the storage impact per message is high.

Pseudocode:

```python
# Phase 2 worker (real, working pattern)
for q in VOICEMAIL_QUERIES:
    page_token = None
    while True:
        ids, page_token, _ = fetch_ids(q, page_token=page_token, include_spam_trash=True)
        if not ids: break
        for chunk in chunked(ids, 50):
            batch_delete(chunk)
            # every voicemail is potentially a few MB
        if not page_token: break
```

Expected yield: 50-500 voicemails × ~200 KB average = 10-100 MB reclaimed. Not huge, but real, and safe.
