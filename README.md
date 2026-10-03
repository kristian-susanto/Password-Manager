# Umbra — Zero-Knowledge Password Manager

> A fully client-side, file-based password manager. Every field is sealed with AES-256-GCM using a key derived from your master password via PBKDF2-SHA256. Nothing is uploaded, synced, or tracked — the entire vault lives in a single encrypted CSV or JSON file that you own.

---

## Table of Contents

1. [Overview](#overview)
2. [Security Model](#security-model)
3. [Vault File Format](#vault-file-format)
4. [Getting Started](#getting-started)
5. [Features](#features)
6. [UI Reference](#ui-reference)
7. [Keyboard & Interaction Behaviors](#keyboard--interaction-behaviors)
8. [Architecture](#architecture)
9. [Storage & Session](#storage--session)
10. [Browser Requirements](#browser-requirements)
11. [Limitations & Caveats](#limitations--caveats)
12. [Glossary](#glossary)

---

## Overview

**Umbra** is a single-file (`index.html`) password manager that runs entirely inside your browser tab. It stores credentials in an encrypted file — **CSV** or **JSON** — that you download, back up, and re-open manually. There is:

- No backend, no server, no API.
- No network requests, no analytics, no cookies.
- No cloud sync, no account, no email.

Everything is derived from a **master password** you choose.

---

## Security Model

### Zero-Knowledge by Design

| Aspect                  | Detail                                                                |
| ----------------------- | --------------------------------------------------------------------- |
| Key derivation          | PBKDF2-SHA256                                                         |
| Iterations              | 310,000 (default)                                                     |
| Salt                    | 16 random bytes per vault, stored in plaintext in the file            |
| Cipher                  | AES-256-GCM                                                           |
| IV                      | 12 random bytes per encryption operation                              |
| Master password storage | **Never stored** — not even hashed                                    |
| Verifier                | An encrypted `{ v: 1, check: "UMBRA-CHECK-v1" }` blob in the meta row |

### How Unlock Works

1. The vault file's plaintext `salt` and `iterations` are read.
2. The master password is passed through PBKDF2 to derive an AES-GCM key.
3. The **verifier blob** (stored in the `meta` row) is decrypted.
   - If it decrypts to `{ check: "UMBRA-CHECK-v1" }`, the password is correct.
   - Otherwise, decryption fails (AES-GCM authentication tag mismatch) and an error is shown.
4. Each entry's ciphertext is decrypted individually. Any rows that fail to decrypt are counted as **skipped**.

### Threat Model

**Protects against:**

- Anyone who obtains your vault file without the master password.
- Tampering — AES-GCM rejects modified ciphertext.
- Network eavesdropping (nothing is ever transmitted).

**Does NOT protect against:**

- Malware on your device (keyloggers, memory scrapers).
- A weak master password (PBKDF2 slows, but does not stop, brute force).
- Someone who sees your screen while the vault is unlocked.

---

## Vault File Format

Umbra supports two interchangeable formats. Both wrap the same ciphertext payloads; only the container differs.

### CSV Format

```csv
type,id,updated,iv,data,salt,iterations
meta,,,<b64-iv>,<b64-verifier>,<b64-salt>,310000
entry,<uuid>,<iso8601>,<b64-iv>,<b64-ct>,,
entry,<uuid>,<iso8601>,<b64-iv>,<b64-ct>,,
```

- `type` — `meta` (one row) or `entry` (zero or more rows).
- `iv` — base64-encoded 12-byte AES-GCM initialization vector.
- `data` — base64-encoded ciphertext (includes GCM auth tag).
- `salt` / `iterations` — only present on the `meta` row.
- Fields containing `"`, `,`, or newlines are quoted per RFC 4180.

### JSON Format

```json
{
  "format": "umbra",
  "version": 1,
  "meta": {
    "salt": "<b64-salt>",
    "iterations": 310000,
    "iv": "<b64-iv>",
    "data": "<b64-verifier-ct>"
  },
  "entries": [
    {
      "id": "<uuid>",
      "updated": "<iso8601>",
      "iv": "<b64-iv>",
      "data": "<b64-ct>"
    }
  ]
}
```

### Entry Payload (decrypted)

Each entry's plaintext, once decrypted, is a JSON object:

```json
{
  "title": "GitHub",
  "username": "you@example.com",
  "password": "correct-horse-battery-staple",
  "url": "https://github.com",
  "notes": "2FA recovery codes in 1Password",
  "customFields": [
    { "id": "<uuid>", "type": "text", "label": "PIN", "value": "1234" },
    { "id": "<uuid>", "type": "hidden", "label": "API Key", "value": "sk-…" },
    { "id": "<uuid>", "type": "checkbox", "label": "Has 2FA", "value": true },
    {
      "id": "<uuid>",
      "type": "linked",
      "label": "Admin URL",
      "value": "https://admin.example.com"
    }
  ]
}
```

---

## Getting Started

### Create a New Vault

1. Open `index.html` in a modern browser.
2. In the **Create a new vault** card:
   - Enter a master password (≥ 8 characters).
   - Confirm it.
   - _(Optional)_ Click the generate icon for a strong password, or the copy icon to copy it.
   - _(Optional)_ Watch the strength bar to gauge quality.
3. Click **Create encrypted vault**.
4. You are taken to the vault screen. **Immediately download the encrypted file** via the download icon in the toolbar — nothing is persisted on disk otherwise.

### Open an Existing Vault

1. In the **Open an existing vault** card, drag-and-drop your `.csv` or `.json` vault onto the drop zone (or click it to browse).
2. On the unlock screen, enter your master password.
3. Click **Unlock vault** (or press <kbd>Enter</kbd>).

### Save the Vault

Click the **download icon** in the top toolbar. A modal lets you pick **CSV** or **JSON**. The file extension of your current vault name determines the default suggestion.

> **Always re-download after changing the master password** — the derived key changes, so the old file will only open with the old password.

---

## Features

### Credential Management

- Add, edit, view, and delete entries.
- Bulk select via the per-row checkbox, the select-all pill, or the **selection action bar** at the bottom.
- Bulk delete selected entries with confirmation.
- Search across `title`, `username`, `url`, `notes`, and custom fields.
- Entries are auto-sorted alphabetically by title (case-insensitive).

### Entry Fields

| Field            | Notes                                                                      |
| ---------------- | -------------------------------------------------------------------------- |
| Title            | Required, max 120 chars                                                    |
| Username / Email | Max 200 chars                                                              |
| Password         | Max 512 chars, masked by default with reveal toggle                        |
| Website URL      | Max 300 chars; auto-prefixed with `https://` when opened if scheme missing |
| Notes            | Max 2000 chars, multi-line                                                 |
| Custom fields    | Up to any number; see below                                                |

### Custom Fields

Four types are supported, each rendered differently in the **detail modal**:

| Type       | Editor       | Detail view                             |
| ---------- | ------------ | --------------------------------------- |
| `text`     | Plain input  | Plain text                              |
| `hidden`   | Masked input | Masked with reveal toggle + copy button |
| `checkbox` | Checkbox     | `✓ Yes` / `✗ No`                        |
| `linked`   | URL input    | Clickable hyperlink                     |

### Password Generator

Two modes, available from the toolbar, entry form, and master-password fields:

**Random Password**

- Length: 8–128 (default 20)
- Toggles: uppercase, lowercase, digits, symbols
- Minimum digits (0–9), minimum symbols (0–9)
- Option: avoid ambiguous characters (`l`, `I`, `O`, `0`, `1`)
- Guarantees at least one character from each enabled set, then Fisher–Yates shuffles.

**Memorable Passphrase**

- 2–20 words (default 3)
- Custom separator (max 3 chars, default `-`)
- Optional capitalization of each word
- Word list of ~350 English nouns (nature/animal themes).

### Username Generator

| Type                 | Output example          |
| -------------------- | ----------------------- |
| Random Word          | `Lantern42`             |
| Plus Addressed Email | `user+a1b2c3@gmail.com` |
| Catch-All Email      | `x7k9m2p4@mydomain.com` |

### File Operations

- **Download as CSV or JSON** — via the save-format modal.
- **Rename vault file** — the extension (`.csv` / `.json`) drives the save format.
  - Quick random name generator (e.g. `Comet482.json`).
  - Username generator can be used as a filename base.
- **Change master password** — generates a new salt and key, sets vault to "unsaved changes."

### Vault Locking

- **Lock vault** — encrypts the vault, keeps the session, and immediately drops you on the unlock screen for the same file.
- **Log out** — clears the vault from memory entirely and returns to the welcome screen. Warns if you have unsaved changes.
- **Auto-lock** — 5 minutes of inactivity triggers a silent logout with a toast.
- **beforeunload guard** — browser warns if you have unsaved changes and try to close the tab.

### Clipboard Safety

- Copy actions auto-clear the clipboard after **30 seconds**.
- Copy button provides a fallback via `document.execCommand("copy")` for contexts where the Clipboard API is blocked.

### Theme

- Dark and light themes, toggled via the sun/moon button.
- Selection is remembered in `localStorage` under `umbra-theme`.
- Defaults to your OS preference via `prefers-color-scheme`.

---

## UI Reference

### Screens

| Screen            | Purpose                                 |
| ----------------- | --------------------------------------- |
| `#screen-welcome` | Create or open a vault                  |
| `#screen-unlock`  | Enter master password for a loaded file |
| `#screen-vault`   | Main vault interface                    |

### Toolbar Buttons (Vault Screen)

| Icon               | Action                                 |
| ------------------ | -------------------------------------- |
| **Add**            | Open the Add Entry modal               |
| Username generator | Open username generator (standalone)   |
| Password generator | Open password generator (standalone)   |
| Download           | Open the save-format modal             |
| Rename             | Rename the vault file                  |
| Change password    | Change the master password             |
| Theme toggle       | Switch light/dark                      |
| Lock vault         | Lock and jump to unlock screen         |
| Log out            | Clear everything and return to welcome |

### Modals

| ID                       | Purpose                                          |
| ------------------------ | ------------------------------------------------ |
| `#entry-modal`           | Add/edit an entry, including custom fields       |
| `#add-field-modal`       | Choose a new custom field's type                 |
| `#gen-modal`             | Password generator (random / passphrase)         |
| `#username-modal`        | Username generator                               |
| `#rename-modal`          | Rename vault file                                |
| `#change-password-modal` | Change master password                           |
| `#save-format-modal`     | Choose CSV or JSON on download                   |
| `#confirm-modal`         | Generic confirmation dialog                      |
| `#detail-modal`          | Read-only entry details with copy/reveal buttons |

---

## Keyboard & Interaction Behaviors

- <kbd>Enter</kbd> on the unlock password field triggers unlock.
- <kbd>Enter</kbd> on the rename field applies the new name.
- <kbd>Escape</kbd> closes any open modal.
- Clicking the modal backdrop (outside the modal card) closes the modal.
- Clicking the confirm modal backdrop resolves it as **cancel**.
- The floating selection action bar shifts the toast downward when visible.

---

## Architecture

The entire app is a single IIFE in `index.html`:

```
┌────────────────────────────────────────────────┐
│  UI layer                                       │
│   Screens:  welcome / unlock / vault            │
│   Modals:   entry, generator, confirm, …        │
├────────────────────────────────────────────────┤
│  State                                          │
│   state = { key, saltB64, iterations,           │
│             entries, fileName, dirty }          │
│   pending   – parsed file awaiting unlock       │
│   revealed  – set of decrypted entry IDs        │
│   selected  – set of selected entry IDs         │
├────────────────────────────────────────────────┤
│  Crypto (WebCrypto)                             │
│   deriveKey → PBKDF2-SHA256 → AES-GCM 256       │
│   encryptData / decryptData                     │
├────────────────────────────────────────────────┤
│  Serialization                                  │
│   buildCSV / buildJSON ↔ parseVaultCSV / JSON   │
├────────────────────────────────────────────────┤
│  Storage                                        │
│   sessionStorage: umbra-session-v1              │
│   localStorage:   umbra-theme                   │
└────────────────────────────────────────────────┘
```

### Key Functions

| Function                                  | Role                                                      |
| ----------------------------------------- | --------------------------------------------------------- |
| `deriveKey(password, salt, iterations)`   | PBKDF2 → AES-GCM key                                      |
| `encryptData(key, obj)`                   | Returns `{ iv, ct }` as base64                            |
| `decryptData(key, iv, ct)`                | Returns the parsed JSON object                            |
| `serializeVault(format)`                  | Builds CSV or JSON text                                   |
| `prepareVaultFromText(text, name, dirty)` | Parses a file into `pending`                              |
| `unlockVault()`                           | Derives key, verifies, decrypts entries                   |
| `persistSession()`                        | Debounced (300 ms) re-serialization into `sessionStorage` |
| `lockVault(silent)`                       | Full memory wipe + navigate to welcome                    |
| `lockVaultKeepFile()`                     | Wipe + immediately re-load same file into `pending`       |

---

## Storage & Session

| Key                | Storage          | Purpose                        |
| ------------------ | ---------------- | ------------------------------ |
| `umbra-theme`      | `localStorage`   | `"light"` or `"dark"`          |
| `umbra-session-v1` | `sessionStorage` | `{ text, fileName, wasDirty }` |

**Why `sessionStorage`?** It survives a page refresh (so you don't have to re-open the file), but it's discarded when the tab closes. Since the blob is still **encrypted**, anyone reading the storage key without the master password learns nothing.

The session is:

- Written on vault creation, unlock, rename, save, and every state mutation (debounced).
- Cleared on logout and when the user cancels the file picker.

---

## Browser Requirements

Umbra requires the **Web Crypto API** (`crypto.subtle`). It will refuse to start with a friendly alert if unavailable.

Supported environments:

- Modern Chrome, Edge, Firefox, Safari (recent versions).
- Any HTTPS origin, `localhost`, or `file://` in most browsers.

> Opening from `file://` works in Chrome/Edge/Firefox, but Safari may block the Web Crypto API in some configurations. Serving over HTTPS is the safest option.

### File Save Behavior

On browsers supporting the **File System Access API** (`showSaveFilePicker`), Umbra can detect when you press **Cancel** and will keep the vault marked as dirty. On other browsers, a plain anchor download is used, which cannot detect cancellation.

---

## Limitations & Caveats

1. **No password recovery.** If you lose your master password, the vault is unrecoverable. Period.
2. **No per-entry key rotation.** Changing the master password changes the key for **all** entries; re-download immediately.
3. **No history / versioning.** Deletes and edits are permanent on save.
4. **No metadata hiding.** The number of entries, their IDs, and timestamps are visible in the file even without the key.
5. **Memory exposure.** While unlocked, plaintext credentials live in the JavaScript heap. Do not run untrusted extensions or scripts alongside Umbra.
6. **Auto-lock is activity-based**, not visibility-based. Minimizing the tab does not reset the timer; if anything, closing it does.
7. **The clipboard-clear timer** runs only while the tab is open. If you close the tab before 30 seconds, the clipboard content remains.
8. **Session persistence in `sessionStorage`** means the encrypted blob is briefly written to disk (browser profile). It is still encrypted, but be aware.

---

## Glossary

| Term                | Meaning                                                                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Master password** | The single secret that unlocks the vault. Never stored.                                                                     |
| **Salt**            | Random bytes added to the master password before PBKDF2 to prevent rainbow-table attacks. Stored plainly in the vault file. |
| **PBKDF2**          | Password-Based Key Derivation Function 2 — deliberately slow, hardens against brute force.                                  |
| **AES-256-GCM**     | Symmetric cipher with authentication — tampering is detectable.                                                             |
| **IV**              | Initialization Vector — random, per-encryption, ensures identical plaintexts yield different ciphertexts.                   |
| **Verifier**        | A tiny encrypted blob whose successful decryption proves the master password is correct.                                    |
| **Zero-knowledge**  | The storing entity (in this case, you — the file itself) learns nothing about the plaintext.                                |

---

## Quick Reference Cheatsheet

```text
Create vault      → enter master password twice → Create encrypted vault
Open vault        → drag & drop file → enter password → Unlock
Save vault        → download icon → pick CSV or JSON
Rename vault      → rename icon → edit name (extension = format)
Change master pw  → change-password icon → set new pw → re-download
Lock (stay on tab)→ lock-vault icon → enter pw to resume
Log out           → log-out icon → confirmed → memory wiped
Auto-lock         → 5 minutes of inactivity
Copy clears in    → 30 seconds
```

---

_Umbra is a small, self-contained tool. Read the source — it's right there in the single HTML file — and audit the crypto before trusting it with your secrets._
