# embercrypt - encvfs

A transparent encryption layer for SQLite implemented as a custom VFS (Virtual File System).

Every database page is encrypted with **AES-256-XTS** and authenticated with a **SHA-256 HMAC** before being written to disk. The host application supplies a 64-byte master key; subkeys for XTS and HMAC are derived internally via `derive_subkeys`.

## Design at a glance

| Layer | Detail |
|---|---|
| Encryption | AES-256-XTS, one tweak per page number |
| Integrity | SHA-256 HMAC appended per page |
| Page size | 4096 bytes |
| Key input | 64 bytes (512-bit master key) |
| Key scope | Per-database, registered before `sqlite3_open_v2` |

The VFS is a shim that wraps the default OS VFS. SQLite itself has no knowledge of encryption.

## Dependencies

- SQLite 3
- OpenSSL 3 (EVP API)
