# crypto

A thin, misuse-resistant Mettle API over **libsodium**. This library
implements **no cryptography in Mettle**: every primitive is a call into
libsodium through Mettle's C interop. What the Mettle side adds is fixed-size
types, argument checks, fail-closed results, wiping of temporaries, and a
documented contract on every export.

OpenSSL is not used: libsodium covers the whole scope.

## Requirements

- **libsodium 1.0.18 or newer**, runtime only (no headers needed). Debian 13
  ships it as `/usr/lib/x86_64-linux-gnu/libsodium.so.23` (package
  `libsodium23`). The newest symbols used (`crypto_pwhash_str_alg`,
  `sodium_bin2base64`, `crypto_kx_*`, XChaCha20-Poly1305) arrived in 1.0.12 to
  1.0.14; 1.0.18 is what the tests ran against.
- Linux x86-64 (the Mettle ELF linker binds shared libraries on that target).
- The dev Mettle compiler and mbuild 0.2.0.

## Quick start

```mettle
// mbuild.mettle of your project
import "mbuild/api";

fn project() {
  mb_package("app", "0.1.0");
}

fn deps() {
  mb_dep_path("crypto", "../crypto");
}
```

```mettle
// main.mettle
import "std/io";
import "crypto/crypto";

fn main() -> int32 {
  if (cr_init() != CR_OK) { return 1; }

  // Password storage
  var pw: string = "correct horse battery staple";
  var h: CrPwHash;
  if (cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_INTERACTIVE) != CR_OK) { return 1; }
  println(cr_pwhash_text(&h));                       // store this string
  if (cr_pwhash_verify(&h, pw.chars, (int64)pw.length)) { println("ok"); }

  // Authenticated encryption
  var k: CrKey32;
  cr_aead_keygen(&k);
  defer cr_wipe(&k.b[0], 32);
  var n: CrNonce24;
  cr_aead_nonce(&n);                                 // fresh for every message
  var msg: string = "hello";
  var ct: uint8[64];
  var clen: int64 = 0;
  cr_aead_encrypt(&ct[0], 64, &clen, msg.chars, (int64)msg.length, 0, 0, &n, &k);
  var pt: uint8[64];
  var plen: int64 = 0;
  if (cr_aead_decrypt(&pt[0], 64, &plen, &ct[0], clen, 0, 0, &n, &k) != CR_OK) { return 1; }
  return 0;
}
```

No link flags are needed in the dependent: crypto's `mbuild.mettle` declares
`mb_link_libs(":libsodium.so.23")` for Linux, and mbuild passes
`-l:libsodium.so.23 --rpath $ORIGIN` to every program whose graph contains
crypto. `tests/dependent/` is a project that does exactly this.

## Conventions

- **Prefixes:** functions `cr_`, types `Cr`, constants `CR_`.
- **Status:** functions return `int32`: `CR_OK` (0) or a negative `CR_E_*`
  (`ARG`, `SPACE`, `VERIFY`, `DECODE`, `KEY`, `INIT`, `INTERNAL`);
  `cr_status_name` names them. Results go through out-params.
- **Predicates return `bool`** and are fail-closed: `cr_equal`,
  `cr_hmac_*_verify`, `cr_ed25519_verify`, `cr_pwhash_verify` return `true`
  only for a positive, valid answer and `false` for every error.
  `cr_pwhash_needs_rehash` errs the other way (true unless certainly current).
- **Fixed-size types** (`CrKey32`, `CrNonce24`, `CrSha256`, `CrEd25519Sig`,
  ...) hold one field `b: uint8[N]`; pass `&x`, address bytes as `&x.b[0]`.
- **Buffers** are `uint8*` plus an `int64` length; a null pointer is allowed
  only with length 0. Outputs with a variable size take a capacity and
  report the written length; too small is `CR_E_SPACE` with nothing written.
- **Ownership:** every buffer is the caller's. The library never allocates
  on the Mettle heap, never keeps a pointer, and never returns a pointer into
  libsodium-owned memory. libsodium allocates internally only inside
  Argon2id, with libc's `malloc`/`free`, freed before the call returns; the
  program does not export Mettle's `malloc`, so the two allocators never
  meet (checked with `LD_DEBUG=bindings`).
- **Initialization:** `cr_init()` (sodium_init) is idempotent and called by
  every other function; call it at start-up to fail early.
- **Threads: main thread only.** Mettle may call foreign C only from the main
  thread (reports/001). libsodium is thread-safe, but not when called from a
  thread Mettle started.
- Each export has a `///` contract: summary, `in:`, `returns:`, `owns:`,
  `thread:`, `example:`.

## API and backing C functions

| Area | Mettle | libsodium |
|---|---|---|
| Init | `cr_init` | `sodium_init` |
| Random | `cr_random_bytes`, `cr_random_u32_below(n, &out)` | `randombytes_buf`, `randombytes_uniform` |
| Compare / wipe | `cr_equal` (constant time), `cr_wipe` | `sodium_memcmp`, `sodium_memzero` |
| Hashes | `cr_sha256`, `cr_sha512`, `cr_blake2b` (out 16..64, key 0 or 16..64) | `crypto_hash_sha256`, `crypto_hash_sha512`, `crypto_generichash` |
| MAC | `cr_hmac_sha256`, `_init/_update/_final`, `_verify`, `_state_bytes`; same for `sha512` | `crypto_auth_hmacsha256_*`, `crypto_auth_hmacsha512_*` (init/update/final, any key length) |
| Passwords | `cr_pwhash_str`, `cr_pwhash_verify`, `cr_pwhash_needs_rehash`, `cr_pwhash_text`, `cr_pwhash_from_text`, `cr_pwhash_preset_limits` | `crypto_pwhash_str_alg` (Argon2id13), `crypto_pwhash_str_verify`, `crypto_pwhash_argon2id_str_needs_rehash` |
| AEAD | `cr_aead_keygen`, `cr_aead_nonce`, `cr_aead_encrypt`, `cr_aead_decrypt` | `crypto_aead_xchacha20poly1305_ietf_{keygen,encrypt,decrypt}` |
| Key exchange | `cr_x25519_keypair`, `cr_x25519_public_from_secret`, `cr_x25519_shared`, `cr_kx_client_session_keys`, `cr_kx_server_session_keys` | `crypto_kx_keypair`, `crypto_scalarmult_base`, `crypto_scalarmult`, `crypto_kx_*_session_keys` |
| Signatures | `cr_ed25519_keypair`, `cr_ed25519_seed_keypair`, `cr_ed25519_sign`, `cr_ed25519_verify` | `crypto_sign_keypair`, `crypto_sign_seed_keypair`, `crypto_sign_detached`, `crypto_sign_verify_detached` |
| Encoding | `cr_hex_encode/_decode`, `cr_base64url_encode/_decode`, `cr_hex_encoded_cap`, `cr_base64url_encoded_cap` | `sodium_bin2hex`, `sodium_hex2bin`, `sodium_bin2base64` / `sodium_base642bin` (URL-safe, no padding) |

Sources: `src/` (one module per row, plus `status`, `types` and the facade
`crypto.mettle`, which re-exports everything).

## Password presets

Argon2id, parallelism 1, values read from the linked libsodium:

| Preset | opslimit (t) | memlimit (m) | Use |
|---|---|---|---|
| `CR_PW_INTERACTIVE` | 2 | 64 MiB | online logins (default; about 0.1 s) |
| `CR_PW_MODERATE` | 3 | 256 MiB | higher margin where latency allows (about 0.5 s) |
| `CR_PW_SENSITIVE` | 4 | 1 GiB | rare, high-value unlocks; several seconds, never per request |

Store the string from `cr_pwhash_text`; it carries the algorithm, costs and
salt. After a successful verify, `cr_pwhash_needs_rehash(&h, preset)` says
when to store a fresh hash with the current preset.

## Security notes

- **Never roll your own.** Do not add primitives here in Mettle. If libsodium
  lacks something, bind a vetted C library and test it against published
  vectors, as `tests/` does.
- **Wipe secrets.** Keys, seeds, X25519 shared points, session keys,
  plaintexts and passwords live in your memory: `cr_wipe` them as soon as
  you are done (`defer cr_wipe(&k.b[0], 32);`). Library temporaries (HMAC
  states, recomputed tags) are wiped for you, and failed decryptions wipe
  their output.
- **Compare secrets with `cr_equal`** (or the `_verify` functions), never
  `==` or `mem_equal`.
- **Nonces:** one `cr_aead_nonce` per message. Never reuse a key/nonce pair.
- **`CrX25519Shared` is not a key.** Use `cr_kx_*` session keys, or hash the
  shared point with both public keys in your protocol's KDF.
- **Encodings are strict**: anything non-canonical is `CR_E_DECODE`. That is
  intended for tokens; do not loosen it.

## Tests

```bash
export METTLE=/path/to/Mettle/bin/mettle
mbuild test-programs            # debug and release; --debug / --release for one
```

builds `tests/` (a package depending on crypto by path) and
`tests/dependent/`, then runs each suite, which prints one line:

```text
OK hash (29 checks)       FIPS 180-2 SHA-256/512 abc, "", 448-bit; RFC 7693 BLAKE2b-512 abc
OK mac (138 checks)       RFC 4231 cases 1-4, 6, 7, HMAC-SHA256/512, one-shot + streaming
OK pwhash (36 checks)     Argon2id round trip, wrong password, needs_rehash, presets
OK aead (32 checks)       draft-irtf-cfrg-xchacha A.3.1, fail-closed decryption
OK kx (23 checks)         RFC 7748 5.2 vector 1, 6.1 Alice/Bob, crypto_kx mirror
OK sign (26 checks)       RFC 8032 7.1 tests 1 and 2
OK encode (72 checks)     RFC 4648 vectors (URL alphabet), strict rejection, round trips
OK random (27 checks)     randombytes sanity, cr_equal, cr_wipe
OK dependent (...)        a dependent with no flags of its own links and runs
```

The vectors were also checked against Python's `hashlib`/`hmac` and
`cryptography` (OpenSSL), independently of libsodium. The suites are built
programs because `mbuild test` runs `@test` functions in the interpreter,
which cannot call C.

## Known limitations

See [reports/README.md](reports/README.md).
