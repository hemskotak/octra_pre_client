# Copilot instructions — octra_pre_client

Quick, focused guidance to help an AI coding agent get productive quickly.

## Big picture
- This repository is a small terminal wallet client (`cli.py`) for the Octra network. It's a TUI (ANSI terminal) that talks to an Octra RPC server over HTTP(S).
- Main runtime model: asynchronous CLI using `asyncio` + `aiohttp` for network calls, with a small threadpool to handle blocking `input()` calls.

## Key files & flow
- `cli.py` — single-file app containing the entire client logic (UI, network, crypto). Primary functions of interest:
  - `ld()` — loads `wallet.json` (prefers `~/.octra/wallet.json`) and derives keys (base64 `priv` -> NaCl signing key, public key `pub`).
  - `req()` / `req_private()` — wrappers for `aiohttp` calls to the RPC server; default timeout 10s. Use these when adding network calls.
  - `st()` / `gh()` — fetch balance/nonce and recent transactions; `mk()` and `snd()` create and submit signed transactions.
  - UI entry points: `scr()` (main screen), `tx()`, `multi()`, `encrypt_balance_ui()`, `decrypt_balance_ui()`, `private_transfer_ui()`, `claim_transfers_ui()`.

## RPC endpoints and patterns (observed in `cli.py`)
- GET /balance/{addr}
- GET /staging
- GET /address/{addr}
- GET /public_key/{addr}
- GET /tx/{hash}
- POST /send-tx
- POST /encrypt_balance
- POST /decrypt_balance
- POST /private_transfer
- POST /claim_private_transfer
- GET /view_encrypted_balance/{addr}

When adding features or debugging the backend integration, these endpoints are referenced in `req()` and `req_private()` calls.

## Crypto & formats to be aware of
- Wallet format: `wallet.json` with keys: `{ "priv": "<base64-priv>", "addr": "oct...", "rpc": "https://..." }` (see `wallet.json.example`).
- Address validation: regex in `cli.py`: `^oct[1-9A-HJ-NP-Za-km-z]{44}$`. Use this to validate addresses in tests or UI changes.
- Private key is base64-ed NaCl signing key. `ld()` creates `sk = nacl.signing.SigningKey(base64.b64decode(priv))` and `pub = base64.b64encode(sk.verify_key.encode()).decode()`.
- Backwards-compatible: as of now `ld()` accepts either a 32-byte hex seed (64 hex characters) or a base64 NaCl signing key; hex seeds are converted to base64 automatically. Example hex: `4bc29ffdeb9d376ddb2cc3fd1f44dc4d80acbb43df82ef51e810b4eef1deff6d`.
- Encrypted balance format:
  - v2: `"v2|<base64(nonce + ciphertext)>"` — decrypted with AES-GCM using key derived in `derive_encryption_key(priv)`; salt: `b"octra_encrypted_balance_v2"`.
  - legacy v1: custom HMAC/xor scheme (implemented in `decrypt_client_balance`), with salt `b"octra_encrypted_balance_v1"`. Keep backwards compatibility when changing balance logic.
- Private transfers: ephemeral public key + encrypted amount. Shared secret derivation: `derive_shared_secret_for_claim(my_priv, ephemeral_pub)` — sorts the two public keys, sha256 twice and truncates to 32 bytes. Use same routine if implementing claim/preview features.

## Developer workflows & quick commands
- Setup & run locally:
  - python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
  - Copy `wallet.json.example` -> `wallet.json` and fill `priv`, `addr`, `rpc` (default `rpc` may be `http://localhost:8080` for local dev)
  - Run: `./run.sh` (Linux/mac) or `run.bat` (Windows)
- For quick dev against a local node: set `rpc` to `http://localhost:8080` (the client will warn if RPC is HTTP and not localhost).
- Sessions: `aiohttp.ClientSession` is created lazily in `req()` with a default 10s timeout and global `session`. Close/cleanup happens on exit. When adding long-lived requests or debug logging, consider adjusting timeout or making the session injection-friendly.

## Project-specific conventions & notes
- Single-file app: most logic is in `cli.py` — search there first for relevant functionality.
- UI is position-based (ANSI cursor movement). Be careful when updating UI code — small changes can affect layout across terminals.
- Sensitive data handling:
  - `wallet.json` is read from `~/.octra/wallet.json` if present.
  - When exporting wallet, the code sets file permissions (`os.umask(0o077)`, chmod 0o600). Preserve this when adding export/save features.
  - Avoid printing `priv` in logs or UI except in explicit export flows.
- Address & key formats are validated/used directly — use existing helper functions/patterns (`b58` regex, base64 key usage) for consistency.

## Debugging tips
- If network errors occur, inspect the `req()` return tuple `(status, text, json)` — the code often treats non-JSON as an error string.
- Typical failure modes: timeouts, invalid JSON from the RPC server, missing `has_public_key` flag when attempting private transfers.
- To reproduce claim/transfer flows, use the `private_transfer_ui()` and `claim_transfers_ui()` logic to exercise ephemeral key creation and shared-secret decryption.

## Avoid
- Do not add anything that logs or includes `priv` by default. Keep wallet secrets out of commit history and PRs.

---
If you'd like, I can (1) merge an existing `.github/copilot-instructions.md` if present, (2) expand any of the sections with code examples from `cli.py`, or (3) add a short quick-start snippet as a separate `docs/` file. Which would you prefer?