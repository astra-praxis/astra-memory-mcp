# Security & Privacy Design (Astra Memory MCP)

Astra Memory protects long-term memory with public, well-analyzed primitives
only. There is no security by obscurity: everything below is standard
cryptography, and the exact primitive set is visible in the binary.

## What is stored, where

```
~/.astra-memory/            (or ASTRA_MEMORY_DIR)
├── memory.key              32-byte master key, file mode 0600 (unix)
└── memory-cme.sqlite       single encrypted SQLite database
```

- Nothing leaves the machine: the server is a stdio process started by your
  agent harness; it opens no sockets and performs no telemetry.
- The `ASTRA_MEMORY_KEY_HEX` environment variable (when you choose to inject a
  key) is read once at startup and never logged.

## Key hierarchy

```
Master Key (32 random bytes, per store)
   │
   ├─ HKDF-SHA256 (info: "astra-memory/dek/xchacha20poly1305")
   │        → payload DEK
   └─ HKDF-SHA256 (separate info domain)
            → content-hash key (keyed HMAC)

Payload AEAD   : XChaCha20-Poly1305 (chacha20poly1305 crate, RustCrypto)
Per-item nonce : 192-bit random, stored alongside ciphertext
Content hashes : keyed HMAC over normalized text — non-reversible,
                 used for duplicate detection, not search
```

- Domain separation via HKDF `info` labels: a leaked subkey from one role can
  never be replayed in the other role.
- XChaCha20-Poly1305 gives a 192-bit nonce space, making random-nonce reuse
  practically impossible without any counter state.
- All rows are encrypted **at the payload level**; the SQLite file on disk
  contains no readable memory content — only ids, scope metadata and ciphertext
  columns.

## Failure behavior

- Wrong key / corrupted rows: decryption fails **loudly**. The server refuses
  to start rather than degrading to plaintext (no silent-fallback path).
- A key file shorter than 32 bytes is rejected with an explicit error instead
  of being silently rotated (rotation would make existing data unreadable).

## Threat model & honest boundaries

| Covered | Not covered (by design today) |
|---|---|
| Other local users reading the DB file (unix file mode + payload encryption) | A root/admin attacker with your key file |
| Accidental sync/backup upload of the database (ciphertext only) | Memory-page attacks on a compromised machine |
| Curious MCP clients of *other* servers — tools are the only surface | The harness itself, which the user grants access deliberately |

- The master key lives next to the database for local convenience. This is the
  same trade-off as desktop keychain-less apps; hardware/OS-keychain binding
  is an explicit future direction.
- Anyone with both `memory.key` and `memory-cme.sqlite` can read your
  memories. Treat the key file like a private key.

## Memory is portable — never hostage

The tools exposed over MCP (`astra_remember` / `astra_recall` /
`astra_forget` / `astra_get_memory`) define the complete agent-visible surface,
and every response is plain JSON. Combined with the documented database layout
(scope columns, ciphertext columns, key file format), a user can always export
or migrate their memory. A public export/import CLI is on the roadmap; the
format will be published here first so no release can retroactively lock you
in.
