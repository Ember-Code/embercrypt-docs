# Performance & Testing

## Performance

Embercrypt introduces a measurable overhead due to its security-first design. XTS-AES requires that all read and write operations are performed on full 4096-byte pages; partial page reads or writes are not possible.

In practice, real-world performance can come close to unencrypted SQLite when workloads are properly optimized. The two most impactful levers are:

- **Caching:** Keeping hot pages in memory reduces the number of decrypt operations on the critical path.
- **Page size configuration:** Tuning `ENC_PAGE_SIZE` to match your access patterns can reduce padding overhead and improve throughput.

Because results vary significantly by workload and environment, we provide detailed optimization guidance and consultation. Contact us to discuss your specific use case.

## Test suite

The test suite intentionally exceeds the codebase in size by a factor of [XX]. Coverage includes:

- **Unit tests** for each crypto primitive (`derive_subkeys`, `aes_xts_crypt`, `hmac_page`, `hmac_match`).
- **VFS integration tests** covering open, read, write, and close paths against the encrypted VFS.
- **Schema roundtrip tests** verifying that indexes, triggers, and views survive an encrypt/decrypt cycle intact and behave correctly.
- **Migration tests** for `vfs_encrypt_database`, `vfs_decrypt_database`, and `vfs_rekey_database`.
- **Negative tests** for HMAC failure detection, wrong-key rejection, and tampered page handling.

### Building and running
```sh
make tests      # build test binaries against libencvfs_test.so (ASAN + UBSan enabled)
make run-tests  # execute all tests
```

The test build links against `libencvfs_test.so`, a separate shared library built without hardening flags so that ASAN and UBSan can instrument it fully.
```sh
make bench
make run-bench  # run throughput benchmarks
```

Benchmark output reports per-page encrypt/decrypt throughput and compares against a baseline unencrypted VFS read/write cycle.