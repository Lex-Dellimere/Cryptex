# Cryptex

Safe crypto for Mettle, written in **pure Mettle**.

No C, no shared libraries, no global state. Every function is safe on any thread. Randomness comes straight from the Linux `getrandom` system call.

- **API reference:** [docs/API.md](docs/API.md)
- **Known limits:** [reports/](reports/README.md)

> **Not audited yet.** The code follows the RFCs, matches libsodium byte for byte on thousands of test cases, and avoids secret-dependent branches by design. No outside party has reviewed it. Keep that in mind before you protect high-value data with it.

## Requirements

- Linux x86-64.
- The Mettle compiler and a current mbuild (with `mbuild test`).

## Install

In your `mbuild.mettle`:

```mettle
fn deps() {
  mb_dep_path("crypto", "../cryptex");
}
```

Then `import "crypto/crypto";`. Nothing to link: a program that uses Cryptex stays a static binary.

## Example

```mettle
import "crypto/crypto";

fn main() -> int32 {
  // Encrypt a value. The nonce is inside the output.
  var k: CrKey32;
  if (cr_aead_keygen(&k) != CR_OK) { return 1; }
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
| Two-factor codes (authenticator apps) | `cr_totp`, `cr_totp_verify`, `cr_totp_uri` |
| Encrypt a value | `cr_seal`, `cr_open` |

**Primitives:**

| Area | Functions | Standard |
|---|---|---|
| Random, compare, wipe | `cr_random_bytes`, `cr_equal`, `cr_wipe` | Linux `getrandom` |
| Hashes | `cr_sha256`, `cr_sha512`, `cr_blake2b` | FIPS 180-4, RFC 7693 |
| MAC | `cr_hmac_sha256*`, `cr_hmac_sha512*` | RFC 2104 |
| Key derivation | `cr_hkdf_sha256*` | RFC 5869 |
| Passwords | `cr_pwhash_str`, `cr_pwhash_verify`, `cr_pwhash_needs_rehash` | Argon2id, RFC 9106 |
| Encryption | `cr_aead_encrypt`, `cr_aead_decrypt` | XChaCha20-Poly1305 (RFC 8439 + draft-irtf-cfrg-xchacha) |
| Key exchange | `cr_x25519_*`, `cr_kx_*` | X25519, RFC 7748 |
| Signatures | `cr_ed25519_*` | Ed25519, RFC 8032 |
| Encoding | `cr_hex_*`, `cr_base64url_*` | RFC 4648 |

SHA-1 is inside, but only for TOTP. It is not offered on its own.

## Rules

- **Names:** functions `cr_`, types `Cr`, constants `CR_`. Names starting `crx_` are internal.
- **Results:** functions return `CR_OK` (0) or a negative `CR_E_*` code. Output goes through pointers. Checks like `cr_equal` return `bool` and say `false` on any error.
- **Buffers:** you own every buffer. Cryptex never allocates on your heap and never keeps a pointer. If a buffer is too small you get `CR_E_SPACE` and nothing is written. (Argon2id maps its own work memory and unmaps it before it returns.)
- **Any thread.** No function shares state. `cr_init` does nothing and is only kept so old code still compiles.
- **Wipe your secrets** with `cr_wipe` when done: `defer cr_wipe(&k.b[0], 32);`
- **Compare secrets** with `cr_equal`, never `==`.
- **One key per job.** Do not use a signing key for anything else.

## Password presets

| Preset | Cost | Use |
|---|---|---|
| `CR_PW_INTERACTIVE` | t=2, 64 MiB | logins (default, about 0.2 s) |
| `CR_PW_MODERATE` | t=3, 256 MiB | when you can wait about 1 s |
| `CR_PW_SENSITIVE` | t=4, 1 GiB | rare, high-value unlocks (several seconds) |

Store the text from `cr_pwhash_text`. After a good login, `cr_pwhash_needs_rehash` tells you when to store a new hash.

## Tests

```bash
export METTLE=/path/to/Mettle/bin/mettle
mbuild test        # debug
mbuild test -r     # release
```

Each `tests/<name>_test.mettle` is a test program. Two kinds of checks:

- **Standard vectors** from the RFCs and FIPS documents.
- **libsodium vectors:** `tests/vec_*.mettle`, thousands of random cases computed by libsodium (and Python for SHA-1 and HKDF). Cryptex must match them byte for byte. Rebuild them with `python3 -I tools/gen_vectors.py`. Only that script needs libsodium; the library and its tests do not.

`tests/dependent` is a small project that uses Cryptex the way your project would. Check it with `cd tests/dependent && mbuild run`.

## Docs

`docs/API.md` is built from the `///` comments in `src/`. After changing one:

```bash
python3 tools/api_docs.py
```

`python3 tools/api_docs.py --check` fails if the file is out of date.

## License

Apache License 2.0. See [LICENSE](LICENSE).
