# Cryptex

Safe crypto for Mettle, built on **libsodium**.

Cryptex writes no crypto of its own. Every primitive is a libsodium call. Cryptex adds fixed-size types, argument checks, safe failure, and wiping of secrets.

- **API reference:** [docs/API.md](docs/API.md)
- **Known limits:** [reports/](reports/README.md)

## Requirements

- libsodium 1.0.18 or newer (runtime only). On Debian: `libsodium23`.
- Linux x86-64.
- The Mettle compiler and a current mbuild (with `mb_link_lib_any` and `mbuild test`).

## Install

In your `mbuild.mettle`:

```mettle
fn deps() {
  mb_dep_path("crypto", "../cryptex");
}
```

Then `import "crypto/crypto";`. No link flags needed: Cryptex links libsodium for you.

## Example

```mettle
import "crypto/crypto";

fn main() -> int32 {
  if (cr_init() != CR_OK) { return 1; }

  // Encrypt a value. The nonce is inside the output.
  var k: CrKey32;
  cr_aead_keygen(&k);
  defer cr_wipe(&k.b[0], 32);
  var msg: string = "hello";
  var box: uint8[64];
  var blen: int64 = 0;
  cr_seal(&box[0], 64, &blen, msg.chars, (int64)msg.length, 0, 0, &k);

  var out: uint8[64];
  var olen: int64 = 0;
  if (cr_open(&out[0], 64, &olen, &box[0], blen, 0, 0, &k) != CR_OK) { return 1; }
  return 0;
}
```

## What is in it

**Recipes** (start here):

| Need | Use |
|---|---|
| Sign a cookie or link | `cr_signed_sign`, `cr_signed_verify` |
| Sign a link that expires | `cr_signed_sign_until`, `cr_signed_verify_until` |
| API token or session id | `cr_token_new`, `cr_token_hash` |
| Recovery codes | `cr_code_new`, `cr_code_hash` |
| Encrypt a value | `cr_seal`, `cr_open` |

**Primitives:**

| Area | Functions | libsodium |
|---|---|---|
| Random, compare, wipe | `cr_random_bytes`, `cr_equal`, `cr_wipe` | `randombytes_*`, `sodium_memcmp`, `sodium_memzero` |
| Hashes | `cr_sha256`, `cr_sha512`, `cr_blake2b` | `crypto_hash_*`, `crypto_generichash` |
| MAC | `cr_hmac_sha256*`, `cr_hmac_sha512*` | `crypto_auth_hmacsha*` |
| Passwords | `cr_pwhash_str`, `cr_pwhash_verify`, `cr_pwhash_needs_rehash` | Argon2id |
| Encryption | `cr_aead_encrypt`, `cr_aead_decrypt` | XChaCha20-Poly1305 |
| Key exchange | `cr_x25519_*`, `cr_kx_*` | X25519 |
| Signatures | `cr_ed25519_*` | Ed25519 |
| Encoding | `cr_hex_*`, `cr_base64url_*` | `sodium_bin2hex`, `sodium_bin2base64` |

## Rules

- **Names:** functions `cr_`, types `Cr`, constants `CR_`.
- **Results:** functions return `CR_OK` (0) or a negative `CR_E_*` code. Output goes through pointers. Checks like `cr_equal` return `bool` and say `false` on any error.
- **Buffers:** you own every buffer. Cryptex never allocates and never keeps a pointer. If a buffer is too small you get `CR_E_SPACE` and nothing is written.
- **Main thread only.** Mettle can only call C libraries from the main thread (reports/001).
- **Wipe your secrets** with `cr_wipe` when done: `defer cr_wipe(&k.b[0], 32);`
- **Compare secrets** with `cr_equal`, never `==`.
- **One key per job.** Do not use a signing key for anything else.

## Password presets

| Preset | Cost | Use |
|---|---|---|
| `CR_PW_INTERACTIVE` | t=2, 64 MiB | logins (default, ~0.1 s) |
| `CR_PW_MODERATE` | t=3, 256 MiB | when you can wait ~0.5 s |
| `CR_PW_SENSITIVE` | t=4, 1 GiB | rare, high-value unlocks |

Store the text from `cr_pwhash_text`. After a good login, `cr_pwhash_needs_rehash` tells you when to store a new hash.

## Tests

```bash
export METTLE=/path/to/Mettle/bin/mettle
mbuild test        # debug
mbuild test -r     # release
```

Each `tests/<name>_test.mettle` is a test program. Test vectors come from the RFCs and were checked again with Python.

`tests/dependent` is a small project that uses Cryptex with no link flags of its own. Check it with `cd tests/dependent && mbuild run`.

## Docs

`docs/API.md` is built from the `///` comments in `src/`. After changing one:

```bash
python3 tools/api_docs.py
```

`python3 tools/api_docs.py --check` fails if the file is out of date.

## License

Apache License 2.0. See [LICENSE](LICENSE).
