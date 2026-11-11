---
name: gmail-operations
description: "Run real, multi-step Gmail work through the Composio MCP gateway: bulk delete, message triage, and the failure modes that leave a draft unsent."
version: 1.0.0
license: MIT
author: "Hermes Agent"
---
# Composio Gmail Operations

Running anything non-trivial against Gmail through Composio (`connect.composio.dev/mcp`) hits a small set of well-defined gotchas. This skill is the operational playbook distilled from running 100k+ message operations end-to-end.

Use this skill whenever the user asks to:
- Bulk delete, archive, or label a large number of Gmail messages
- Reclaim Gmail / Google account storage
- Organize an inbox by category (Personal, Business, Finance, etc.)
- Run a "cleanup my email" workflow

For initial Composio **setup** (OAuth, account linking, the "tools/list is empty" activation step), use the parent `native-mcp` skill. This one assumes you already have a working Composio session and access token.

## Architecture of a multi-thousand-message job

Do **not** try to fit a big Gmail job into a single chat turn. The right shape is a long-running worker script in the background, with state and logs written to disk. The agent's job is to design + launch the worker, then poll its log/state file.

```
[chat turn]                  [background process]
  design worker    ---->      python worker.py (writes log + state)
  launch it                   walks pages, batches deletes/labels
  check log/state             backoff on 401, auto re-init
  report back
```

A working reference implementation is at:
`<hermes-home>/run/gmail_cleanup_worker.py` — drove a 165,724-message purge in this session.

Key shape of that worker:
1. `init()` — open MCP session, call `COMPOSIO_SEARCH_TOOLS` to activate Gmail toolkit, save `session_id`.
2. `fetch_ids(query, page_token=None)` — page through Gmail search results, 50 IDs at a time.
3. `batch_delete(ids)` / `batch_label(ids, add, remove)` — chunk into 50-ID groups, call via `COMPOSIO_MULTI_EXECUTE_TOOL`.
4. Outer loop: walk a query until 2 consecutive empty pages, then move to the next query.
5. Always catch 401s: re-init the session and retry. Composio rotates Bearer tokens periodically; the cached access token keeps working but the live session can get bounced.

## The 50-message hard cap (the #1 trap)

**Composio's `GMAIL_FETCH_EMAILS` silently returns a `data_preview` (only 2 IDs + a `'...N more items'` string) for any `max_results > 50`.** The response still includes `resultSizeEstimate` (capped at 201) and a `nextPageToken`, but you only get 2 real IDs and a placeholder string. If you then hand that string array to `GMAIL_BATCH_DELETE_MESSAGES`, it fails with:

```
Invalid Gmail message ID format detected: ['...444 more items'].
Gmail message IDs must be hexadecimal strings.
```

**`verbose: true` does NOT override the cap. It's silently ignored.**

The cap is on the **fetch** side, not the mutate side. `BATCH_DELETE_MESSAGES` and `BATCH_MODIFY_MESSAGES` accept up to 1000 IDs per call. So:

- Always request `max_results: 50` (or less).
- Walk `nextPageToken` to get the full set of IDs.
- Chunk the IDs into 50-per-batch (or 1000) for mutations.
- For the truly large results, set `sync_response_to_workbench: true` and read back from `remote_file_info.file_path`.

## size data is unavailable through Composio

`GMAIL_FETCH_EMAILS` (any format) and `GMAIL_FETCH_MESSAGE_BY_MESSAGE_ID` (`metadata`, `minimal`, `full`, `raw`) both return `sizeEstimate: null` and `attachmentList[].body.size: 0`. Filenames and headers are preserved, every size field is wiped.

This means:
- You **cannot** rank messages by size.
- You **cannot** decide per-message whether to keep/delete based on storage impact.
- You can only use Gmail's server-side search operators (`larger:5M`, `has:attachment filename:mp3`, etc.) as filters.
- For real per-message size data, bypass Composio and use `google-api-python-client` with a direct Google OAuth client.

Implication for storage-reclaim projects: design around categories and Gmail search operators, not per-message byte measurement. The 95% case is good enough; the 5% that wants exact numbers needs the direct API path.

## Gmail search operators that work well at scale

For "delete a category" sweeps, the right pattern is a *chain* of broad queries with safety guards, not one perfect query:

```python
DELETE_BUCKETS = [
    ("spam",                 "in:spam",                                                       True),
    ("promotions",           "category:promotions",                                           False),
    ("social",               "category:social",                                               False),
    ("forums",               "category:forums",                                               False),
    ("updates_old_6m",       "category:updates older_than:6m",                                False),
    ("voicemails",           "subject:(voicemail OR \"voice mail\" OR \"voice message\") has:attachment", False),
    ("google_photos_mail",   "from:googlephotos OR from:photos.google.com OR subject:\"Google Photos\"", False),
]
```

For every category beyond spam, **protect** starred and important:
```python
PROTECTED = "-is:starred -is:important"
q = f"({query}) {PROTECTED}"
```

`from:` matching in Gmail search is loose. `from:tmobile.com` does not match `voicemail@tmomail.net` even though they're the same brand. **Always OR several variants** and verify with a small test before launching the full sweep.

## Label / category organization pattern

When applying user-created category labels, the working pattern is:

1. **Create labels first** via `GMAIL_CREATE_LABEL`. Numeric prefixes (`01 Personal`, `02 Business`, ...) sort them in the Gmail sidebar.
2. **Capture each label's ID** (`Label_14`, `Label_15`, ...). Custom-label API calls need the ID, not the name.
3. **Query the messages to label** with negated-label filters so you don't re-label already-labeled mail:
   ```
   (from:chase OR subject:statement) -label:"04 Finance" -label:"03 Tech"
   ```
4. **Order your rules by specificity** — most specific label first. The first matching rule wins, so put Finance before Personal, Tech before Notifications, etc.
5. **For non-personal categories, remove the INBOX label** in the same `BATCH_MODIFY_MESSAGES` call (`removeLabelIds: ["INBOX"]`) so the messages leave the inbox. Keep INBOX on Personal / Business / Finance where the user actively checks.

A label-creation + categorization worker that handled 17,703 messages: `<hermes-home>/run/gmail_cleanup_worker.py`.

## Reauth handling

Composio rotates session bearer tokens silently. Your long-running worker will see occasional `401 Unauthorized: Bearer token rejected`. Recovery:
- Re-initialize the MCP session: re-send `initialize` and `notifications/initialized`, capture the new `mcp-session-id` from response headers.
- Resume the worker. The cached `access_token` in `<hermes-home>/mcp-tokens/composio.json` is still valid; only the live session needs refresh.
- Don't re-trigger the OAuth browser flow unless `access_token` itself is rejected (different error).

## Label deletion (after purge)

When the user wants to drop a category of labels (e.g. "delete the GMass Reports labels"):
- `GMAIL_LIST_LABELS` (no `include_details`) returns the `id` (`Label_2`, etc.) and `name` for every label.
- `GMAIL_DELETE_LABEL` with `label_id` permanently removes the label from the account and from all messages. The messages themselves are not deleted; they just lose that tag.
- This is fast (~9 labels in <30s). No safety query needed — Gmail doesn't allow deleting system labels (INBOX, SENT, etc.) so the worst case is a "label not found" error.

## Drive uploads from a worker (the local-path trap)

`GOOGLEDRIVE_UPLOAD_FILE` accepts a `file_path` parameter, but **the path must point to a server-side staged file** (e.g. `/mnt/files/...` in the Composio remote sandbox), not a local `/tmp/...` on the agent's host. A local path returns:
```
Failed to retrieve uploaded file content. The file does not exist in storage,
may have expired, or the file reference is invalid.
```

Three workarounds:
1. Stage via `COMPOSIO_REMOTE_BASH_TOOL` (`echo '...' > /mnt/files/foo.txt`).
2. `COMPOSIO_REMOTE_WORKBENCH` with `upload_local_file` to push to Composio's S3/R2, then drive-upload from there.
3. `GOOGLEDRIVE_UPLOAD_FROM_URL` if you have a public URL (Gmail attachments are not public — this won't work for archive flows).

For bulk archive-of-many-emails flows, write the archive *manifest* (one small text file per email with metadata) and upload that. Don't try to upload the actual attachment bytes — by the time you've streamed them through Composio's layers you've lost the size data anyway.

## Performance

Illustrative throughput estimates for a large mailbox (measure against your own provider and gateway):
- 50-msg batch delete: ~3-5 seconds per batch (delete API itself is fast; rate is constrained by Composio's MCP gateway).
- 50-msg batch label: ~3-5 seconds per batch.
- 50-msg fetch: ~2-3 seconds.
- Effective throughput: **~1,000 messages per minute** for pure delete sweeps; slower (~500/min) for label + delete combined.

A 100k-message purge is a ~2-hour background job. Budget for at least 2-3 transient 401s that need session re-init (auto-handled by the worker pattern above).

## Quick start checklist

For any new "clean up my email" or "reclaim my Gmail storage" request:

1. ✅ Read `native-mcp` skill — confirms OAuth + toolkit activation still works.
2. ✅ Run a `GMAIL_GET_PROFILE` + `GMAIL_LIST_LABELS` to snapshot the current state (save to a state file).
3. ✅ Plan queries by category — use Gmail's built-in operators, add `PROTECTED = "-is:starred -is:important"`.
4. ✅ Create labels first if organizing.
5. ✅ Write a worker to a stable path (NOT `/tmp` — Hermes' sandboxed temp dir is a symlink loop). Use `<hermes-home>/run/`.
6. ✅ Launch the worker as a `background=true` terminal process with `notify_on_complete=true`. Do not block the chat.
7. ✅ Poll the log every few minutes. Report the final state from the state JSON file, not from memory.
8. ✅ After the worker finishes, propose follow-up actions (storage check, label deletion, archive-to-Drive) — those are the natural next questions.

## Reference files

- `references/storage-reclaim-playbook.md` — second-pass playbook for storage reclaim after a Phase 1 purge: what you can/can't measure through Composio, recommended queries per category, decision tree, and the "give up and use a direct Google API" escape hatch.
- `<hermes-home>/run/gmail_cleanup_worker.py` — production worker that purged 165,724 messages in ~3.2 hours and applied 17,703 labels.
- `<hermes-home>/run/gmail_phase2_worker.py` — second-pass worker for storage reclaim (use as starting point, not as-is — the size field issue means it needs the simpler "filter-and-delete" pattern).
