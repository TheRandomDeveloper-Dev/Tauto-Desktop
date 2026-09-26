# Tauto — Telegram Auto Poster

Electron port of the Python Telethon-based auto-poster. Same behavior model
(human-simulation reads, mark-as-read, weighted delays, revisits) but with a
proper UI and multi-account session management.

## Stack

- Electron 32 (main + preload + renderer, context-isolated)
- [GramJS](https://github.com/gram-js/gramjs) (`telegram` npm package) — the
  Node.js equivalent of Telethon
- `qrcode` for on-screen QR generation

## Install & run

```bash
cd tauto
npm install
npm start
```

Dev mode (opens DevTools):

```bash
npm run dev
```

Build installers:

```bash
npm run build:win     # NSIS installer
npm run build:mac     # DMG
npm run build:linux   # AppImage
```

## First-run flow

1. **Settings → Telegram API credentials.** Optional. Leave blank to use the
   bundled fallback (`API_ID = 33727858`, `API_HASH = 435e2b…4bb0e`). If you
   provide your own from `my.telegram.org`, they take precedence and are used
   for every subsequent session.
2. **Accounts → Add account.** QR (recommended) or phone code, both with 2FA
   support. Session strings are saved as `<phone>.session` under the OS user
   data directory (`%APPDATA%/Tauto/sessions` on Windows,
   `~/Library/Application Support/Tauto/sessions` on macOS,
   `~/.config/Tauto/sessions` on Linux).
3. **Messages.** Create named templates. Each has a type (`text` or `photo`),
   Markdown body/caption, and an image path/URL for `photo` templates.
4. **Groups → Fetch from Telegram.** Loads every supergroup + legacy chat the
   active account is in. For each group: tick to enable, pick a template, set
   a topic ID if the group is a forum. Click **Save selection** to persist.
5. **Auto Post → Start.** Runs the loop. Stop is honored between actions.

## Config file

Everything except sessions lives in `<userData>/config.json`. Safe to edit by
hand; the app reloads it on start.

```json
{
  "apiId": "",
  "apiHash": "",
  "accessCheckDisabled": false,
  "messages": {
    "default": { "type": "text", "text": "Hello 👋 from Tauto." }
  },
  "groups": {
    "-1001234567890": { "name": "My Group", "type": "default", "topic": null }
  },
  "posterDefaults": {
    "shortDelayMin": 20, "shortDelayMax": 60,
    "mediumDelayMin": 60, "mediumDelayMax": 180,
    "longDelayMin": 300, "longDelayMax": 600,
    "shortWeight": 0.6, "mediumWeight": 0.3,
    "revisitProbability": 0.25,
    "humanBehavior": true,
    "shuffle": true
  }
}
```

## Access gate

The original script pings a GitHub raw file for an `allowed_to_jay` marker.
That's preserved in `src/access.js`. Two ways to bypass:

- Toggle **Settings → Access gate → Disable remote access check**, or
- Set `"accessCheckDisabled": true` in `config.json`, or
- Delete `src/access.js` and the `access:check` handler in `main.js` if you
  don't want it in the app at all.

## Behavior parity with the Python version

| Python                                      | Tauto                                          |
|---------------------------------------------|------------------------------------------------|
| `TelegramClient` (Telethon)                 | `TelegramClient` (GramJS)                      |
| `qr_login()` + ASCII QR in terminal         | `signInUserWithQrCode` + PNG QR in modal       |
| `client.start(phone=...)`                   | `client.start({ phoneNumber, phoneCode, ... })`|
| `get_dialogs()` → `Channel(megagroup)`/`Chat` | `getDialogs()` with same classification      |
| `send_message` / `send_file` with markdown  | `sendMessage` / `sendFile` `parseMode: "md"`   |
| `send_read_acknowledge`                     | `markAsRead`                                   |
| `human_behavior()` — read+idle+mark         | Identical, `src/telegram-service.js`           |
| Weighted delay 60/30/10                     | `AutoPoster._pickDelay()`                      |
| 25% revisit chance                          | `revisitProbability` in settings               |
| `messages.py` sibling file                  | In-app template editor persisted to config     |
| `groups.json`                               | In-app groups table persisted to config        |
| `logs.txt` next to the script               | `<userData>/logs.txt` + live pane in UI        |

## Notes

- **Sessions are portable.** Copy a `.session` file between machines and it
  logs in as that account, no code needed.
- **`markAsRead` failures are swallowed** — some groups refuse it, matching
  the Python `try/except: pass` behavior.
- The renderer never sees API credentials or session strings directly; all
  Telegram calls happen in the main process behind IPC.
- CSP is set to `default-src 'self'` with `data:` allowed for images so QR
  PNGs render inline without exposing renderer to network.