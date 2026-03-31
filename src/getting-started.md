# Getting Started

## Installation

### Prebuilt binary

You receive `libencvfs.so` and the three headers (`encvfs.h`, `encvfs_crypto.h`, `encvfs_migrate.h`).
Place the `.so` somewhere your linker can find it, then link:

```sh
gcc main.c -lencvfs -lsqlite3 -lcrypto -L/path/to/lib -I/path/to/include -o myapp
```

If the library lives next to your binary at runtime and was built with `-rpath '$ORIGIN'`:

```sh
gcc main.c -lencvfs -lsqlite3 -lcrypto -L. -I./include -Wl,-rpath,'$ORIGIN' -o myapp
```

### Building from source

```sh
make          # builds libencvfs.so
make DEBUG=1  # adds -g and -DDEBUG
```

The resulting `.so` is position-independent and hardened:

| Flag | Purpose |
|---|---|
| `-fstack-protector-strong` | Stack canaries |
| `-fstack-clash-protection` | Stack clash mitigation |
| `-D_FORTIFY_SOURCE=2` | Glibc buffer checks |
| `-fcf-protection=full` | Intel CET (IBT + SHSTK) |
| `-z relro -z now` | Full RELRO |
| `-z noexecstack` | Non-executable stack segment |

To build and run the test suite:

```sh
make tests
make run-tests
```

---

## Minimal setup

Opening an encrypted database requires three steps before the standard SQLite calls:

1. Create the VFS instance.
2. Register the key for the target database path.
3. Register the VFS with SQLite, then open normally.

```c
#include "encvfs.h"

unsigned char key[ENC_VFS_KEY_SIZE]; /* 64 bytes */
/* ... fill key from a secure source, never hardcode ... */

sqlite3_vfs *vfs = sqlite3_encvfs("enc");
encvfs_register_key(vfs, "data/myapp.db", key);
sqlite3_vfs_register(vfs, 0);  /* 0 = not the default VFS */

sqlite3 *db;
sqlite3_open_v2("data/myapp.db", &db,
    SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE, "enc");

encvfs_init_db(db, 0);  /* 0 = do not force re-init */
```

## Teardown

Keys should be cleared from memory when no longer needed.

```c
sqlite3_close(db);
encvfs_unregister_all_keys(vfs);
sqlite3_encvfs_free(vfs);
```

To remove a single key without destroying the VFS:

```c
encvfs_deregister_key(vfs, "data/myapp.db");
```

## Encrypting an existing plaintext database

Use the migration API or the shell tool if you already have a SQLite database.

```c
#include "encvfs_migrate.h"

vfs_encrypt_database("plain.db", "encrypted.db", key);
```

See [Migration API](api/migration.md) and the [Shell Tool](examples/shell.md) for details.
