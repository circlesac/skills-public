---
name: gog
description: Reference guide for the gog CLI — interact with Gmail (search, read, send, attachments, drafts) from the terminal
user-invocable: true
argument-hint: "<search|read|send|draft>"
---

# gog CLI — Gmail Reference

`gog` is a Google CLI (v0.9.0) covering Gmail, Calendar, Drive, and more. This guide focuses on Gmail operations.

## Prerequisites

### Authentication

```bash
# List stored accounts
gog auth list

# Add a new account
gog auth add user@example.com
```

### Account Selection

Every command requires `--account` unless exactly one token is stored or a default is set:

```bash
gog gmail search "from:alice" --account=user@gmail.com
```

### Output Formats

| Flag | Description |
|------|-------------|
| `--json` | Full JSON output (best for scripting/parsing) |
| `--plain` | Stable TSV output (no colors) |
| *(default)* | Human-readable colored output |

Always use `--json` when you need to parse the output programmatically.

## Search

### Search threads

```bash
gog gmail search "<query>" --account=EMAIL [--max=10] [--json]
```

Uses Gmail query syntax:

| Query | Meaning |
|-------|---------|
| `from:alice@example.com` | From a specific sender |
| `to:me subject:invoice` | To me with subject containing "invoice" |
| `has:attachment filename:pdf` | Has PDF attachment |
| `after:2026/01/01 before:2026/02/01` | Date range |
| `in:inbox is:unread` | Unread inbox messages |
| `label:work` | Messages with label |

Options:
- `--max=N` — Max results (default 10)
- `--oldest` — Show first message date instead of last

### Search messages (not threads)

```bash
gog gmail messages search "<query>" --account=EMAIL [--max=10] [--include-body] [--json]
```

Options:
- `--include-body` — Include decoded message body

## Read

### Get a single message

```bash
gog gmail get <messageId> --account=EMAIL [--format=full] [--json]
```

Format options: `full` (default), `metadata`, `raw`.

The JSON response structure:

```
{
  "headers": {           # dict: subject, from, to, cc, date, etc.
    "subject": "...",
    "from": "...",
    "to": "...",
    "date": "..."
  },
  "body": "...",          # string: decoded plain text body
  "attachments": [        # array of attachment objects
    {
      "filename": "doc.pdf",
      "size": 12345,
      "sizeHuman": "12.1 KB",
      "mimeType": "application/pdf",
      "attachmentId": "ANGjdJ9..."
    }
  ],
  "message": { ... }      # raw Gmail API message object
}
```

### Get a thread (all messages)

```bash
gog gmail thread get <threadId> --account=EMAIL [--full] [--download] [--out-dir=DIR] [--json]
```

Options:
- `--full` — Show full message bodies
- `--download` — Download all attachments
- `--out-dir=DIR` — Directory for downloaded attachments (default: current dir)

### List attachments in a thread

```bash
gog gmail thread attachments <threadId> --account=EMAIL [--download] [--out-dir=DIR] [--json]
```

## Download Attachments

### Single attachment

```bash
gog gmail attachment <messageId> <attachmentId> --account=EMAIL [--out=PATH]
```

Options:
- `--out=PATH` — Output file path (default: gog config dir)

### Workflow: find and download attachments

```bash
# 1. Get message with --json to find attachment IDs
gog gmail get <messageId> --account=EMAIL --json

# 2. Parse attachments array — filter out image signatures (image001.png etc.)

# 3. Download each real attachment
gog gmail attachment <messageId> <attachmentId> --account=EMAIL --out="/path/to/file.pdf"
```

### Bulk download from thread

```bash
gog gmail thread attachments <threadId> --account=EMAIL --download --out-dir="/path/to/dir"
```

## Send

### Send an email

```bash
gog gmail send --account=EMAIL \
  --to="recipient@example.com" \
  --subject="Subject line" \
  --body="Message body"
```

Options:
- `--cc=STRING` — CC recipients (comma-separated)
- `--bcc=STRING` — BCC recipients (comma-separated)
- `--body-file=PATH` — Read body from file (`-` for stdin)
- `--body-html=STRING` — HTML body
- `--attach=PATH` — Attach file (repeatable)
- `--from=STRING` — Send-as alias

### Reply to a message

```bash
gog gmail send --account=EMAIL \
  --reply-to-message-id=<messageId> \
  --reply-all \
  --body="Reply body"
```

- `--reply-to-message-id` — Sets In-Reply-To/References headers and thread
- `--thread-id` — Reply within a thread (uses latest message for headers)
- `--reply-all` — Auto-populate recipients from original message

## Drafts

```bash
# List drafts
gog gmail drafts list --account=EMAIL [--json]

# Create a draft
gog gmail drafts create --account=EMAIL \
  --to="recipient@example.com" \
  --subject="Subject" \
  --body="Draft body" \
  [--attach=PATH]

# Create a reply draft
gog gmail drafts create --account=EMAIL \
  --reply-to-message-id=<messageId> \
  --body="Reply draft"

# Get draft details
gog gmail drafts get <draftId> --account=EMAIL [--json]

# Update a draft
gog gmail drafts update <draftId> --account=EMAIL [--body="Updated body"]

# Send a draft
gog gmail drafts send <draftId> --account=EMAIL

# Delete a draft
gog gmail drafts delete <draftId> --account=EMAIL
```

## Labels

```bash
# List all labels
gog gmail labels list --account=EMAIL

# Get label details (including counts)
gog gmail labels get <labelIdOrName> --account=EMAIL

# Create a label
gog gmail labels create <name> --account=EMAIL

# Modify labels on threads
gog gmail labels modify <threadId> ... --account=EMAIL [--add-labels=STRING] [--remove-labels=STRING]
```

## Batch Operations

```bash
# Modify labels on multiple messages
gog gmail batch modify <messageId> ... --account=EMAIL [--add-labels=STRING] [--remove-labels=STRING]

# Permanently delete multiple messages (destructive!)
gog gmail batch delete <messageId> ... --account=EMAIL --force
```

## Common Patterns

### Save an email thread to local files

```bash
# 1. Search for the thread
gog gmail search "subject:invoice from:vendor" --account=EMAIL --json

# 2. Get thread with full bodies
gog gmail thread get <threadId> --account=EMAIL --full --json > thread.json

# 3. Download all attachments
gog gmail thread attachments <threadId> --account=EMAIL --download --out-dir=./attachments
```

### Read a specific message and download its attachments

```bash
# 1. Get message JSON
MSG=$(gog gmail get <messageId> --account=EMAIL --json)

# 2. Extract non-image attachment IDs (skip email signature images)
echo "$MSG" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for a in data.get('attachments', []):
    if not a['filename'].startswith('image'):
        print(f\"{a['filename']}\t{a['attachmentId']}\")
"

# 3. Download each attachment
gog gmail attachment <messageId> <attachmentId> --account=EMAIL --out="./filename.pdf"
```

### Reply to the latest message in a thread

```bash
gog gmail send --account=EMAIL \
  --thread-id=<threadId> \
  --reply-all \
  --body="Thanks, received."
```

## Tips

- Gmail message/thread IDs are hex strings (e.g., `19c8023ff8229cd2`)
- Thread search returns threads; message search returns individual messages
- Email signatures often appear as `image001.png`, `image002.png` etc. — filter these when processing attachments
- Use `--json` output and parse with `python3` or `jq` for reliable scripting
- Large attachments may be sent as external links (e.g., Daum/Naver bigfile) — these won't appear in the Gmail attachment list
- The `body` field in JSON output is the decoded plain text; for HTML, check `message.payload`
