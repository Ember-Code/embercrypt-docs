# Migration Examples

## Encrypting an existing deployment

If you have an existing SQLite database and want to migrate it to encvfs without downtime, encrypt to a new file and swap atomically:
```c
int migrate_to_encrypted(const char *db_path, const char *key_path) {
    unsigned char key[ENC_VFS_KEY_SIZE];
    if (load_key(key_path, key) != 0)
        return -1;

    /* Write to a temp file first, never encrypt over the live database. */
    char tmp_path[512];
    snprintf(tmp_path, sizeof(tmp_path), "%s.enc.tmp", db_path);

    int rc = vfs_encrypt_database(db_path, tmp_path, key);
    if (rc != 0) {
        fprintf(stderr, "encryption failed\n");
        return -1;
    }

    /* Atomic swap on POSIX systems. */
    if (rename(tmp_path, db_path) != 0) {
        perror("rename");
        remove(tmp_path);
        return -1;
    }

    return 0;
}
```

The plaintext file is not modified. Only rename once encryption has succeeded.

## Exporting for inspection or support

Decrypt to a temporary plaintext file, inspect it with standard SQLite tooling, then delete it:
```c
unsigned char key[ENC_VFS_KEY_SIZE];
load_key("myapp.key", key);

vfs_decrypt_database("myapp.db", "/tmp/myapp_export.db", key);

/* Use sqlite3 CLI, DB Browser, or any other tool on the export. */
/* Delete securely when done - do not leave plaintext on disk. */
remove("/tmp/myapp_export.db");
```

> **Warning:** The decrypted file is unprotected. Store it on a `tmpfs` or encrypted volume, and delete it as soon as you are done.

## Key rotation

Rotate the database key periodically or in response to a key compromise. Always back up first:
```c
int rotate_key(const char *db_path,
               const char *old_key_path,
               const char *new_key_path) {
    unsigned char old_key[ENC_VFS_KEY_SIZE];
    unsigned char new_key[ENC_VFS_KEY_SIZE];

    if (load_key(old_key_path, old_key) != 0) return -1;
    if (load_key(new_key_path, new_key) != 0) return -1;

    /* Back up before modifying in-place. */
    copy_file(db_path, "myapp.db.bak");

    int rc = vfs_rekey_database(db_path, old_key, new_key);
    if (rc != 0) {
        fprintf(stderr, "rekey failed - restore from myapp.db.bak\n");
        return -1;
    }

    remove("myapp.db.bak");
    return 0;
}
```

`vfs_rekey_database` verifies the HMAC of every page under the old key before re-encrypting. If any page fails, the operation aborts. The backup copy allows recovery if the process is interrupted.

## Backup strategy

For regular backups, encrypt a separate copy rather than decrypting the live database:
```c
/* Decrypt live -> plaintext -> re-encrypt under backup key. */
unsigned char live_key[ENC_VFS_KEY_SIZE];
unsigned char backup_key[ENC_VFS_KEY_SIZE];

load_key("live.key",   live_key);
load_key("backup.key", backup_key);

vfs_decrypt_database("myapp.db",        "/tmp/myapp.plain.db", live_key);
vfs_encrypt_database("/tmp/myapp.plain.db", "myapp.backup.db", backup_key);
remove("/tmp/myapp.plain.db");
```

The backup copy is protected by a separate key, so a live key compromise does not automatically expose historical backups.
