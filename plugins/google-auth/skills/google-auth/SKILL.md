---
name: google-auth
description: Get OAuth access tokens for Google APIs — Sheets, Drive, Docs, Gmail, and more. Use this skill whenever a task involves reading or writing Google Sheets, listing or downloading files from Google Drive, reading or editing Google Docs, or reading Gmail. If a task requires calling any Google API, use this skill to get a valid token first.
---

# Google Auth

Provides a valid Google OAuth access token. The bundled script handles credential setup, browser-based authentication, token caching, and automatic refresh. The token grants access to Sheets, Drive, Docs, and Gmail (read-only).

## Getting a token

```bash
TOKEN=$(python3 skills/google-auth/scripts/token.py)
```

The `token.py` script is bundled with this plugin at `skills/google-auth/scripts/token.py`.

The token is printed to stdout. All prompts and status messages go to stderr, so `$TOKEN` will always be just the access token string.

### First run

On first run, the script requires user interaction:

1. It prints a link to a shared Google Drive file containing the OAuth client credentials
2. The user pastes the Client ID and Client Secret
3. It opens the user's browser for Google sign-in
4. After sign-in, tokens are saved to `~/.oauth-store/`

**Before running `token.py` for the first time**, tell the user something like:

> I need to set up Google API access. This is a one-time setup — you'll need to paste some credentials and sign in with Google in your browser. Ready?

Wait for the user to confirm before proceeding.

### Subsequent runs

The script returns immediately with a cached token. If the token is expired, it refreshes automatically via the saved refresh token — no user interaction needed.

## Using the token

Fetch a fresh token before each API call (the script handles caching internally):

```bash
TOKEN=$(python3 skills/google-auth/scripts/token.py)
```

---

## Google Sheets API

### Read cell values

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://sheets.googleapis.com/v4/spreadsheets/SPREADSHEET_ID/values/RANGE"
```

- `SPREADSHEET_ID` is the long ID in the sheet URL: `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`
- `RANGE` uses A1 notation, e.g. `Sheet1!A1:D10` or just `Sheet1` for all data

### Get spreadsheet metadata

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://sheets.googleapis.com/v4/spreadsheets/SPREADSHEET_ID?fields=sheets.properties"
```

### Write cell values

```bash
curl -s -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://sheets.googleapis.com/v4/spreadsheets/SPREADSHEET_ID/values/RANGE?valueInputOption=USER_ENTERED" \
  -d '{"values": [["row1col1", "row1col2"], ["row2col1", "row2col2"]]}'
```

### Append rows

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://sheets.googleapis.com/v4/spreadsheets/SPREADSHEET_ID/values/RANGE:append?valueInputOption=USER_ENTERED" \
  -d '{"values": [["val1", "val2"]]}'
```

---

## Google Drive API

### List files

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://www.googleapis.com/drive/v3/files?q=name%20contains%20'report'&fields=files(id,name,mimeType)"
```

Common query filters: `mimeType='application/vnd.google-apps.spreadsheet'`, `'FOLDER_ID' in parents`, `trashed=false`.

### Download a file

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://www.googleapis.com/drive/v3/files/FILE_ID?alt=media" -o output.dat
```

For Google-native formats (Docs, Sheets), export instead:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://www.googleapis.com/drive/v3/files/FILE_ID/export?mimeType=text/csv" -o output.csv
```

### Upload a file

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart" \
  -F "metadata={\"name\": \"report.csv\"};type=application/json" \
  -F "file=@report.csv;type=text/csv"
```

---

## Google Docs API

### Get document content

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://docs.googleapis.com/v1/documents/DOCUMENT_ID"
```

The `DOCUMENT_ID` is in the doc URL: `https://docs.google.com/document/d/DOCUMENT_ID/edit`

### Get plain text only

The Docs API returns structured JSON. To extract plain text, parse the `body.content` array for `textRun` elements, or export via Drive:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://www.googleapis.com/drive/v3/files/DOCUMENT_ID/export?mimeType=text/plain"
```

---

## Gmail API (read-only)

### List recent messages

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://gmail.googleapis.com/gmail/v1/users/me/messages?maxResults=10"
```

### Read a message

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://gmail.googleapis.com/gmail/v1/users/me/messages/MESSAGE_ID?format=full"
```

### Search messages

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://gmail.googleapis.com/gmail/v1/users/me/messages?q=from:someone@example.com+after:2025/01/01"
```

---

## Troubleshooting

If the token command fails with a refresh error, the saved tokens may be stale. Delete them and re-authenticate:

```bash
rm ~/.oauth-store/tokens.json
TOKEN=$(python3 skills/google-auth/scripts/token.py)
```

This will re-trigger the browser sign-in flow (requires user interaction).
