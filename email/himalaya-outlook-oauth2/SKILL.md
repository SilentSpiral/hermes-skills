---
name: himalaya-outlook-oauth2
description: "Setup Himalaya CLI with Outlook/Office365 via OAuth2 Device Code Flow. Uses Thunderbird's public client ID — no Azure app registration needed."
version: 1.1.0
author: agent
prerequisites:
  commands: [cargo, himalaya]
---

# Himalaya + Outlook OAuth2 Setup

Microsoft has deprecated basic auth for IMAP/SMTP. This skill configures Himalaya with OAuth2.

## Prerequisites

- Rust/Cargo installed (`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y`)
- An Outlook/Hotmail/Live/Office365 email account
- User must have a browser to complete OAuth authorization

## Step-by-step

### 1. Install Himalaya with OAuth2 Feature

Pre-built binaries do NOT include OAuth2. Must compile from source:

```bash
source "$HOME/.cargo/env"
cargo install himalaya --locked --features imap,maildir,smtp,sendmail,wizard,pgp-commands,oauth2
```

Verify: `himalaya --version` should show `+oauth2` in features.

### 2. Create Config

`~/.config/himalaya/config.toml`:

```toml
[accounts.default]
email = "your@outlook.com"
display-name = "Your Name"
default = true

# IMAP
backend.type = "imap"
backend.host = "outlook.office365.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "your@outlook.com"
backend.auth.type = "oauth2"
backend.auth.method = "xoauth2"
backend.auth.client-id = "9e5f94bc-e8a4-4e73-b8be-63364c29d753"
backend.auth.access-token.cmd = "sh $HOME/.config/himalaya/access_token.sh"
backend.auth.auth-url = "https://login.microsoftonline.com/common/oauth2/v2.0/authorize"
backend.auth.token-url = "https://login.microsoftonline.com/common/oauth2/v2.0/token"
backend.auth.pkce = false
backend.auth.scopes = ["offline_access", "https://outlook.office.com/IMAP.AccessAsUser.All"]

# SMTP
message.send.backend.type = "smtp"
message.send.backend.host = "smtp-mail.outlook.com"
message.send.backend.port = 587
message.send.backend.login = "your@outlook.com"
message.send.backend.encryption.type = "start-tls"
message.send.backend.auth.type = "oauth2"
message.send.backend.auth.method = "xoauth2"
message.send.backend.auth.client-id = "9e5f94bc-e8a4-4e73-b8be-63364c29d753"
message.send.backend.auth.access-token.cmd = "sh $HOME/.config/himalaya/access_token.sh"
message.send.backend.auth.auth-url = "https://login.microsoftonline.com/common/oauth2/v2.0/authorize"
message.send.backend.auth.token-url = "https://login.microsoftonline.com/common/oauth2/v2.0/token"
message.send.backend.auth.pkce = false
message.send.backend.auth.scopes = ["https://outlook.office.com/IMAP.AccessAsUser.All", "https://outlook.office.com/SMTP.Send"]

folder.aliases.inbox = "INBOX"
folder.aliases.sent = "Sent"
folder.aliases.drafts = "Drafts"
folder.aliases.trash = "Deleted Items"
```

> `client-id` `9e5f94bc-e8a4-4e73-b8be-63364c29d753` is Thunderbird's public registered app ID — works for any Microsoft personal/organizational account without registering your own Azure app.

### 3. Get Tokens via Device Code Flow

**⚠️ CRITICAL**: The device_code is SINGLE-USE. After polling succeeds once, you CANNOT reuse it. If anything goes wrong during saving, you MUST generate a fresh code and have the user re-authenticate.

```bash
# Step A: Generate device code
D=$(curl -s -X POST "https://login.microsoftonline.com/common/oauth2/v2.0/devicecode" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=9e5f94bc-e8a4-4e73-b8be-63364c29d753" \
  -d "scope=https://outlook.office.com/IMAP.AccessAsUser.All%20https://outlook.office.com/SMTP.Send%20offline_access")

SAVE the full response immediately to a temp file:
echo "$D" > /tmp/ms_device.json

Extract and show the code:
python3 -c "import json; d=json.load(open('/tmp/ms_device.json')); print(f'Code: {d[\"user_code\"]}\nURL: {d[\"verification_uri\"]}')"
```

Ask the user to open the URL in their browser, enter the code, and authenticate with their Microsoft account.

```bash
# Step B: Poll for tokens (run AFTER user confirms authentication)
# Use the SAME device_code from step A — do NOT generate a new one!
DEVICE_CODE=$(python3 -c "import json; d=json.load(open('/tmp/ms_device.json')); print(d['device_code'])")

# ⚠️ Save directly to file, don't pipe to python for display (token truncation issue)
curl -s -o /tmp/ms_token.json -X POST "https://login.microsoftonline.com/common/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=urn:ietf:params:oauth:grant-type:device_code" \
  -d "client_id=9e5f94bc-e8a4-4e73-b8be-63364c29d753" \
  -d "device_code=$DEVICE_CODE"

# Save refresh_token
python3 -c "
import json, os
d = json.load(open('/tmp/ms_token.json'))
rt_path = os.path.expanduser('~/.config/himalaya/refresh_token.txt')
if 'refresh_token' in d:
    with open(rt_path, 'w') as f: f.write(d['refresh_token'])
    print(f'Saved refresh_token ({len(d[\"refresh_token\"])} chars)')
else:
    print(f'ERROR: {d.get(\"error\", \"unknown\")} - {d.get(\"error_description\", \"\")}')
"
```

### 4. Create Access Token Refresh Script

`~/.config/himalaya/access_token.sh`:

```bash
#!/bin/sh
REFRESH_TOKEN=$(cat $HOME/.config/himalaya/refresh_token.txt)
curl -s -X POST "https://login.microsoftonline.com/common/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=9e5f94bc-e8a4-4e73-b8be-63364c29d753" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=$REFRESH_TOKEN" \
  -d "scope=https://outlook.office.com/IMAP.AccessAsUser.All https://outlook.office.com/SMTP.Send" \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])"
```

```bash
chmod +x ~/.config/himalaya/access_token.sh
```

### 5. Test

```bash
export PATH="$HOME/.cargo/bin:$PATH"
himalaya folder list
himalaya envelope list --page-size 5
```

## Pitfalls & Lessons Learned

### Device Code is Single-Use
The MOST COMMON failure mode. After `curl -o /tmp/ms_token.json` succeeds once, the device_code is consumed. If you try again with the same code you get `AADSTS70000: device_code has already been used`.

**Solution**: If token saving fails on first attempt, generate a FRESH device code by re-running step A. Don't try to re-poll the same code.

### Token Output Truncation
When piping curl output through `python3 -c "..."`, long access tokens get truncated in display output. Always use `curl -o <file>` to save raw JSON first, then parse from the file.

### User Authentication Timing
The device code expires after 900 seconds (15 min). If the user takes too long, generate a new code. The polling interval is 5 seconds.

### User Preference: Step-by-Step Transparency
When running this flow with a human user:
- Explain each step BEFORE executing it
- Save intermediate results to files (not just variables)
- Show what was saved/created after each step
- If something fails, explain WHY before retrying

### PATH Issues
Himalaya installed via cargo goes to `~/.cargo/bin/himalaya`. The pre-built binary at `~/.local/bin/himalaya` may lack OAuth2 support. Always verify with `himalaya --version | grep oauth2`.

### GBK Encoding (WSL2)
When integrating with Windows-native tools from WSL2, output is in Windows GBK encoding. Pipe through `iconv -f GBK -t UTF-8` to display Chinese correctly.

## References

- `references/outlook-imap-smtp-settings.md` — Outlook server endpoints and port details
