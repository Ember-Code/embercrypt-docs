# VFS API

```c
#include "encvfs.h"
```

---

## `sqlite3_encvfs`

```c
sqlite3_vfs* sqlite3_encvfs(const char* name);
```

Creates a new encvfs instance with the given VFS name. The name is passed to `sqlite3_open_v2` to select this VFS.

Returns `NULL` on allocation failure.

```c
sqlite3_vfs *vfs = sqlite3_encvfs("enc");
```

## `sqlite3_encvfs_free`

```c
void sqlite3_encvfs_free(sqlite3_vfs* vfs);
```

Frees the VFS and all associated resources. Call only after `encvfs_unregister_all_keys` and after all databases using this VFS are closed.

---

## `encvfs_register_key`

```c
int encvfs_register_key(sqlite3_vfs* encVfs,
                         char* dbName,
                         unsigned char key[ENC_VFS_KEY_SIZE]);
```

Associates a 64-byte master key with a database path. The path must match exactly the string passed to `sqlite3_open_v2`. Must be called before opening the database.

Returns `SQLITE_OK` on success.

```c
encvfs_register_key(vfs, "data/myapp.db", key);
```

> **Note:** The key is stored in a linked list of `EncKey` structs for the lifetime of the VFS. Clear the source buffer after registration if it is no longer needed.

## `encvfs_deregister_key`

```c
int encvfs_deregister_key(sqlite3_vfs* encVfs, char* dbName);
```

Removes and clears the key for the named database. Safe to call while other databases remain open on the same VFS.

## `encvfs_unregister_all_keys`

```c
void encvfs_unregister_all_keys(sqlite3_vfs* encVfs);
```

Removes and clears all registered keys. Call before `sqlite3_encvfs_free`.

---

## `encvfs_init_db`

```c
int encvfs_init_db(sqlite3* db, int force);
```

Configures an open database for use with encvfs. Sets the page size to `ENC_PAGE_SIZE` (4096) and performs any required header initialization.

`force = 1` re-initializes even if a header already exists. Use `0` for normal opens.

Returns `SQLITE_OK` on success.

---

## Constants

| Constant | Value | Description |
|---|---|---|
| `ENC_PAGE_SIZE` | `4096` | Database page size in bytes |
| `HMAC_SIZE` | `32` | Per-page HMAC size in bytes |
| `ENC_VFS_KEY_SIZE` | `64` | Master key size in bytes |

---

## `EncKey`

```c
typedef struct EncKey {
    const char*          dbName;
    const unsigned char  key[ENC_VFS_KEY_SIZE];
    struct EncKey       *next;
} EncKey;
```

Internal linked-list node. Not intended for direct use by callers.
