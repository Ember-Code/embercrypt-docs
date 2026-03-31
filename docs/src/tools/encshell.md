# encshell

`encshell` is a command-line tool shipped alongside encvfs. It behaves like the standard `sqlite3` shell but opens encrypted databases transparently. It also provides utilities for encrypting, decrypting, and rekeying database files.

## Usage

```
encshell [options] <encrypted-database-file> <key-file>
```

`<key-file>` is a path to a binary file containing exactly 64 bytes (`ENC_VFS_KEY_SIZE`).

## Options

| Option | Long form | Argument | Description |
|---|---|---|---|
| (none) | (none) | | Opens the database in interactive shell mode |
| `-c` | `--encrypt` | `<plaintext-db>` | Encrypt an existing SQLite database |
| `-d` | `--decrypt` | `<plaintext-db>` | Decrypt to a plaintext SQLite database |
| `-r` | `--rekey`   | `<new-key-file>` | Rekey database in-place with a new key |
| `-h` | `--help`    | | Show help message |

---

## Examples

### Interactive shell

```sh
./encshell data/myapp.db data/myapp.key
```

Drops into an interactive SQLite prompt. All SQL commands work as normal.

```
SQLite version 3.x.x
Enter ".help" for usage hints.
sqlite> SELECT count(*) FROM users;
42
sqlite> .quit
```

### Encrypt an existing database

```sh
./encshell -c plain.db encrypted.db mykey.bin
```

Reads `plain.db`, encrypts it, and writes the result to `encrypted.db`. The plaintext file is not deleted.

### Decrypt for inspection or export

```sh
./encshell -d encrypted.db plain_export.db mykey.bin
```

HMAC verification runs on every page. The command aborts if any page fails.

### Rekey

```sh
./encshell -r encrypted.db old.key new.key
```

Re-encrypts all pages in-place under the new key. Keep a backup first.

---

## Generating a key file

The key file must be exactly 64 random bytes. Using OpenSSL:

```sh
openssl rand 64 > myapp.key
chmod 600 myapp.key
```

Or with `/dev/urandom`:

```sh
dd if=/dev/urandom of=myapp.key bs=64 count=1
```

> **Warning:** Losing the key file means the database is permanently unrecoverable. Store it separately from the database and back it up securely.
