# Umbra — Zero-Knowledge Password Manager

A file-based, zero-knowledge password manager that runs entirely in your browser. Every field is sealed with **AES-256-GCM** using a key derived from your master password via **PBKDF2-SHA256 (310,000 iterations)**. Nothing is uploaded, nothing is synced, and no server ever sees your data — the entire vault lives in a single CSV or JSON file that you own.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Getting Started](#getting-started)
4. [Security Model](#security-model)
5. [How Unlock Works](#how-unlock-works)
6. [Vault File Formats](#vault-file-formats)
   - [CSV Format](#csv-format)
   - [JSON Format](#json-format)
   - [The `format` field — a per-vault derived marker](#the-format-field--a-per-vault-derived-marker)
7. [Architecture](#architecture)
8. [Key Functions](#key-functions)
9. [UI Reference](#ui-reference)
10. [Storage](#storage)
11. [Browser Requirements](#browser-requirements)
12. [Limitations](#limitations)
13. [Cheatsheet](#cheatsheet)
14. [Glossary](#glossary)

---

## Overview

Umbra stores your whole vault in a **single CSV or JSON file** of base64 ciphertext. There is no database, no backend, no telemetry, no analytics — the app is a single HTML document that you can open offline.

Each entry is encrypted independently, so the file leaks nothing about its contents beyond the number of entries and their last-updated timestamps. The vault file itself never carries a product marker, product name, or vendor identifier — an outside observer sees only opaque base64 blobs, an opaque per-vault marker, and generic column headers.

> **Note:** the vault file contains **no product name and no fixed marker**. The only "format" value in the JSON envelope is a _derived_ marker (`SHA-256(salt)` truncated to 96 bits), which is unique to each vault.

---

## Features

- **Zero-knowledge by design** — your master password is never stored, never hashed, never sent anywhere.
- **AES-256-GCM** authenticated encryption for every entry payload.
- **PBKDF2-SHA256** with **310,000 iterations** for key derivation (16-byte random salt per vault).
- **Single-file vault** in either **CSV** or **JSON** — back it up, put it on a USB stick, sync it yourself.
- **Offline & tracker-free** — no network calls, no cookies, no third-party scripts.
- **Two password generators**: Random Password and Memorable Passphrase.
- **Username generator**: random word, plus-addressed email, or catch-all email.
- **Custom fields** per entry: `text`, `hidden`, `checkbox`, `linked`.
- **Multi-select** with a floating action bar for bulk deletion.
- **Auto-lock** after 5 minutes of inactivity.
- **Clipboard auto-clear** 30 seconds after copying a secret.
- **Session persistence** — refreshing the page returns you to the unlock screen without losing the loaded vault file.
- **Rename vault file**, **change master password**, **save as CSV or JSON**.
- **Light / dark theme** with `prefers-color-scheme` detection.

---

## Getting Started

### Create a new vault

1. Open `index.html` in a modern browser (over **HTTPS** or `localhost`).
2. In the **Create a new vault** card, enter a master password (min. 8 characters) and confirm it.
3. Optionally click the refresh icon for a generated password, or the key icon to open the full generator.
4. Click **Create encrypted vault**.
5. Add entries with **Add**.
6. Click the **download icon** in the toolbar and pick **CSV** or **JSON** to save your encrypted vault file.

> Remember to download the vault after creating it — until you do, the only copy lives in your browser's memory.

### Open an existing vault

1. Drag your `.csv` / `.json` vault onto the dropzone, or click to browse.
2. Enter your master password on the unlock screen.
3. Umbra decrypts the vault in memory only.

---

## Security Model

### Zero-Knowledge table

| Component               | Value / Mechanism                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Cipher                  | AES-256-GCM                                                                                                        |
| Key derivation          | PBKDF2-SHA256                                                                                                      |
| Iterations              | `310000` (constant `DEFAULT_ITERATIONS`)                                                                           |
| Salt                    | 16 random bytes, base64-encoded, stored in the vault file                                                          |
| IV                      | 12 random bytes per encryption, base64-encoded                                                                     |
| Verifier payload        | `{ v: 1, check: "9f4c2a1eb7d3" }` — encrypted with the derived key; decrypts successfully only on correct password |
| Encrypted entry payload | `{ title, username, password, url, notes, customFields[] }`                                                        |
| **File format marker**  | `SHA-256("f1:" + saltB64)` truncated to the first **96 bits → 24 hex chars**                                       |
| Master password storage | **Never stored**, not even hashed                                                                                  |
| Network calls           | **None**                                                                                                           |

### What an attacker sees in the vault file

- The number of entries.
- The `updated` timestamp of each entry (ISO 8601).
- The random salt and iteration count.
- A per-vault marker (96-bit hex) in the JSON envelope.
- Opaque base64 blobs: IVs and ciphertexts.

Nothing else.

---

## How Unlock Works

1. You load a vault file. Umbra auto-detects the format (from the `.csv` / `.json` extension, or by sniffing the first non-whitespace character).
2. You enter your master password.
3. Umbra derives the AES key with **PBKDF2-SHA256** using the salt and iteration count read from the file.
4. It decrypts the verifier blob and checks that the plaintext equals `{ v: 1, check: "9f4c2a1eb7d3" }`.
   - If decryption throws (GCM authentication failure) or the check value mismatches, the password is wrong.
5. Only then does Umbra decrypt each entry row.

### Pre-flight check for JSON

Before touching the crypto, the JSON parser verifies that `format` equals `deriveFileMarker(salt)`. If the marker doesn't match the salt, the file is rejected as an unrecognized layout. This is a **structural sanity check**, not a security boundary — it simply prevents Umbra from trying to open a generic JSON file.

---

## Vault File Formats

### CSV Format

The CSV header is **fully generic** — nothing in it reveals the product, the vendor, or the app version.

```
type,id,updated,iv,data,salt,iterations
```

Example (with truncated base64 for readability):

```csv
type,id,updated,iv,data,salt,iterations
meta,,,,AAAAAAAAAAAAAAAA,BBBBBBBBBBBBBBBBBBBBBBBBBBBBBB,CCCCCCCCCCCCCCCCCCCCCC=,310000
entry,5e3f…-…-…,2025-01-15T12:34:56.789Z,DDDDDDDDDDDDDDDD,EEEEEEEEEE…,,
entry,9a1c…-…-…,2025-01-15T12:35:01.012Z,FFFFFFFFFFFFFFFF,GGGGGGGGGG…,,
```

| Column       | Row `meta`                        | Row `entry`                    |
| ------------ | --------------------------------- | ------------------------------ |
| `type`       | `meta`                            | `entry`                        |
| `id`         | _(empty)_                         | UUID v4                        |
| `updated`    | _(empty)_                         | ISO 8601 timestamp             |
| `iv`         | base64 IV of the verifier         | base64 IV of the entry         |
| `data`       | base64 ciphertext of the verifier | base64 ciphertext of the entry |
| `salt`       | base64 salt (16 bytes)            | _(empty)_                      |
| `iterations` | PBKDF2 iterations (e.g. `310000`) | _(empty)_                      |

- Values are escaped per RFC 4180 (`"` doubled, fields containing `,`, `"`, `\r`, or `\n` quoted).
- Line endings are `\r\n`.
- UTF-8, no BOM written by Umbra.
- The file is only invalidated by a missing `meta` row — every `entry` row that fails to decrypt is silently skipped and reported in the toast (e.g. _"Vault unlocked — 2 unreadable row(s) skipped"_).

### JSON Format

```json
{
  "format": "<24-hex-char derived marker>",
  "version": 1,
  "meta": {
    "salt": "<base64 salt>",
    "iterations": 310000,
    "iv": "<base64 IV of verifier>",
    "data": "<base64 ciphertext of verifier>"
  },
  "entries": [
    {
      "id": "<uuid v4>",
      "updated": "<ISO 8601 timestamp>",
      "iv": "<base64 IV>",
      "data": "<base64 ciphertext>"
    }
  ]
}
```

### The `format` field — a per-vault derived marker

The `format` value is **not a fixed string**. It is derived from the vault's own salt:

```
format = SHA-256("f1:" + base64(salt))[0..12]   // 96 bits, hex-encoded → 24 chars
```

Properties:

- **Stable** — the same salt always produces the same marker.
- **Unique** — different vaults (different salts) produce unrelated markers.
- **Opaque** — the output reveals nothing about Umbra, the user, or the vault's contents.
- **Verifiable offline** — anyone who has the salt can recompute it.

Example output:

```
f3a91c0b7e42d5a88f1c6b3e
```

> **No legacy marker.** There is no static, well-known string used as a fallback. If `format` doesn't equal `SHA-256("f1:" + salt)[0..12]`, the file is rejected.

---

## Architecture

```
                ┌──────────────────────────────────────────────┐
                │                 Browser tab                  │
                │                                              │
  user input ─▶ │  UI  ──▶  deriveKey(PBKDF2-SHA256, 310k)     │
                │           │                                  │
                │           ├──▶  encryptData / decryptData    │
                │           │      (AES-256-GCM, 12-byte IV)   │
                │           │                                  │
                │           ├──▶  serializeVault(format)        │
                │           │      ├── buildCSV()              │
                │           │      └── buildJSON()             │
                │           │                                  │
                │           └──▶  deriveFileMarker(salt)        │
                │                  SHA-256("f1:"+salt) → hex    │
                │                                              │
                │  sessionStorage  ◀── persistSession()        │
                │  (encrypted vault snapshot, tab-scoped)      │
                └──────────────────────────────────────────────┘
```

No network. No backend. No analytics.

---

## Key Functions

| Function                                  | Purpose                                                                                       |
| ----------------------------------------- | --------------------------------------------------------------------------------------------- |
| `randomBytes(n)`                          | Cryptographically secure random bytes via `crypto.getRandomValues`.                           |
| `bufToB64(buf)` / `b64ToBuf(b64)`         | Base64 encode/decode of `ArrayBuffer` / `Uint8Array`, chunked for large payloads.             |
| `deriveKey(password, salt, iterations)`   | PBKDF2-SHA256 → AES-256-GCM key (non-extractable, `encrypt`/`decrypt` only).                  |
| `encryptData(key, obj)`                   | Encrypts `JSON.stringify(obj)` with a fresh 12-byte IV. Returns `{ iv, ct }` as base64.       |
| `decryptData(key, ivB64, ctB64)`          | Decrypts and `JSON.parse`s; throws on GCM auth failure.                                       |
| `deriveFileMarker(saltB64)`               | `SHA-256("f1:" + salt)` → first 12 bytes → 24 hex chars. Per-vault format marker.             |
| `csvEscape(value)`                        | RFC 4180 escaping for CSV output.                                                             |
| `parseCSV(text)`                          | Quote-aware CSV parser producing an array of rows.                                            |
| `buildCSV()`                              | Serializes the current vault as CSV (meta row + one row per entry).                           |
| `buildJSON()`                             | Serializes the current vault as JSON (with `format`, `version`, `meta`, `entries`).           |
| `serializeVault(format)`                  | Dispatches to `buildCSV` or `buildJSON`.                                                      |
| `downloadText(text, filename, mime, ext)` | Saves via File System Access API when available; falls back to anchor download.               |
| `passwordStrength(pw)`                    | 0–4 score used by the strength meter.                                                         |
| `generatePassword(opts)`                  | Random password generator (length, charsets, min digits/symbols, avoid-ambiguous).            |
| `generatePassphrase(opts)`                | Word-based passphrase generator.                                                              |
| `generateUsername()`                      | Random word / plus-addressed email / catch-all email.                                         |
| `createVault()`                           | Derives a new key + salt and opens the vault screen with an empty vault.                      |
| `prepareVaultFromText(text, name, dirty)` | Parses a vault file (CSV or JSON) and moves to the unlock screen.                             |
| `unlockVault()`                           | Verifies the master password against the verifier blob, then decrypts all entries.            |
| `saveVault()` / `performSave(format)`     | Opens the format picker and downloads the encrypted vault.                                    |
| `lockVault(silent)`                       | Clears all in-memory state and returns to the welcome screen (also clears `sessionStorage`).  |
| `lockVaultKeepFile()`                     | Locks but keeps the loaded vault file (goes straight to the unlock screen for the same file). |
| `changeMasterPassword()`                  | Generates a new salt + key and marks the vault dirty for re-encryption on the next download.  |

---

## UI Reference

| Element                          | Purpose                                                                                   |
| -------------------------------- | ----------------------------------------------------------------------------------------- |
| **Theme toggle**                 | Top-right (welcome/unlock) or toolbar (vault). Persists in `localStorage`.                |
| **Dropzone**                     | Drag & drop or click to load a vault file.                                                |
| **Master password field**        | Eye toggle, quick-generate, full generator, copy.                                         |
| **Strength bar**                 | Live feedback on password quality.                                                        |
| **Search box**                   | Filters entries by title, username, URL, notes, and custom fields.                        |
| **Select-all checkbox**          | Selects all _visible_ (filtered) entries.                                                 |
| **Entry card**                   | Copy username, copy password, view, edit, delete; individual select checkbox on the left. |
| **Entry detail modal**           | Reveals password on demand; per-field copy buttons; handles hidden custom fields.         |
| **Selection bar**                | Appears when ≥1 entry is selected; offers Clear and Delete.                               |
| **Add / Edit entry modal**       | Title, username, password, URL, notes, plus custom fields.                                |
| **Add field modal**              | Choose one of four field types before adding to the entry.                                |
| **Password generator modal**     | Two tabs: Random Password / Memorable Passphrase.                                         |
| **Username generator modal**     | Three types: Random Word, Plus Addressed Email, Catch-All Email.                          |
| **Rename vault modal**           | Changes the file name; extension selects CSV or JSON for the next download.               |
| **Change master password modal** | New salt + key. Re-encrypts everything on the next download.                              |
| **Save format modal**            | Pick CSV or JSON.                                                                         |
| **Confirm modal**                | Replaces `window.confirm` with a themed dialog.                                           |
| **Toast**                        | Bottom-center notifications for copy, save, delete, errors, etc.                          |

---

## Storage

| Location             | What lives there                                                                                       |
| -------------------- | ------------------------------------------------------------------------------------------------------ |
| **In memory**        | Decrypted entries, the derived AES key, the current salt and iteration count. Cleared on lock/logout.  |
| **`sessionStorage`** | An encrypted snapshot of the vault plus its file name and dirty flag — used to survive a page refresh. |
| **`localStorage`**   | Only the theme preference under the key `umbra-theme`.                                                 |
| **Your disk**        | The encrypted `.csv` / `.json` vault file you download.                                                |

`sessionStorage` is scoped to the tab and wiped when the tab closes. `localStorage` never holds anything sensitive.

---

## Browser Requirements

- **Web Crypto API** (`crypto.subtle`) — mandatory. Umbra will refuse to run if it's missing.
- **Secure context** — `https://`, `localhost`, or `file://` in some browsers. Opening `index.html` over `http://` on a remote host will disable `crypto.subtle`.
- **Modern browser** — Chrome, Edge, Firefox, or Safari from the last few years.
- Optional: **File System Access API** (`window.showSaveFilePicker`) for a better save experience. Umbra falls back to a standard anchor download when it's unavailable.

---

## Limitations

1. **No password recovery.** If you lose your master password, the vault is unrecoverable. There is no backdoor, no reset, no hint.
2. **No sync, no sharing.** Umbra produces a file. How you move that file around is your responsibility.
3. **PBKDF2 is CPU-bound.** 310,000 iterations on a slow device can take a noticeable moment. This is intentional — it slows down brute-force attempts.
4. **Single vault at a time.** Opening a second vault replaces the first in memory.
5. **No password history.** Editing an entry overwrites the previous value.
6. **Clipboard clearing is best-effort.** The 30-second timer empties the clipboard, but the OS may have already cached the value.
7. **Auto-lock is a UI convenience.** It does not defend against a compromised browser or a malicious extension.
8. **Encrypted entries leak metadata.** The number of entries and their last-updated timestamps are visible in the file.
9. **No backward compatibility.** Vault files written by earlier or non-conforming versions that don't match the current `SHA-256("f1:" + salt)` marker are rejected. There is no legacy marker and no migration path.

---

## Cheatsheet

### Common tasks

| Task                   | How                                                                               |
| ---------------------- | --------------------------------------------------------------------------------- |
| Create a vault         | Welcome screen → enter master password twice → **Create encrypted vault**.        |
| Open a vault           | Drag the `.csv` / `.json` file onto the dropzone, then enter the master password. |
| Add an entry           | **Add** in the toolbar → fill the form → **Save entry**.                          |
| Generate a password    | Key icon next to the password field, or the toolbar generator.                    |
| Generate a passphrase  | Open the generator → **Memorable Passphrase** tab.                                |
| Generate a username    | Person icon in the toolbar or next to the username field.                         |
| Copy without revealing | Copy icon next to the password in the entry card.                                 |
| Bulk delete            | Tick the checkbox on each entry → **Delete** in the floating bar.                 |
| Rename the vault file  | Document icon in the toolbar.                                                     |
| Change master password | Key icon in the toolbar → enter new password twice → **Change password**.         |
| Download the vault     | Download icon in the toolbar → pick CSV or JSON.                                  |
| Lock but keep the file | Closed-padlock icon in the toolbar → re-enter the master password.                |
| Log out completely     | Arrow-right icon in the toolbar → confirm.                                        |

### Keyboard

| Key      | Action                          |
| -------- | ------------------------------- |
| `Enter`  | Submit the active form / unlock |
| `Escape` | Close the top-most modal        |

---

## Glossary

| Term                | Meaning                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Master password** | The only secret you memorise. Never stored. All keys are derived from it.                                        |
| **Salt**            | 16 random bytes stored in the vault. Ensures two vaults with the same password produce different keys.           |
| **IV**              | 12-byte nonce, randomly generated per encryption. Prevents ciphertext reuse.                                     |
| **Verifier**        | A small encrypted blob containing `{ v: 1, check: "9f4c2a1eb7d3" }`. Decrypts only with the correct key.         |
| **Derived marker**  | `SHA-256("f1:" + salt)` truncated to 96 bits, hex-encoded. Written to the JSON `format` field. Unique per vault. |
| **Entry**           | A single record: title, username, password, URL, notes, and any number of custom fields.                         |
| **Custom field**    | An extra per-entry field of type `text`, `hidden`, `checkbox`, or `linked`.                                      |
| **Dirty**           | Vault has in-memory changes not yet written to disk. The status pill turns amber.                                |
| **Lock**            | Clears the derived key and decrypted entries from memory. The file is untouched.                                 |
| **Auto-lock**       | Automatic lock after 5 minutes of inactivity (`AUTOLOCK_MS`).                                                    |
| **Zero-knowledge**  | No party — not even the app itself — can decrypt your vault without the master password.                         |

---

_Umbra is a single HTML file. Save it, open it, and your vault is wherever you put it._
