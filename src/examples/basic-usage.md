# Basic Usage

A complete example: open an encrypted database, create a table, insert rows, and read them back.

```c
#include <stdio.h>
#include <sqlite3.h>
#include "encvfs.h"

int main(void) {
    sqlite3     *db   = NULL;
    sqlite3_stmt *stmt = NULL;
    int rc;

    /* --- Key setup -------------------------------------------------- */
    /* In production, load this from a keyfile, hardware token, or KDF. */
    unsigned char key[ENC_VFS_KEY_SIZE];
    /* ... fill key ... */

    /* --- VFS setup -------------------------------------------------- */
    sqlite3_vfs *vfs = sqlite3_encvfs("enc");
    encvfs_register_key(vfs, "data/myapp.db", key);
    sqlite3_vfs_register(vfs, 0);

    /* --- Open ------------------------------------------------------- */
    rc = sqlite3_open_v2("data/myapp.db", &db,
             SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE, "enc");
    if (rc != SQLITE_OK) {
        fprintf(stderr, "open: %s\n", sqlite3_errmsg(db));
        return 1;
    }

    rc = encvfs_init_db(db, 0);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "init_db failed\n");
        sqlite3_close(db);
        return 1;
    }

    /* --- Write ------------------------------------------------------ */
    rc = sqlite3_exec(db,
             "CREATE TABLE IF NOT EXISTS t(x TEXT);"
             "DELETE FROM t;"
             "INSERT INTO t VALUES('lorem'),('ipsum'),('dolor');",
             0, 0, 0);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "exec: %s\n", sqlite3_errmsg(db));
        return 1;
    }

    /* --- Read ------------------------------------------------------- */
    rc = sqlite3_prepare_v2(db, "SELECT x FROM t ORDER BY x;", -1, &stmt, 0);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "prepare: %s\n", sqlite3_errmsg(db));
        return 1;
    }

    while ((rc = sqlite3_step(stmt)) == SQLITE_ROW) {
        printf("%s\n", sqlite3_column_text(stmt, 0));
    }
    if (rc != SQLITE_DONE) {
        fprintf(stderr, "step failed\n");
    }

    /* --- Teardown --------------------------------------------------- */
    sqlite3_finalize(stmt);
    sqlite3_close(db);
    encvfs_unregister_all_keys(vfs);
    sqlite3_encvfs_free(vfs);

    return 0;
}
```
