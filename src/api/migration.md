# Migration API

```c
#include "encvfs_migrate.h"
```

Utilities for converting between plaintext SQLite databases and encvfs-encrypted ones, and for rekeying. All functions operate on file paths directly and do not require an open `sqlite3*` handle.

## Schema preservation

The migration functions operate at the page level: the entire SQLite file is encrypted or decrypted as-is. This means all schema objects survive a roundtrip without any special handling:

- Tables and their data
- Explicit indexes
- Triggers
- Views

After a `vfs_encrypt_database` + `vfs_decrypt_database` roundtrip, the result is a byte-equivalent plaintext SQLite file. Triggers fire correctly, views return correct results, and indexes are intact.

---

## `vfs_encrypt_database`

```c
int vfs_encrypt_database(char* plain_path,
                          char* enc_path,
                          unsigned char key[ENC_VFS_KEY_SIZE]);
```

Reads a plaintext SQLite database at `plain_path`, encrypts it page by page, and writes the result to `enc_path`. The source file is not modified.

Returns `0` on success.

```c
unsigned char key[ENC_VFS_KEY_SIZE];
/* ... load key ... */

int rc = vfs_encrypt_database("orders.db", "orders.enc.db", key);
if (rc != 0) {
    fprintf(stderr, "encryption failed\n");
}
```

## `vfs_decrypt_database`

```c
int vfs_decrypt_database(char* enc_path,
                          char* plain_path,
                          unsigned char key[ENC_VFS_KEY_SIZE]);
```

Decrypts an encvfs database at `enc_path` and writes a standard SQLite database to `plain_path`. HMAC verification runs on every page; the operation aborts on the first integrity failure.

Returns `0` on success.

```c
int rc = vfs_decrypt_database("orders.enc.db", "orders.plain.db", key);
if (rc != 0) {
    fprintf(stderr, "decryption or integrity check failed\n");
}
```

### Verifying schema after a roundtrip

The output of `vfs_decrypt_database` is a normal SQLite file and can be opened without the VFS:

```c
sqlite3 *db;
sqlite3_open_v2("orders.plain.db", &db, SQLITE_OPEN_READWRITE, NULL);

/* Schema objects are intact. */
/* sqlite_master will contain the index, trigger, and view. */

/* Views query correctly. */
/* (only rows with total > 100 are returned by big_orders) */
sqlite3_exec(db, "SELECT count(*) FROM big_orders;", cb, &n, NULL);

/* Triggers fire on new inserts. */
sqlite3_exec(db, "INSERT INTO orders VALUES (3, 'carol', 300.0);", 0, 0, 0);
/* audit table gains one row from trg_orders_insert */
```

## `vfs_rekey_database`

```c
int vfs_rekey_database(char*         db_path,
                        unsigned char old_key[ENC_VFS_KEY_SIZE],
                        unsigned char new_key[ENC_VFS_KEY_SIZE]);
```

Re-encrypts every page of an encvfs database in-place under `new_key`. Each page is decrypted with `old_key`, HMAC-verified, then re-encrypted and re-tagged under `new_key`. Aborts on any HMAC failure.

Returns `0` on success.

> **Warning:** This modifies the file in-place. Back up the database before rekeying.

```c
int rc = vfs_rekey_database("orders.enc.db", old_key, new_key);
if (rc != 0) {
    fprintf(stderr, "rekey failed - restore from backup\n");
}
```

The same operations are available from the command line via [encshell](../tools/encshell.md).
