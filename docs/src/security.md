# Security Considerations

## Threat model

Embercrypt is designed to protect against attackers with read or write access to files on disk. The entire database file is encrypted at rest, providing both confidentiality and integrity:

- **Confidentiality:** Data is inaccessible to unauthorized parties without the master key.
- **Integrity:** SHA-256 HMACs are computed over the full ciphertext of each page. Unauthenticated modifications are rejected outright.

## WAL and journal files

WAL and journal files are not currently encrypted within the standard offering. If your threat model requires encryption of those files, please contact us.

## In-memory protections

Embercrypt makes no formal guarantees about data in memory, though care is taken to avoid unnecessary exposure of keys and plaintext in the process address space. If you require stronger in-memory or in-transit protections, please contact us.

## Stability and reliability

The Embercrypt codebase is intentionally kept small to minimize attack surface and make independent auditing practical. No SQLite source code is modified in any form.

- All cryptographic primitives are delegated exclusively to OpenSSL, which carries an extensive real-world testing and audit history.
- The implementation leans on well-established SQLite internals wherever possible.
- The codebase is analyzed for memory safety and undefined behavior.

That commitment to a small codebase does not extend to the test suite, which exceeds the codebase itself.

## Subkey derivation

The master key is never used directly for encryption or authentication. `derive_subkeys` produces separate XTS and HMAC subkeys, ensuring domain separation so that compromise of one subkey does not trivially affect the other.

## Encryption: AES-256-XTS

XTS mode is designed for storage encryption where each sector is an independently addressable unit. The per-page tweak is derived from the page number, binding ciphertext to its position and preventing page-relocation attacks.

## Integrity: SHA-256 HMAC

Each page carries a 32-byte HMAC computed over the encrypted page data and the page number. Including the page number in the MAC prevents block-swap attacks, where an attacker reorders pages within the same or a different database. HMAC comparison uses `hmac_match`, which runs in constant time to avoid timing side-channels.

## Rekeying

`vfs_rekey_database` modifies the database in-place. Every page is HMAC-verified under the old key before being re-encrypted under the new key. Back up the database before rekeying. See [Migration API](api/migration.md).
