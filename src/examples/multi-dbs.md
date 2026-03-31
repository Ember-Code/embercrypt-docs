# Multiple databases on one VFS

A single VFS instance can handle multiple databases, each with its own key.

```c
sqlite3_vfs *vfs = sqlite3_encvfs("enc");

encvfs_register_key(vfs, "data/users.db",  key_users);
encvfs_register_key(vfs, "data/events.db", key_events);
sqlite3_vfs_register(vfs, 0);

sqlite3 *users_db, *events_db;
sqlite3_open_v2("data/users.db",  &users_db,  SQLITE_OPEN_READWRITE, "enc");
sqlite3_open_v2("data/events.db", &events_db, SQLITE_OPEN_READWRITE, "enc");

encvfs_init_db(users_db,  0);
encvfs_init_db(events_db, 0);
```

Keys are matched by database path, so each file is independently encrypted.
