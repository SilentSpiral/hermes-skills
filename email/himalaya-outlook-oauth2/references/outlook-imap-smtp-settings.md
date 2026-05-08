# Outlook/Office365 IMAP & SMTP Settings

## Server Endpoints

| Protocol | Server | Port | Encryption |
|----------|--------|------|------------|
| IMAP | outlook.office365.com | 993 | TLS |
| SMTP | smtp-mail.outlook.com | 587 | STARTTLS |

## Auth Requirements

- **Basic auth (password)**: DEPRECATED by Microsoft since 2022. Most accounts reject it with `5.7.139 Authentication unsuccessful, basic authentication is disabled`.
- **App passwords**: Only work if the account has 2FA enabled. Even then, some accounts reject them as basic auth.
- **OAuth2 (modern auth)**: The ONLY reliable method for Outlook/Hotmail/Live/Office365 accounts in 2025+.

## Device Code OAuth2 Flow

Uses Thunderbird's public client ID (`9e5f94bc-e8a4-4e73-b8be-63364c29d753`) — no Azure app registration needed.

### Scopes Required

- `https://outlook.office.com/IMAP.AccessAsUser.All` — read mail
- `https://outlook.office.com/SMTP.Send` — send mail
- `offline_access` — get refresh token for persistent sessions

### Auth Endpoints

- Authorize: `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`
- Token: `https://login.microsoftonline.com/common/oauth2/v2.0/token`
- Device code: `https://login.microsoftonline.com/common/oauth2/v2.0/devicecode`

## Common Error Messages

| Error | Meaning | Fix |
|-------|---------|-----|
| `AUTHENTICATE failed` on PLAIN | Basic auth rejected | Switch to OAuth2 |
| `5.7.139 Authentication unsuccessful, basic authentication is disabled` | Microsoft explicitly blocks basic auth | Use OAuth2 |
| `AADSTS70000: device_code has already been used` | Device code is single-use | Generate fresh code + re-authenticate |
| `AADSTS7000014: device_code is not valid` | Device code expired (15min TTL) | Generate fresh code |

## Folder Name Mappings

| Himalaya canonical | Outlook server |
|--------------------|----------------|
| INBOX | INBOX |
| Sent | Sent |
| Drafts | Drafts |
| Trash | Deleted Items |
