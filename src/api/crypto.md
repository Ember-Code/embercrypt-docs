# Crypto API

```c
#include "encvfs_crypto.h"
```

These functions are used internally by the VFS layer. You do not need to call them directly unless you are extending encvfs or writing custom migration tooling.

---

## Key derivation

### `derive_subkeys`

```c
int derive_subkeys(const uint8_t* master,   size_t master_len,
                   uint8_t*       xts_key_out, size_t xts_len,
                   uint8_t*       hmac_key_out, size_t hmac_key_len);
```

Derives separate XTS and HMAC subkeys from the master key. The two subkeys are domain-separated so that a single master key is safe to use for both operations.

Returns `1` on success, `0` on failure.

---

## AES-XTS

### `aes_xts_init`

```c
EVP_CIPHER_CTX* aes_xts_init(const unsigned char* key, int encrypt);
```

Allocates and initializes an OpenSSL EVP cipher context for AES-256-XTS.

`encrypt`: `1` to encrypt, `0` to decrypt.

The caller owns the returned context and must free it with `EVP_CIPHER_CTX_free`.

### `make_xts_tweak`

```c
void make_xts_tweak(uint64_t page_no, uint8_t tweak[16]);
```

Builds the 16-byte XTS tweak from a page number. Each page uses a unique tweak, binding ciphertext to its position in the file.

### `aes_xts_crypt`

```c
int aes_xts_crypt(EVP_CIPHER_CTX*    ctx,
                  const unsigned char* iv,
                  const unsigned char* in,
                  unsigned char*       out,
                  size_t               len);
```

Encrypts or decrypts one page in-place using the context initialized by `aes_xts_init`. `iv` is the tweak produced by `make_xts_tweak`.

Returns `1` on success, `0` on failure.

---

## HMAC

### `create_hmac_params`

```c
void create_hmac_params(const uint8_t* key, size_t key_len, OSSL_PARAM* params);
```

Populates an `OSSL_PARAM` array for use with the OpenSSL MAC EVP API (SHA-256).

### `hmac_page`

```c
int hmac_page(EVP_MAC_CTX*   ctx,
              OSSL_PARAM*    params,
              const uint8_t* page_data,
              size_t         page_len,
              uint32_t       page_no,
              uint8_t        out[32]);
```

Computes a 32-byte SHA-256 HMAC over `page_data` and `page_no`. Including the page number in the MAC binds the tag to the page's location, preventing block-swapping attacks.

Returns `1` on success, `0` on failure.

### `hmac_match`

```c
int hmac_match(uint8_t a[32], uint8_t b[32]);
```

Constant-time comparison of two 32-byte HMACs. Returns `1` if equal, `0` otherwise. Use this instead of `memcmp` to avoid timing side-channels.
