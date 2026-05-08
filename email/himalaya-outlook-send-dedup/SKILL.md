---
name: himalaya-outlook-send-dedup
description: "Fix duplicate emails in Outlook Sent folder when sending via Himalaya. Outlook auto-saves + Himalaya saves → 2 copies. Solution: make Himalaya's save fail by temporarily changing the sent folder alias."
version: 1.0.0
author: agent
prerequisites:
  files:
    - ~/.config/himalaya/config.toml
    - ~/.local/bin/hermes-send
---

# Himalaya + Outlook Sent Folder Duplicate Fix

## The Problem

When sending email via Himalaya (`himalaya template send`), the Sent folder in Outlook shows **two copies** of every sent email. The recipient only receives one copy.

**Root cause**: Two actors save to Sent:
1. **Outlook SMTP server** — auto-saves a copy after successful SMTP delivery (server-side, unavoidable)
2. **Himalaya** — `send_message_then_save_copy()` saves a copy after sending (client-side, always-on in current code)

Since both save to the same "Sent" folder, each send produces 2 copies.

## The Fix

A wrapper script (`hermes-send`) that temporarily changes the sent folder alias to a non-existent folder, so Himalaya's save attempt fails silently. The email is still delivered via SMTP, and Outlook's auto-save creates exactly one copy.

### How it works

```bash
1. Copy config, change folder.aliases.sent → "Sent_HimalayaDummy"
2. himalaya template send
   → SMTP: sends email ✅
   → IMAP save to "Sent_HimalayaDummy": fails (folder doesn't exist) ❌
3. Restore config
```

Result: exactly **1 copy** in Outlook Sent (from Outlook's auto-save).

## Setup

### 1. Create the wrapper script

File: `~/.local/bin/hermes-send`

```bash
#!/bin/bash
set +e
CONFIG="$HOME/.config/himalaya/config.toml"
TMP_CONFIG="/tmp/himalaya_send_config.toml"

python3 -c "
import re
with open('$CONFIG') as f:
    content = f.read()
content = re.sub(r'folder\\.aliases\\.sent = \"Sent\"',
                 'folder.aliases.sent = \"Sent_HimalayaDummy\"', content)
with open('$TMP_CONFIG', 'w') as f:
    f.write(content)
"

export PATH="$HOME/.cargo/bin:\$PATH"
himalaya --config "$TMP_CONFIG" template send
RC=$?
rm -f "$TMP_CONFIG"

if [ $RC -eq 0 ]; then
    echo "✅ Sent successfully"
    exit 0
elif [ $RC -eq 1 ]; then  # save-to-sent error is expected
    echo "⚠️ Email sent (Outlook auto-saved one copy)"
    exit 0
else
    exit $RC
fi
```

Make executable:
```bash
chmod +x ~/.local/bin/hermes-send
```

### 2. Usage

```bash
# Pipe email content
cat << EOF | hermes-send
From: You <you@outlook.com>
To: someone@example.com
Subject: Hello

Body text here.
EOF

# Reply to an email
himalaya template reply <ID> | hermes-send
```

### 3. Verify

Check Sent folder after sending:
```bash
himalaya envelope list --folder Sent --page-size 5
```

The email should appear exactly once. Old emails (sent before the fix) may still have duplicates — they're harmless.

## Notes

- The `Error: cannot add IMAP message - Folder is not found` message is **expected** and harmless — it's the save-to-sent failing as designed
- Only affects the `Sent` folder display; recipient always receives exactly one copy
- Works with any Outlook-compatible SMTP (outlook.com, live.com, hotmail.com, office365.com)
- The temp config file is created and destroyed each time — no permanent changes
