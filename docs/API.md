# Cryptex API

Generated from the `///` contracts in `src/` by `tools/api_docs.py`. Do not edit by hand.

Every function is **main thread only** (see reports/001) unless it says otherwise.

Primitives: [status](#status), [types](#types), [core](#core), [hash](#hash), [mac](#mac), [pwhash](#pwhash), [aead](#aead), [kx](#kx), [sign](#sign), [encode](#encode).
Recipes: [signed](#signed), [token](#token), [seal](#seal).

## status

Status codes shared by every crypto module.

### CR_OK

```mettle
export const CR_OK: int32 = 0;
```

the operation succeeded.

- **returns:** 0
- **owns:** nothing
- **thread:** constant

```mettle
if (cr_sha256(&d, p, n) != CR_OK) { ... }
```

### CR_E_INIT

```mettle
export const CR_E_INIT: int32 = -1;
```

libsodium could not be initialized (sodium_init failed).

- **returns:** -1
- **owns:** nothing
- **thread:** constant

```mettle
if (cr_init() == CR_E_INIT) { abort(); }
```

### CR_E_ARG

```mettle
export const CR_E_ARG: int32 = -2;
```

an argument is invalid (null pointer with a non-zero length, negative length, length outside the primitive's allowed range).

- **returns:** -2
- **owns:** nothing
- **thread:** constant

```mettle
cr_random_bytes(0, 4) == CR_E_ARG
```

### CR_E_SPACE

```mettle
export const CR_E_SPACE: int32 = -3;
```

the caller's output buffer is too small. Nothing is written.

- **returns:** -3
- **owns:** nothing
- **thread:** constant

```mettle
cr_hex_encode(&buf[0], 2, &n, p, 8) == CR_E_SPACE
```

### CR_E_VERIFY

```mettle
export const CR_E_VERIFY: int32 = -4;
```

authentication failed (AEAD tag mismatch). The output is wiped.

- **returns:** -4
- **owns:** nothing
- **thread:** constant

```mettle
if (cr_aead_decrypt(...) == CR_E_VERIFY) { reject(); }
```

### CR_E_DECODE

```mettle
export const CR_E_DECODE: int32 = -5;
```

hex or base64url input is malformed or non-canonical.

- **returns:** -5
- **owns:** nothing
- **thread:** constant

```mettle
cr_hex_decode(&b[0], 4, &n, "zz", 2) == CR_E_DECODE
```

### CR_E_INTERNAL

```mettle
export const CR_E_INTERNAL: int32 = -6;
```

libsodium reported a failure that is not the caller's fault (for example Argon2id could not allocate its memory).

- **returns:** -6
- **owns:** nothing
- **thread:** constant

```mettle
if (rc == CR_E_INTERNAL) { retry_later(); }
```

### CR_E_KEY

```mettle
export const CR_E_KEY: int32 = -7;
```

a public key is unusable (X25519 result is all zero, i.e. the peer sent a low-order point). Outputs are wiped.

- **returns:** -7
- **owns:** nothing
- **thread:** constant

```mettle
if (cr_x25519_shared(&s, &sk, &peer) == CR_E_KEY) { reject(); }
```

### CR_E_EXPIRED

```mettle
export const CR_E_EXPIRED: int32 = -8;
```

a signed value is authentic but its expiry time has passed (cr_signed_verify_until). The value is not released.

- **returns:** -8
- **owns:** nothing
- **thread:** constant

```mettle
if (rc == CR_E_EXPIRED) { show("link expired"); }
```

### cr_status_name

```mettle
export fn cr_status_name(code: int32) -> string
```

Names a status code for logs.

- **in:** code, a CR_* status
- **returns:** a static string such as "CR_E_ARG" ("CR_E_UNKNOWN" otherwise)
- **owns:** nothing; the result points at a string literal, never free it
- **thread:** any thread (pure)

```mettle
println(cr_status_name(rc));
```

## types

Fixed-size value types. Sizes are part of the type, so a nonce cannot be passed where a key is expected and no length can be mixed up. Each type is one field, `b`, the raw bytes; address them as &v.b[0].

### CrKey32

```mettle
export struct CrKey32 {
  b: uint8[32];
}
```

A 32-byte symmetric secret key: XChaCha20-Poly1305 keys and the session keys cr_kx_* derive.

- **in:** - (zeroed when declared: `var x: CrKey32;`)
- **owns:** secret: wipe with cr_wipe(&k.b[0], 32) when done
- **thread:** plain value; no shared state

```mettle
var k: CrKey32; cr_aead_keygen(&k);
```

### CrNonce24

```mettle
export struct CrNonce24 {
  b: uint8[24];
}
```

A 24-byte XChaCha20-Poly1305 nonce. Random nonces of this size are safe to generate per message.

- **in:** - (zeroed when declared: `var x: CrNonce24;`)
- **owns:** public: sent alongside the ciphertext
- **thread:** plain value; no shared state

```mettle
var n: CrNonce24; cr_aead_nonce(&n);
```

### CrSha256

```mettle
export struct CrSha256 {
  b: uint8[32];
}
```

A SHA-256 digest.

- **in:** - (zeroed when declared: `var x: CrSha256;`)
- **owns:** public
- **thread:** plain value; no shared state

```mettle
var d: CrSha256; cr_sha256(&d, msg, len);
```

### CrSha512

```mettle
export struct CrSha512 {
  b: uint8[64];
}
```

A SHA-512 digest.

- **in:** - (zeroed when declared: `var x: CrSha512;`)
- **owns:** public
- **thread:** plain value; no shared state

```mettle
var d: CrSha512; cr_sha512(&d, msg, len);
```

### CrHmacSha256

```mettle
export struct CrHmacSha256 {
  b: uint8[32];
}
```

An HMAC-SHA256 tag.

- **in:** - (zeroed when declared: `var x: CrHmacSha256;`)
- **owns:** public, but compare it only with cr_hmac_sha256_verify or cr_equal
- **thread:** plain value; no shared state

```mettle
var t: CrHmacSha256; cr_hmac_sha256(&t, key, klen, msg, mlen);
```

### CrHmacSha512

```mettle
export struct CrHmacSha512 {
  b: uint8[64];
}
```

An HMAC-SHA512 tag.

- **in:** - (zeroed when declared: `var x: CrHmacSha512;`)
- **owns:** public, but compare it only with cr_hmac_sha512_verify or cr_equal
- **thread:** plain value; no shared state

```mettle
var t: CrHmacSha512; cr_hmac_sha512(&t, key, klen, msg, mlen);
```

### CrX25519Public

```mettle
export struct CrX25519Public {
  b: uint8[32];
}
```

An X25519 (Curve25519 Diffie-Hellman) public key; also the key type of cr_kx_*.

- **in:** - (zeroed when declared: `var x: CrX25519Public;`)
- **owns:** public
- **thread:** plain value; no shared state

```mettle
var pk: CrX25519Public; var sk: CrX25519Secret; cr_x25519_keypair(&pk, &sk);
```

### CrX25519Secret

```mettle
export struct CrX25519Secret {
  b: uint8[32];
}
```

An X25519 secret scalar; also the key type of cr_kx_*.

- **in:** - (zeroed when declared: `var x: CrX25519Secret;`)
- **owns:** secret: wipe with cr_wipe(&sk.b[0], 32) when done
- **thread:** plain value; no shared state

```mettle
var sk: CrX25519Secret; cr_x25519_keypair(&pk, &sk);
```

### CrX25519Shared

```mettle
export struct CrX25519Shared {
  b: uint8[32];
}
```

A raw X25519 shared point. Not a key: hash it (with both public keys) or use cr_kx_* session keys instead of using it directly.

- **in:** - (zeroed when declared: `var x: CrX25519Shared;`)
- **owns:** secret: wipe with cr_wipe(&s.b[0], 32) when done
- **thread:** plain value; no shared state

```mettle
var s: CrX25519Shared; cr_x25519_shared(&s, &sk, &peer);
```

### CrEd25519Public

```mettle
export struct CrEd25519Public {
  b: uint8[32];
}
```

An Ed25519 public (verification) key.

- **in:** - (zeroed when declared: `var x: CrEd25519Public;`)
- **owns:** public
- **thread:** plain value; no shared state

```mettle
var pk: CrEd25519Public; var sk: CrEd25519Secret; cr_ed25519_keypair(&pk, &sk);
```

### CrEd25519Secret

```mettle
export struct CrEd25519Secret {
  b: uint8[64];
}
```

An Ed25519 secret key in libsodium's layout: the 32-byte seed followed by the 32-byte public key.

- **in:** - (zeroed when declared: `var x: CrEd25519Secret;`)
- **owns:** secret: wipe with cr_wipe(&sk.b[0], 64) when done
- **thread:** plain value; no shared state

```mettle
var sk: CrEd25519Secret; cr_ed25519_keypair(&pk, &sk);
```

### CrSeed32

```mettle
export struct CrSeed32 {
  b: uint8[32];
}
```

A 32-byte seed for deterministic key generation (cr_ed25519_seed_keypair). Equal to the RFC 8032 private key.

- **in:** - (zeroed when declared: `var x: CrSeed32;`)
- **owns:** secret: wipe with cr_wipe(&s.b[0], 32) when done
- **thread:** plain value; no shared state

```mettle
var s: CrSeed32; cr_random_bytes(&s.b[0], 32);
```

### CrEd25519Sig

```mettle
export struct CrEd25519Sig {
  b: uint8[64];
}
```

A detached Ed25519 signature.

- **in:** - (zeroed when declared: `var x: CrEd25519Sig;`)
- **owns:** public
- **thread:** plain value; no shared state

```mettle
var sig: CrEd25519Sig; cr_ed25519_sign(&sig, msg, len, &sk);
```

### CrPwHash

```mettle
export struct CrPwHash {
  b: uint8[128];
}
```

An Argon2id password hash in libsodium's self-describing, nul-terminated string form ($argon2id$v=19$m=...,t=...,p=1$salt$hash). Store cr_pwhash_text(&h); the salt and costs travel inside it.

- **in:** - (zeroed when declared: `var x: CrPwHash;`)
- **owns:** not secret, but treat as sensitive (offline guessing)
- **thread:** plain value; no shared state

```mettle
var h: CrPwHash; cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_INTERACTIVE);
```

## core

Initialization, randomness, constant-time comparison and wiping. Backend: libsodium (sodium_init, randombytes_*, sodium_memcmp, sodium_memzero).

### cr_init

```mettle
export fn cr_init() -> int32
```

Initializes libsodium. Idempotent: the first call runs sodium_init, later calls return immediately. Every other cr_ function calls it for you, so an explicit call is only needed to fail early at program start.

- **returns:** CR_OK, or CR_E_INIT if libsodium cannot initialize (no entropy source); then every other function fails with CR_E_INIT / false
- **owns:** nothing
- **thread:** main thread only (reports/001). sodium_init is itself thread-safe; the Mettle-side flag is a plain global

```mettle
if (cr_init() != CR_OK) { return 1; }
```

### cr_random_bytes

```mettle
export fn cr_random_bytes(buf: uint8*, n: int64) -> int32
```

Fills buf with n cryptographically secure random bytes (randombytes_buf, getrandom(2) on Linux).

- **in:** buf, writable for n bytes (may be 0 only when n == 0); n >= 0
- **returns:** CR_OK; CR_E_ARG for a null buf with n > 0 or n < 0; CR_E_INIT
- **owns:** nothing; the caller owns buf
- **thread:** main thread only

```mettle
var salt: uint8[16]; cr_random_bytes(&salt[0], 16);
```

### cr_random_u32_below

```mettle
export fn cr_random_u32_below(n: uint32, out: uint32*) -> int32
```

Picks a uniformly distributed random integer in [0, n) without modulo bias (randombytes_uniform).

- **in:** n >= 1, the exclusive upper bound; out, where the value goes
- **returns:** CR_OK; CR_E_ARG if n == 0 or out is null (out untouched); CR_E_INIT
- **owns:** nothing
- **thread:** main thread only

```mettle
var die: uint32; cr_random_u32_below(6, &die);  // 0..5
```

### cr_equal

```mettle
export fn cr_equal(a: uint8*, b: uint8*, n: int64) -> bool
```

Compares two byte ranges in constant time (sodium_memcmp): the time depends on n only, never on the contents. Use it for every comparison of secrets, tags and tokens; never use == or mem_equal for those.

- **in:** a, b readable for n bytes; n >= 0
- **returns:** true only if all n bytes are equal; false otherwise, and false for invalid arguments (null pointer with n > 0, n < 0) or CR_E_INIT
- **owns:** nothing
- **thread:** main thread only

```mettle
if (cr_equal(&got.b[0], &want.b[0], 32)) { accept(); }
```

### cr_wipe

```mettle
export fn cr_wipe(buf: uint8*, n: int64) -> int32
```

Overwrites n bytes with zeros in a way the optimizer cannot remove (sodium_memzero). Call it on every secret (keys, seeds, passwords, plaintexts, HMAC states) as soon as you are done with it, including on error paths: `defer cr_wipe(&k.b[0], 32);`.

- **in:** buf writable for n bytes; n >= 0
- **returns:** CR_OK; CR_E_ARG for a null buf with n > 0 or n < 0
- **owns:** nothing; the caller still owns (and frees) buf
- **thread:** main thread only

```mettle
var k: CrKey32; cr_aead_keygen(&k); defer cr_wipe(&k.b[0], 32);
```

## hash

Hashes: SHA-256, SHA-512 (crypto_hash_sha256/512) and BLAKE2b (crypto_generichash). Backend: libsodium.

### CR_BLAKE2B_MIN

```mettle
export const CR_BLAKE2B_MIN: int64 = 16;
```

Smallest BLAKE2b output length cr_blake2b accepts (crypto_generichash_BYTES_MIN).

- **returns:** 16
- **owns:** nothing
- **thread:** constant

```mettle
cr_blake2b(&o[0], CR_BLAKE2B_MIN, m, n, 0, 0);
```

### CR_BLAKE2B_MAX

```mettle
export const CR_BLAKE2B_MAX: int64 = 64;
```

Largest BLAKE2b output and key length (crypto_generichash_BYTES_MAX / KEYBYTES_MAX).

- **returns:** 64
- **owns:** nothing
- **thread:** constant

```mettle
var o: uint8[64]; cr_blake2b(&o[0], CR_BLAKE2B_MAX, m, n, 0, 0);
```

### CR_BLAKE2B_KEY_MIN

```mettle
export const CR_BLAKE2B_KEY_MIN: int64 = 16;
```

Smallest non-empty BLAKE2b key (crypto_generichash_KEYBYTES_MIN).

- **returns:** 16
- **owns:** nothing
- **thread:** constant

```mettle
cr_blake2b(&o[0], 32, m, n, &k[0], CR_BLAKE2B_KEY_MIN);
```

### cr_sha256

```mettle
export fn cr_sha256(out: CrSha256*, msg: uint8*, n: int64) -> int32
```

Computes SHA-256 of n bytes (FIPS 180-4; libsodium crypto_hash_sha256).

- **in:** out; msg readable for n bytes (may be 0 when n == 0); n >= 0
- **returns:** CR_OK; CR_E_ARG (null out, null msg with n > 0, n < 0); CR_E_INIT. On error out is not written
- **owns:** nothing; the caller owns out and msg
- **thread:** main thread only

```mettle
var d: CrSha256; cr_sha256(&d, s.chars, (int64)s.length);
```

### cr_sha512

```mettle
export fn cr_sha512(out: CrSha512*, msg: uint8*, n: int64) -> int32
```

Computes SHA-512 of n bytes (FIPS 180-4; libsodium crypto_hash_sha512).

- **in:** out; msg readable for n bytes (may be 0 when n == 0); n >= 0
- **returns:** CR_OK; CR_E_ARG (null out, null msg with n > 0, n < 0); CR_E_INIT. On error out is not written
- **owns:** nothing; the caller owns out and msg
- **thread:** main thread only

```mettle
var d: CrSha512; cr_sha512(&d, s.chars, (int64)s.length);
```

### cr_blake2b

```mettle
export fn cr_blake2b(out: uint8*, outlen: int64, msg: uint8*, n: int64, key: uint8*, keylen: int64) -> int32
```

Computes BLAKE2b (RFC 7693; libsodium crypto_generichash), optionally keyed (then it is a MAC/PRF), with a chosen output length.

- **in:** out writable for outlen bytes, outlen in [16, 64]; msg readable for n bytes (0 allowed when n == 0), n >= 0; key readable for keylen bytes, keylen 0 (unkeyed; key may be 0) or in [16, 64]
- **returns:** CR_OK; CR_E_ARG for any length outside those ranges or a null pointer with a non-zero length; CR_E_INIT. On error out is not written
- **owns:** nothing; the caller owns out, msg and key (wipe the key yourself)
- **thread:** main thread only

```mettle
var d: uint8[32]; cr_blake2b(&d[0], 32, s.chars, (int64)s.length, 0, 0);
```

## mac

HMAC-SHA256 and HMAC-SHA512 (RFC 2104, FIPS 198-1) with keys of any length. Backend: libsodium crypto_auth_hmacsha256_* / _hmacsha512_* (init/update/final, so RFC 4231 keys of 4..131 bytes work; libsodium's one-shot crypto_auth_hmacsha256 takes a fixed 32-byte key only). Verification is constant time (sodium_memcmp).

### CrHmacSha256State

```mettle
export struct CrHmacSha256State {
  st: uint64[32];
}
```

an incremental HMAC-SHA256 computation (crypto_auth_hmacsha256_state, held in caller memory; 256 bytes reserved, libsodium needs 208).

- **in:** - (initialize with cr_hmac_sha256_init before use)
- **owns:** secret (it holds key-derived pads): cr_hmac_sha256_final wipes it; if you abandon a computation, call cr_wipe(cr_hmac_sha256_state_bytes(&st), 256)
- **thread:** one state per computation; main thread only

```mettle
var st: CrHmacSha256State; cr_hmac_sha256_init(&st, k, klen);
```

### cr_hmac_sha256_state_bytes

```mettle
export fn cr_hmac_sha256_state_bytes(st: CrHmacSha256State*) -> uint8*
```

Gives the address of a state's storage, for cr_wipe.

- **in:** st
- **returns:** a pointer to its 256 bytes (0 if st is 0)
- **owns:** the pointer borrows st; it is caller memory, not libsodium's
- **thread:** main thread only

```mettle
cr_wipe(cr_hmac_sha256_state_bytes(&st), 256);
```

### cr_hmac_sha256_init

```mettle
export fn cr_hmac_sha256_init(st: CrHmacSha256State*, key: uint8*, keylen: int64) -> int32
```

Starts an HMAC-SHA256 computation (RFC 2104 / FIPS 198-1) with a key of any length (longer than the block size it is hashed, as RFC 2104 says).

- **in:** st; key readable for keylen bytes, keylen >= 0 (32+ random bytes recommended; key may be 0 only when keylen == 0)
- **returns:** CR_OK; CR_E_ARG (null st, null key with keylen > 0, keylen < 0); CR_E_INIT; CR_E_INTERNAL if the linked libsodium's state is larger than this binding reserves
- **owns:** st stays the caller's; the key is copied into st's pads, so the caller may wipe its key right away
- **thread:** main thread only

```mettle
var st: CrHmacSha256State; cr_hmac_sha256_init(&st, &k[0], 32);
```

### cr_hmac_sha256_update

```mettle
export fn cr_hmac_sha256_update(st: CrHmacSha256State*, msg: uint8*, n: int64) -> int32
```

Feeds n more message bytes into an HMAC-SHA256 computation.

- **in:** st from cr_hmac_sha256_init; msg readable for n bytes; n >= 0
- **returns:** CR_OK; CR_E_ARG (null st, null msg with n > 0, n < 0)
- **owns:** nothing new
- **thread:** main thread only

```mettle
cr_hmac_sha256_update(&st, part.chars, (int64)part.length);
```

### cr_hmac_sha256_final

```mettle
export fn cr_hmac_sha256_final(st: CrHmacSha256State*, out: CrHmacSha256*) -> int32
```

Finishes an HMAC-SHA256 computation, writes the tag and wipes the state.

- **in:** st from cr_hmac_sha256_init; out
- **returns:** CR_OK; CR_E_ARG (null st or out)
- **owns:** st is wiped (start again with cr_hmac_sha256_init to reuse it)
- **thread:** main thread only

```mettle
var t: CrHmacSha256; cr_hmac_sha256_final(&st, &t);
```

### cr_hmac_sha256

```mettle
export fn cr_hmac_sha256(out: CrHmacSha256*, key: uint8*, keylen: int64, msg: uint8*, n: int64) -> int32
```

Computes HMAC-SHA256(key, msg) in one call (init + update + final), for a key of any length.

- **in:** out; key/keylen as cr_hmac_sha256_init; msg/n as cr_hmac_sha256_update
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_INTERNAL. On error out is left zeroed
- **owns:** nothing; the temporary state is wiped
- **thread:** main thread only

```mettle
var t: CrHmacSha256; cr_hmac_sha256(&t, &k[0], 32, m.chars, (int64)m.length);
```

### cr_hmac_sha256_verify

```mettle
export fn cr_hmac_sha256_verify(tag: CrHmacSha256*, key: uint8*, keylen: int64, msg: uint8*, n: int64) -> bool
```

Checks an HMAC-SHA256 tag in constant time: recomputes the tag and compares with sodium_memcmp.

- **in:** tag, the received tag; key/keylen and msg/n as cr_hmac_sha256
- **returns:** true only if the tag is valid; false for a wrong tag and for every error (fails closed)
- **owns:** nothing; the recomputed tag is wiped
- **thread:** main thread only

```mettle
if (!cr_hmac_sha256_verify(&t, &k[0], 32, m.chars, (int64)m.length)) { reject(); }
```

### CrHmacSha512State

```mettle
export struct CrHmacSha512State {
  st: uint64[64];
}
```

an incremental HMAC-SHA512 computation (crypto_auth_hmacsha512_state, held in caller memory; 512 bytes reserved, libsodium needs 416).

- **in:** - (initialize with cr_hmac_sha512_init before use)
- **owns:** secret (it holds key-derived pads): cr_hmac_sha512_final wipes it; if you abandon a computation, call cr_wipe(cr_hmac_sha512_state_bytes(&st), 512)
- **thread:** one state per computation; main thread only

```mettle
var st: CrHmacSha512State; cr_hmac_sha512_init(&st, k, klen);
```

### cr_hmac_sha512_state_bytes

```mettle
export fn cr_hmac_sha512_state_bytes(st: CrHmacSha512State*) -> uint8*
```

Gives the address of a state's storage, for cr_wipe.

- **in:** st
- **returns:** a pointer to its 512 bytes (0 if st is 0)
- **owns:** the pointer borrows st; it is caller memory, not libsodium's
- **thread:** main thread only

```mettle
cr_wipe(cr_hmac_sha512_state_bytes(&st), 512);
```

### cr_hmac_sha512_init

```mettle
export fn cr_hmac_sha512_init(st: CrHmacSha512State*, key: uint8*, keylen: int64) -> int32
```

Starts an HMAC-SHA512 computation (RFC 2104 / FIPS 198-1) with a key of any length (longer than the block size it is hashed, as RFC 2104 says).

- **in:** st; key readable for keylen bytes, keylen >= 0 (32+ random bytes recommended; key may be 0 only when keylen == 0)
- **returns:** CR_OK; CR_E_ARG (null st, null key with keylen > 0, keylen < 0); CR_E_INIT; CR_E_INTERNAL if the linked libsodium's state is larger than this binding reserves
- **owns:** st stays the caller's; the key is copied into st's pads, so the caller may wipe its key right away
- **thread:** main thread only

```mettle
var st: CrHmacSha512State; cr_hmac_sha512_init(&st, &k[0], 32);
```

### cr_hmac_sha512_update

```mettle
export fn cr_hmac_sha512_update(st: CrHmacSha512State*, msg: uint8*, n: int64) -> int32
```

Feeds n more message bytes into an HMAC-SHA512 computation.

- **in:** st from cr_hmac_sha512_init; msg readable for n bytes; n >= 0
- **returns:** CR_OK; CR_E_ARG (null st, null msg with n > 0, n < 0)
- **owns:** nothing new
- **thread:** main thread only

```mettle
cr_hmac_sha512_update(&st, part.chars, (int64)part.length);
```

### cr_hmac_sha512_final

```mettle
export fn cr_hmac_sha512_final(st: CrHmacSha512State*, out: CrHmacSha512*) -> int32
```

Finishes an HMAC-SHA512 computation, writes the tag and wipes the state.

- **in:** st from cr_hmac_sha512_init; out
- **returns:** CR_OK; CR_E_ARG (null st or out)
- **owns:** st is wiped (start again with cr_hmac_sha512_init to reuse it)
- **thread:** main thread only

```mettle
var t: CrHmacSha512; cr_hmac_sha512_final(&st, &t);
```

### cr_hmac_sha512

```mettle
export fn cr_hmac_sha512(out: CrHmacSha512*, key: uint8*, keylen: int64, msg: uint8*, n: int64) -> int32
```

Computes HMAC-SHA512(key, msg) in one call (init + update + final), for a key of any length.

- **in:** out; key/keylen as cr_hmac_sha512_init; msg/n as cr_hmac_sha512_update
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_INTERNAL. On error out is left zeroed
- **owns:** nothing; the temporary state is wiped
- **thread:** main thread only

```mettle
var t: CrHmacSha512; cr_hmac_sha512(&t, &k[0], 32, m.chars, (int64)m.length);
```

### cr_hmac_sha512_verify

```mettle
export fn cr_hmac_sha512_verify(tag: CrHmacSha512*, key: uint8*, keylen: int64, msg: uint8*, n: int64) -> bool
```

Checks an HMAC-SHA512 tag in constant time: recomputes the tag and compares with sodium_memcmp.

- **in:** tag, the received tag; key/keylen and msg/n as cr_hmac_sha512
- **returns:** true only if the tag is valid; false for a wrong tag and for every error (fails closed)
- **owns:** nothing; the recomputed tag is wiped
- **thread:** main thread only

```mettle
if (!cr_hmac_sha512_verify(&t, &k[0], 32, m.chars, (int64)m.length)) { reject(); }
```

## pwhash

Password hashing: Argon2id (RFC 9106) through libsodium's crypto_pwhash_str_alg / crypto_pwhash_str_verify / crypto_pwhash_argon2id_str_needs_rehash.

### CR_PW_INTERACTIVE

```mettle
export const CR_PW_INTERACTIVE: int32 = 1;
```

Preset for online logins: Argon2id t=2 (opslimit), m=64 MiB (crypto_pwhash_argon2id_*_INTERACTIVE). About 0.1 s and 64 MiB of RAM per hash on a desktop core; the right default for web logins.

- **returns:** 1
- **owns:** nothing
- **thread:** constant

```mettle
cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_INTERACTIVE);
```

### CR_PW_MODERATE

```mettle
export const CR_PW_MODERATE: int32 = 2;
```

Preset with a higher margin: Argon2id t=3, m=256 MiB (*_MODERATE). Roughly 0.5-0.7 s per hash.

- **returns:** 2
- **owns:** nothing
- **thread:** constant

```mettle
cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_MODERATE);
```

### CR_PW_SENSITIVE

```mettle
export const CR_PW_SENSITIVE: int32 = 3;
```

Preset for rarely used, high-value secrets (key files, vault unlock): Argon2id t=4, m=1 GiB (*_SENSITIVE). Several seconds and 1 GiB of RAM per hash; never use it on a server that hashes on request.

- **returns:** 3
- **owns:** nothing
- **thread:** constant

```mettle
cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_SENSITIVE);
```

### cr_pwhash_preset_limits

```mettle
export fn cr_pwhash_preset_limits(preset: int32, ops: uint64*, mem: uint64*) -> int32
```

Reads the Argon2id costs of a preset, as the linked libsodium defines them.

- **in:** preset, one of CR_PW_*; ops and mem, where the costs go
- **returns:** CR_OK; CR_E_ARG for an unknown preset or a null out-param
- **owns:** nothing
- **thread:** main thread only

```mettle
var t: uint64; var m: uint64; cr_pwhash_preset_limits(CR_PW_MODERATE, &t, &m);
```

### cr_pwhash_str

```mettle
export fn cr_pwhash_str(out: CrPwHash*, pw: uint8*, n: int64, preset: int32) -> int32
```

Hashes a password with Argon2id and a fresh random salt.

- **in:** out; pw readable for n bytes (any bytes; 0 allowed when n == 0), n >= 0; preset, one of CR_PW_INTERACTIVE / MODERATE / SENSITIVE
- **returns:** CR_OK; CR_E_ARG (null out, null pw with n > 0, n < 0, unknown preset); CR_E_INIT; CR_E_INTERNAL if Argon2id could not allocate its memory. On error out is all zeros
- **owns:** nothing; libsodium's working memory is allocated and freed inside the call by libc (never by Mettle's allocator). The caller should wipe the plaintext password after the call
- **thread:** main thread only. Blocks for the preset's time

```mettle
var h: CrPwHash; if (cr_pwhash_str(&h, pw.chars, (int64)pw.length, CR_PW_INTERACTIVE) == CR_OK) { store(cr_pwhash_text(&h)); }
```

### cr_pwhash_verify

```mettle
export fn cr_pwhash_verify(h: CrPwHash*, pw: uint8*, n: int64) -> bool
```

Checks a password against a stored hash (constant time inside libsodium). Accepts any Argon2 string libsodium understands (argon2id, legacy argon2i); use cr_pwhash_needs_rehash afterwards to upgrade.

- **in:** h, a hash from cr_pwhash_str or cr_pwhash_from_text; pw/n the candidate password
- **returns:** true only if the password matches; false for a wrong password, a malformed or unterminated hash, any argument error, or an allocation failure inside Argon2id (fails closed)
- **owns:** nothing
- **thread:** main thread only. Takes as long as the hash's own costs

```mettle
if (cr_pwhash_verify(&h, pw.chars, (int64)pw.length)) { login(); }
```

### cr_pwhash_needs_rehash

```mettle
export fn cr_pwhash_needs_rehash(h: CrPwHash*, preset: int32) -> bool
```

Tells whether a stored hash should be replaced: it is not Argon2id, or its costs differ from the preset. Call it after a successful cr_pwhash_verify and, when true, store cr_pwhash_str of the same password.

- **in:** h; preset, one of CR_PW_*
- **returns:** false only if h is a well-formed Argon2id string with exactly the preset's costs; true otherwise (including malformed hashes and argument errors, so a caller that rehashes is never left on old costs)
- **owns:** nothing
- **thread:** main thread only

```mettle
if (cr_pwhash_needs_rehash(&h, CR_PW_INTERACTIVE)) { rehash(); }
```

### cr_pwhash_text

```mettle
export fn cr_pwhash_text(h: CrPwHash*) -> string
```

Views a hash as text for storage (without the terminating nul).

- **in:** h
- **returns:** a string view of h's bytes up to the nul; empty if h is 0 or unterminated
- **owns:** the view borrows h (caller memory, never libsodium's); copy it before h goes away
- **thread:** main thread only

```mettle
db_put(user, cr_pwhash_text(&h));
```

### cr_pwhash_from_text

```mettle
export fn cr_pwhash_from_text(out: CrPwHash*, text: string) -> int32
```

Loads a stored hash string into a CrPwHash. Only the shape is checked here (length 1..127, no nul bytes, "$argon2" prefix); libsodium parses the rest when verifying.

- **in:** out; text, the string cr_pwhash_text gave you
- **returns:** CR_OK; CR_E_ARG (null out); CR_E_DECODE if text cannot be a libsodium Argon2 string. On error out is all zeros
- **owns:** text is copied; the caller keeps ownership of it
- **thread:** main thread only

```mettle
var h: CrPwHash; if (cr_pwhash_from_text(&h, row.hash) == CR_OK) { ... }
```

## aead

Authenticated encryption: XChaCha20-Poly1305-IETF (draft-irtf-cfrg-xchacha; libsodium crypto_aead_xchacha20poly1305_ietf_*).

### CR_AEAD_TAG_BYTES

```mettle
export const CR_AEAD_TAG_BYTES: int64 = 16;
```

Bytes the authentication tag adds to every ciphertext (crypto_aead_xchacha20poly1305_ietf_ABYTES).

- **returns:** 16
- **owns:** nothing
- **thread:** constant

```mettle
var cap: int64 = mlen + CR_AEAD_TAG_BYTES;
```

### cr_aead_keygen

```mettle
export fn cr_aead_keygen(k: CrKey32*) -> int32
```

Generates a random XChaCha20-Poly1305 key.

- **in:** k
- **returns:** CR_OK; CR_E_ARG (null k); CR_E_INIT
- **owns:** k is secret caller memory: cr_wipe(&k.b[0], 32) when done
- **thread:** main thread only

```mettle
var k: CrKey32; cr_aead_keygen(&k); defer cr_wipe(&k.b[0], 32);
```

### cr_aead_nonce

```mettle
export fn cr_aead_nonce(n: CrNonce24*) -> int32
```

Generates a random 24-byte nonce. Call it once per message.

- **in:** n
- **returns:** CR_OK; CR_E_ARG (null n); CR_E_INIT
- **owns:** nothing; the nonce is public and travels with the ciphertext
- **thread:** main thread only

```mettle
var nonce: CrNonce24; cr_aead_nonce(&nonce);
```

### cr_aead_encrypt

```mettle
export fn cr_aead_encrypt(ct: uint8*, ct_cap: int64, ct_len: int64*, msg: uint8*, mlen: int64, ad: uint8*, adlen: int64, nonce: CrNonce24*, key: CrKey32*) -> int32
```

Encrypts and authenticates msg, and authenticates (does not encrypt) ad. Output is ciphertext || 16-byte tag.

- **in:** ct writable for ct_cap bytes, ct_cap >= mlen + CR_AEAD_TAG_BYTES (ct may equal msg for in-place encryption; no other overlap); ct_len, where the ciphertext length goes; msg/mlen the plaintext; ad/adlen associated data (0, 0 for none); nonce, unique per key (cr_aead_nonce); key
- **returns:** CR_OK and *ct_len = mlen + 16; CR_E_ARG (null pointer with a non-zero length, negative length, null nonce/key/ct_len); CR_E_SPACE if ct_cap is too small (nothing written, *ct_len = 0); CR_E_INIT
- **owns:** nothing; the caller owns every buffer and must wipe key and plaintext when done
- **thread:** main thread only

```mettle
cr_aead_encrypt(&ct[0], 64, &clen, m.chars, (int64)m.length, 0, 0, &nonce, &k);
```

### cr_aead_decrypt

```mettle
export fn cr_aead_decrypt(pt: uint8*, pt_cap: int64, pt_len: int64*, ct: uint8*, clen: int64, ad: uint8*, adlen: int64, nonce: CrNonce24*, key: CrKey32*) -> int32
```

Verifies and decrypts ct (ciphertext || tag) produced by cr_aead_encrypt with the same key, nonce and ad. Fails closed.

- **in:** pt writable for pt_cap bytes, pt_cap >= clen - CR_AEAD_TAG_BYTES (pt may equal ct for in-place decryption); pt_len; ct/clen; ad/adlen; nonce; key
- **returns:** CR_OK and *pt_len = clen - 16; CR_E_VERIFY if the tag does not match (wrong key, nonce or ad, tampered or truncated ciphertext, clen < 16); CR_E_ARG; CR_E_SPACE; CR_E_INIT. On every failure *pt_len = 0 and the first pt_cap bytes of pt are wiped (sodium_memzero), so no unauthenticated plaintext ever reaches the caller (when decrypting in place, pt == ct, that also wipes the ciphertext)
- **owns:** nothing; the caller owns every buffer and must wipe the plaintext when done
- **thread:** main thread only

```mettle
if (cr_aead_decrypt(&pt[0], 64, &plen, &ct[0], clen, 0, 0, &nonce, &k) != CR_OK) { reject(); }
```

## kx

Key exchange: X25519 (RFC 7748; libsodium crypto_scalarmult_curve25519 via crypto_scalarmult / crypto_scalarmult_base) and libsodium's crypto_kx session-key derivation (BLAKE2b-512 of the X25519 point and both public keys, split into a receive key and a transmit key).

### cr_x25519_keypair

```mettle
export fn cr_x25519_keypair(pk: CrX25519Public*, sk: CrX25519Secret*) -> int32
```

Generates a random X25519 key pair (crypto_kx_keypair). The pair works for cr_x25519_shared and for cr_kx_*.

- **in:** pk, sk
- **returns:** CR_OK; CR_E_ARG (null pointer); CR_E_INIT; CR_E_INTERNAL
- **owns:** sk is secret caller memory: cr_wipe(&sk.b[0], 32) when done
- **thread:** main thread only

```mettle
var pk: CrX25519Public; var sk: CrX25519Secret; cr_x25519_keypair(&pk, &sk);
```

### cr_x25519_public_from_secret

```mettle
export fn cr_x25519_public_from_secret(pk: CrX25519Public*, sk: CrX25519Secret*) -> int32
```

Computes the public key of a secret scalar: X25519(sk, 9) (crypto_scalarmult_base; the scalar is clamped as RFC 7748 says).

- **in:** pk; sk
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_KEY if the result is zero
- **owns:** nothing
- **thread:** main thread only

```mettle
cr_x25519_public_from_secret(&pk, &sk);
```

### cr_x25519_shared

```mettle
export fn cr_x25519_shared(out: CrX25519Shared*, sk: CrX25519Secret*, peer: CrX25519Public*) -> int32
```

Computes the raw X25519 shared point X25519(sk, peer) (crypto_scalarmult). This is not a key: feed it, with both public keys, into a KDF (as Noise does) or use cr_kx_* instead.

- **in:** out; sk, our secret; peer, the other side's public key
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_KEY if peer is a low-order point (the result would be all zero; out is wiped)
- **owns:** out is secret caller memory: cr_wipe(&out.b[0], 32) when done
- **thread:** main thread only

```mettle
var s: CrX25519Shared; if (cr_x25519_shared(&s, &sk, &peer) != CR_OK) { reject(); }
```

### cr_kx_client_session_keys

```mettle
export fn cr_kx_client_session_keys(rx: CrKey32*, tx: CrKey32*, client_pk: CrX25519Public*, client_sk: CrX25519Secret*, server_pk: CrX25519Public*) -> int32
```

Derives the client's session keys (crypto_kx_client_session_keys): rx decrypts what the server sends, tx encrypts what the client sends. The server's cr_kx_server_session_keys gives the mirror image (server rx == client tx, server tx == client rx).

- **in:** rx, tx; client_pk, client_sk (cr_x25519_keypair); server_pk
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_KEY if server_pk is unusable (rx and tx are wiped)
- **owns:** rx and tx are secret caller memory: wipe them when done
- **thread:** main thread only

```mettle
var rx: CrKey32; var tx: CrKey32; cr_kx_client_session_keys(&rx, &tx, &cpk, &csk, &spk);
```

### cr_kx_server_session_keys

```mettle
export fn cr_kx_server_session_keys(rx: CrKey32*, tx: CrKey32*, server_pk: CrX25519Public*, server_sk: CrX25519Secret*, client_pk: CrX25519Public*) -> int32
```

Derives the server's session keys (crypto_kx_server_session_keys): rx decrypts what the client sends, tx encrypts what the server sends.

- **in:** rx, tx; server_pk, server_sk; client_pk
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_KEY if client_pk is unusable (rx and tx are wiped)
- **owns:** rx and tx are secret caller memory: wipe them when done
- **thread:** main thread only

```mettle
cr_kx_server_session_keys(&rx, &tx, &spk, &ssk, &cpk);
```

## sign

Signatures: Ed25519 (RFC 8032; libsodium crypto_sign_*), detached form. libsodium's secret key is 64 bytes: the RFC 8032 32-byte private key (the seed) followed by the public key.

### cr_ed25519_keypair

```mettle
export fn cr_ed25519_keypair(pk: CrEd25519Public*, sk: CrEd25519Secret*) -> int32
```

Generates a random Ed25519 key pair (crypto_sign_keypair).

- **in:** pk, sk
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_INTERNAL
- **owns:** sk is secret caller memory: cr_wipe(&sk.b[0], 64) when done
- **thread:** main thread only

```mettle
var pk: CrEd25519Public; var sk: CrEd25519Secret; cr_ed25519_keypair(&pk, &sk);
```

### cr_ed25519_seed_keypair

```mettle
export fn cr_ed25519_seed_keypair(pk: CrEd25519Public*, sk: CrEd25519Secret*, seed: CrSeed32*) -> int32
```

Derives the Ed25519 key pair of a 32-byte seed (the RFC 8032 private key) deterministically (crypto_sign_seed_keypair).

- **in:** pk, sk; seed
- **returns:** CR_OK; CR_E_ARG; CR_E_INIT; CR_E_INTERNAL
- **owns:** seed and sk are secret caller memory: wipe both when done
- **thread:** main thread only

```mettle
cr_ed25519_seed_keypair(&pk, &sk, &seed);
```

### cr_ed25519_sign

```mettle
export fn cr_ed25519_sign(sig: CrEd25519Sig*, msg: uint8*, n: int64, sk: CrEd25519Secret*) -> int32
```

Signs msg, producing a detached 64-byte signature (crypto_sign_detached). Ed25519 is deterministic: the same key and message give the same signature.

- **in:** sig; msg/n (0 allowed when n == 0); sk
- **returns:** CR_OK; CR_E_ARG (null sig/sk, null msg with n > 0, n < 0); CR_E_INIT; CR_E_INTERNAL. On error sig is zeroed
- **owns:** nothing; the caller owns sk and must wipe it when done
- **thread:** main thread only

```mettle
var sig: CrEd25519Sig; cr_ed25519_sign(&sig, m.chars, (int64)m.length, &sk);
```

### cr_ed25519_verify

```mettle
export fn cr_ed25519_verify(sig: CrEd25519Sig*, msg: uint8*, n: int64, pk: CrEd25519Public*) -> bool
```

Verifies a detached Ed25519 signature (crypto_sign_verify_detached, which rejects non-canonical and small-order encodings).

- **in:** sig; msg/n; pk, the signer's public key
- **returns:** true only if sig is a valid signature of exactly these n bytes under pk; false otherwise and for every argument error (fails closed)
- **owns:** nothing
- **thread:** main thread only

```mettle
if (!cr_ed25519_verify(&sig, m.chars, (int64)m.length, &pk)) { reject(); }
```

## encode

Encodings for tokens and keys: lowercase hex and base64url without padding (RFC 4648 section 5). Backend: libsodium sodium_bin2hex / sodium_hex2bin / sodium_bin2base64 / sodium_base642bin (constant-time implementations, so encoding secrets does not leak them through timing).

### cr_hex_encoded_cap

```mettle
export fn cr_hex_encoded_cap(n: int64) -> int64
```

Capacity cr_hex_encode needs for n input bytes: 2n + 1 (the nul).

- **in:** n >= 0
- **returns:** 2n + 1, or -1 for a negative or absurdly large n
- **owns:** nothing
- **thread:** any thread (pure)

```mettle
var cap: int64 = cr_hex_encoded_cap(32);  // 65
```

### cr_base64url_encoded_cap

```mettle
export fn cr_base64url_encoded_cap(n: int64) -> int64
```

Capacity cr_base64url_encode needs for n input bytes, nul included (sodium_base64_encoded_len with the URL-safe no-padding variant).

- **in:** n >= 0
- **returns:** the capacity, or -1 for a negative or absurdly large n
- **owns:** nothing
- **thread:** main thread only (calls libsodium)

```mettle
var cap: int64 = cr_base64url_encoded_cap(32);  // 44
```

### cr_hex_encode

```mettle
export fn cr_hex_encode(out: uint8*, out_cap: int64, out_len: int64*, bin: uint8*, n: int64) -> int32
```

Encodes n bytes as lowercase hex.

- **in:** out writable for out_cap >= cr_hex_encoded_cap(n) bytes; out_len; bin/n
- **returns:** CR_OK and *out_len = 2n (text followed by a nul); CR_E_ARG; CR_E_SPACE if out_cap is too small (nothing written); CR_E_INIT. On error *out_len = 0
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var hx: uint8[65]; var hl: int64; cr_hex_encode(&hx[0], 65, &hl, &d.b[0], 32);
```

### cr_hex_decode

```mettle
export fn cr_hex_decode(out: uint8*, out_cap: int64, out_len: int64*, text: uint8*, n: int64) -> int32
```

Decodes hex (either case) strictly.

- **in:** out writable for out_cap bytes; out_len; text/n, the hex digits (no nul needed)
- **returns:** CR_OK and *out_len = n / 2; CR_E_DECODE for an odd length or any non-hex character (including spaces and a trailing nul); CR_E_SPACE if out_cap < n / 2; CR_E_ARG; CR_E_INIT. On error *out_len = 0 and out's first out_cap bytes are wiped
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var k: uint8[32]; var kl: int64; cr_hex_decode(&k[0], 32, &kl, s.chars, (int64)s.length);
```

### cr_base64url_encode

```mettle
export fn cr_base64url_encode(out: uint8*, out_cap: int64, out_len: int64*, bin: uint8*, n: int64) -> int32
```

Encodes n bytes as base64url without padding (alphabet A-Z a-z 0-9 - _), the form used in URLs, cookies and tokens.

- **in:** out writable for out_cap >= cr_base64url_encoded_cap(n) bytes; out_len; bin/n
- **returns:** CR_OK and *out_len = the text length (a nul follows it); CR_E_ARG; CR_E_SPACE if out_cap is too small (nothing written); CR_E_INIT. On error *out_len = 0
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var t: uint8[44]; var tl: int64; cr_base64url_encode(&t[0], 44, &tl, &raw[0], 32);
```

### cr_base64url_decode

```mettle
export fn cr_base64url_decode(out: uint8*, out_cap: int64, out_len: int64*, text: uint8*, n: int64) -> int32
```

Decodes unpadded base64url strictly.

- **in:** out writable for out_cap bytes; out_len; text/n (no nul needed)
- **returns:** CR_OK and *out_len = the decoded length; CR_E_DECODE for any character outside the base64url alphabet (including '=', '+', '/', spaces), a length of the form 4k+1, or non-zero trailing bits; CR_E_SPACE if out_cap < floor(3n / 4); CR_E_ARG; CR_E_INIT. On error *out_len = 0 and out's first out_cap bytes are wiped
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var raw: uint8[32]; var rl: int64; cr_base64url_decode(&raw[0], 32, &rl, t.chars, (int64)t.length);
```

## signed

Signed values: "value.SIG", a readable value with an HMAC-SHA256 tag. SIG = base64url, no padding (always 43 chars), of HMAC-SHA256(key, le64(len(purpose)) || purpose || value) where le64 is the purpose length as 8 bytes, little endian. The purpose ("remember", "unsubscribe") separates uses: a value signed for one purpose does not verify for another. The value stays readable by anyone; only key holders can produce a valid tag. The format is byte-compatible with the session library's signed values (same formula).

### CR_SIGNED_EXTRA

```mettle
export const CR_SIGNED_EXTRA: int64 = 44;
```

the bytes cr_signed_sign adds to a value: '.' plus the 43-character tag.

- **returns:** 44
- **owns:** nothing
- **thread:** constant

```mettle
var cap: int64 = (int64)v.length + CR_SIGNED_EXTRA;
```

### CR_SIGNED_MAX_KEYS

```mettle
export const CR_SIGNED_MAX_KEYS: int32 = 8;
```

the most keys cr_signed_verify accepts.

- **returns:** 8
- **owns:** nothing
- **thread:** constant

```mettle
if (nkeys > CR_SIGNED_MAX_KEYS) { ... }
```

### cr_signed_sign

```mettle
export fn cr_signed_sign(out: uint8*, out_cap: int64, out_len: int64*, key: CrKey32*, purpose: string, value: string) -> int32
```

Signs a value for a purpose: writes "value.SIG" (the value, a '.', and the 43-character base64url tag of HMAC-SHA256(key, le64(len(purpose)) || purpose || value)). There is no nul terminator. The value is not encrypted: anyone can read it.

- **in:** out writable for out_cap >= value.length + CR_SIGNED_EXTRA bytes, and not overlapping value; out_len; key (a CrKey32, 32+ random bytes); purpose, 1..255 bytes; value, any bytes (may be empty)
- **returns:** CR_OK and *out_len = value.length + 44; CR_E_ARG (null out_len, out or key; purpose length outside 1..255); CR_E_SPACE if out_cap is too small (nothing written); CR_E_INIT; CR_E_INTERNAL. On error *out_len = 0
- **owns:** nothing; the caller owns out, the tag and encode buffers are wiped
- **thread:** main thread only

```mettle
var buf: uint8[128]; var n: int64; cr_signed_sign(&buf[0], 128, &n, &k, "remember", "user:42");
```

### cr_signed_verify

```mettle
export fn cr_signed_verify(out_value: string*, key_index: int32*, keys: CrKey32*, nkeys: int32, purpose: string, signed_value: string) -> int32
```

Verifies a "value.SIG" string against up to 8 keys for a purpose and returns the value. keys[0] is the current key; keys[1..] are older keys still accepted during rotation. The tag is recomputed with every key (no early exit) and compared in constant time; the lowest matching index is reported. Everything malformed fails closed with CR_E_VERIFY: too short, no '.' before the 43-character tag, a tag that is not canonical base64url of 32 bytes, or no key matching.

- **in:** out_value; key_index (may be 0); keys readable for nkeys CrKey32, 1..CR_SIGNED_MAX_KEYS of them; purpose, 1..255 bytes, the same as when signing; signed_value
- **returns:** CR_OK with *out_value the value and *key_index the matching key (key_index > 0 means the value was signed with an older key: re-sign it with keys[0]); CR_E_ARG (null out_value or keys, nkeys outside 1..8, purpose length outside 1..255); CR_E_VERIFY; CR_E_INIT; CR_E_INTERNAL. On error *out_value = "" and *key_index = -1
- **owns:** *out_value borrows signed_value (a view into it, not a copy; do not free it, and keep signed_value alive while it is used)
- **thread:** main thread only

```mettle
var v: string; var ki: int32; if (cr_signed_verify(&v, &ki, &keys[0], 2, "remember", cookie) == CR_OK) { ... }
```

### CR_SIGNED_UNTIL_EXTRA

```mettle
export const CR_SIGNED_UNTIL_EXTRA: int64 = 61;
```

the bytes cr_signed_sign_until adds to a value: '.', 16 hex digits of the expiry, '.', and the 43-character tag.

- **returns:** 61
- **owns:** nothing
- **thread:** constant

```mettle
var cap: int64 = (int64)v.length + CR_SIGNED_UNTIL_EXTRA;
```

### cr_signed_sign_until

```mettle
export fn cr_signed_sign_until(out: uint8*, out_cap: int64, out_len: int64*, key: CrKey32*, purpose: string, value: string, expires: int64) -> int32
```

Signs a value with an expiry time: writes "value.EXP.SIG" (the value, a '.', EXP, a '.', and the 43-character base64url tag SIG). EXP is exactly 16 lowercase hex digits of the expiry as an unsigned 64-bit big-endian number (Unix seconds; the caller picks the clock). SIG is the tag of HMAC-SHA256(key, "cryptex/signed-until/v1" || le64(len(purpose)) || purpose || le64(expires) || value), which is domain-separated from the plain cr_signed_sign form: a value from one form never verifies as the other. There is no nul terminator. The value is not encrypted: anyone can read it and the expiry.

- **in:** out writable for out_cap >= value.length + CR_SIGNED_UNTIL_EXTRA bytes, and not overlapping value; out_len; key (a CrKey32, 32+ random bytes); purpose, 1..255 bytes; value, any bytes (may be empty); expires, the absolute expiry time, >= 0
- **returns:** CR_OK and *out_len = value.length + 61; CR_E_ARG (null out_len, out or key; purpose length outside 1..255; expires < 0); CR_E_SPACE if out_cap is too small (nothing written); CR_E_INIT; CR_E_INTERNAL. On error *out_len = 0
- **owns:** nothing; the caller owns out, the tag and encode buffers are wiped
- **thread:** main thread only

```mettle
var buf: uint8[128]; var n: int64; cr_signed_sign_until(&buf[0], 128, &n, &k, "confirm-email", "user:42", now + 3600);
```

### cr_signed_verify_until

```mettle
export fn cr_signed_verify_until(out_value: string*, key_index: int32*, keys: CrKey32*, nkeys: int32, purpose: string, signed_value: string, now: int64) -> int32
```

Verifies a "value.EXP.SIG" string against up to 8 keys for a purpose, then checks the expiry, and returns the value. keys[0] is the current key; keys[1..] are older keys still accepted during rotation. The tag is recomputed with every key (no early exit) and compared in constant time; the lowest matching index is reported. Everything malformed fails closed with CR_E_VERIFY: too short, a '.' missing before EXP or before the 43-character tag, an EXP that is not exactly 16 lowercase hex digits (uppercase is rejected: the encoding is canonical) or has its top bit set, a tag that is not canonical base64url of 32 bytes, or no key matching. The expiry is checked only after a key matched, so an unauthenticated value never reports CR_E_EXPIRED. expires is an absolute time bound by the MAC: a verifier cannot extend it, and changing EXP invalidates the tag. Pass the current time (the same clock used when signing) as now.

- **in:** out_value; key_index (may be 0); keys readable for nkeys CrKey32, 1..CR_SIGNED_MAX_KEYS of them; purpose, 1..255 bytes, the same as when signing; signed_value; now, the current time in the signer's units (Unix seconds)
- **returns:** CR_OK with *out_value the value and *key_index the matching key (key_index > 0 means the value was signed with an older key: re-sign it with keys[0]), when now < expires; CR_E_EXPIRED when authentic but now >= expires; CR_E_ARG (null out_value or keys, nkeys outside 1..8, purpose length outside 1..255); CR_E_VERIFY; CR_E_INIT; CR_E_INTERNAL. On any error *out_value = "" and *key_index = -1
- **owns:** *out_value borrows signed_value (a view into it, not a copy; do not free it, and keep signed_value alive while it is used)
- **thread:** main thread only

```mettle
var v: string; var ki: int32; if (cr_signed_verify_until(&v, &ki, &keys[0], 2, "confirm-email", token, now) == CR_OK) { ... }
```

## token

Machine tokens and human codes. They look alike (random characters from an alphabet) but they are different things and are stored differently.

### CR_TOKEN_BASE62

```mettle
export const CR_TOKEN_BASE62: int32 = 1;
```

Alphabet selector: base62, 0-9 A-Z a-z (5.954 bits per character).

- **returns:** 1
- **owns:** nothing
- **thread:** constant

```mettle
cr_token_new(&t[0], 64, &n, "mtlr_", CR_TOKEN_BASE62, 40);
```

### CR_TOKEN_BASE32

```mettle
export const CR_TOKEN_BASE32: int32 = 2;
```

Alphabet selector: Crockford base32, lowercase, without i l o u (5 bits per character).

- **returns:** 2
- **owns:** nothing
- **thread:** constant

```mettle
cr_token_new(&t[0], 32, &n, "", CR_TOKEN_BASE32, 26);
```

### CR_TOKEN_MIN_BITS

```mettle
export const CR_TOKEN_MIN_BITS: int64 = 128;
```

Fewest bits of entropy a token may have (base62 needs 22 characters, base32 26).

- **returns:** 128
- **owns:** nothing
- **thread:** constant

```mettle
if (cr_token_bits(CR_TOKEN_BASE62, 22) >= CR_TOKEN_MIN_BITS) { ok(); }
```

### CR_TOKEN_MAX_CHARS

```mettle
export const CR_TOKEN_MAX_CHARS: int64 = 256;
```

Most random characters a token may have.

- **returns:** 256
- **owns:** nothing
- **thread:** constant

```mettle
var t: uint8[300]; cr_token_new(&t[0], 300, &n, "", CR_TOKEN_BASE62, CR_TOKEN_MAX_CHARS);
```

### CR_TOKEN_MAX_PREFIX

```mettle
export const CR_TOKEN_MAX_PREFIX: int64 = 32;
```

Longest prefix, in bytes.

- **returns:** 32
- **owns:** nothing
- **thread:** constant

```mettle
var cap: int64 = CR_TOKEN_MAX_PREFIX + CR_TOKEN_MAX_CHARS + 1;
```

### CR_CODE_MIN_CHARS

```mettle
export const CR_CODE_MIN_CHARS: int64 = 8;
```

Fewest random characters a code may have (8 characters is 40 bits; 10 or more is recommended).

- **returns:** 8
- **owns:** nothing
- **thread:** constant

```mettle
cr_code_new(&c[0], 16, &n, CR_CODE_MIN_CHARS, 0);
```

### CR_CODE_MAX_CHARS

```mettle
export const CR_CODE_MAX_CHARS: int64 = 32;
```

Most random characters a code may have.

- **returns:** 32
- **owns:** nothing
- **thread:** constant

```mettle
var c: uint8[40]; cr_code_new(&c[0], 40, &n, CR_CODE_MAX_CHARS, 0);
```

### cr_token_new

```mettle
export fn cr_token_new(out: uint8*, out_cap: int64, out_len: int64*, prefix: string, alphabet: int32, nchars: int64) -> int32
```

Generates a random token: prefix, then nchars characters each picked uniformly from the alphabet (no modulo bias), then a nul. A bearer token needs at least 128 bits of entropy (CR_TOKEN_MIN_BITS): base62 >= 22 characters, base32 >= 26 (see cr_token_bits).

- **in:** out writable for out_cap >= prefix.length + nchars + 1 bytes; out_len; prefix, at most 32 bytes of [A-Za-z0-9_-] (may be empty); alphabet, one of CR_TOKEN_BASE62 / CR_TOKEN_BASE32; nchars <= 256 with cr_token_bits(alphabet, nchars) >= 128
- **returns:** CR_OK and *out_len = prefix.length + nchars (the nul is not counted); CR_E_ARG (null out or out_len, bad prefix, unknown alphabet, nchars too small for 128 bits or above 256); CR_E_SPACE if out_cap is too small (nothing written); CR_E_INIT. If the random source fails, out (out_cap bytes) is wiped and its code is returned. On error *out_len = 0
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var t: uint8[64]; var n: int64; cr_token_new(&t[0], 64, &n, "mtlr_", CR_TOKEN_BASE62, 40);  // registry API token
```

### cr_token_check

```mettle
export fn cr_token_check(tok: string, prefix: string, alphabet: int32, nchars: int64) -> bool
```

Checks the format of a token: its length is prefix.length + nchars, it starts with prefix, and every following byte is in the alphabet. Use it to reject malformed input before any store lookup; it is not a secret comparison.

- **in:** tok, prefix; alphabet, one of the CR_TOKEN_* selectors; nchars <= 256 with cr_token_bits(alphabet, nchars) >= 128
- **returns:** true if the token has the format; false otherwise, including for an unknown alphabet or an nchars that cr_token_new would refuse
- **owns:** nothing
- **thread:** any thread (pure)

```mettle
if (!cr_token_check(header, "mtlr_", CR_TOKEN_BASE62, 40)) { reject(); }
```

### cr_token_bits

```mettle
export fn cr_token_bits(alphabet: int32, nchars: int64) -> int64
```

Entropy of a random part in whole bits, rounded down: base62 nchars * 5954 / 1000 (log2 62 = 5.954), base32 5 * nchars. Any size from 1 to 256 characters can be asked, so callers can compute what a length gives before choosing it.

- **in:** alphabet, one of the CR_TOKEN_* selectors; nchars in 1..256
- **returns:** the bits, or -1 for an unknown alphabet or nchars outside 1..256
- **owns:** nothing
- **thread:** any thread (pure)

```mettle
cr_token_bits(CR_TOKEN_BASE62, 22) == 130
```

### cr_token_hash

```mettle
export fn cr_token_hash(out: CrSha256*, tok: string) -> int32
```

SHA-256 of the whole token text (prefix included), for storing tokens at rest. ONLY for high-entropy tokens (>= 128 bits, see cr_token_bits): a leaked SHA-256 of such a token cannot be brute-forced. Low-entropy codes, such as 10-character recovery codes (~50 bits), must NOT be stored this way: store them with cr_code_hash instead, because a leaked SHA-256 of a ~50-bit code is brute-forceable.

- **in:** out; tok, the token text
- **returns:** CR_OK; CR_E_ARG for a null out; otherwise cr_sha256's result (CR_E_INIT). On error out is not written
- **owns:** nothing; the caller owns out
- **thread:** main thread only

```mettle
var d: CrSha256; cr_token_hash(&d, tok);  // store d, not tok
```

### cr_code_new

```mettle
export fn cr_code_new(out: uint8*, out_cap: int64, out_len: int64*, nchars: int64, group: int64) -> int32
```

Generates a random human code (a recovery code): nchars characters from the Crockford base32 lowercase alphabet "0123456789abcdefghjkmnpqrstvwxyz" (uniform, no modulo bias), with a '-' between every `group` characters when group > 0, then a nul. For example nchars 10 and group 5 give "xxxxx-yyyyy". 10 or more characters are recommended (50 bits). The code is a secret until used: show it once, store only cr_code_hash of it.

- **in:** out writable for out_cap >= total + 1 bytes, where total = nchars + (nchars - 1) / group dashes (no dashes when group == 0); out_len; nchars in 8..32; group 0 (no dashes) or in 2..16
- **returns:** CR_OK and *out_len = total (the nul is not counted); CR_E_ARG (null out or out_len, nchars outside 8..32, group < 0, group == 1, group > 16); CR_E_SPACE if out_cap < total + 1 (nothing written); CR_E_INIT. If the random source fails, out (out_cap bytes) is wiped and its code is returned. On error *out_len = 0
- **owns:** nothing; the caller owns out and must cr_wipe it when done
- **thread:** main thread only

```mettle
var c: uint8[16]; var n: int64; cr_code_new(&c[0], 16, &n, 10, 5);  // "k3f9a-7mx2q"
```

### cr_code_normalize

```mettle
export fn cr_code_normalize(out: uint8*, out_cap: int64, out_len: int64*, input: string) -> int32
```

Turns what a person typed into the canonical code: skips '-' and ' ', turns ASCII A-Z into lowercase, then reads i and l as 1 and o as 0 (the Crockford rules); every other byte, u included, must be in the alphabet "0123456789abcdefghjkmnpqrstvwxyz". The result is followed by a nul. It makes no libsodium call except cr_wipe.

- **in:** out writable for out_cap >= normalized length + 1 bytes; out_len; input, the typed text
- **returns:** CR_OK and *out_len = the normalized length (8..32, the nul is not counted); CR_E_ARG (null out or out_len); CR_E_DECODE for a byte outside the rules or a normalized length outside 8..32; CR_E_SPACE if the normalized length + 1 > out_cap. On CR_E_DECODE and CR_E_SPACE out (out_cap bytes) is wiped; on every error *out_len = 0
- **owns:** nothing; the caller owns out (a code is a secret, wipe it when done)
- **thread:** main thread only (cr_wipe)

```mettle
var c: uint8[40]; var n: int64; cr_code_normalize(&c[0], 40, &n, "ABCDE-I2345");  // "abcde12345"
```

### cr_code_hash

```mettle
export fn cr_code_hash(out: CrHmacSha256*, pepper: CrKey32*, code: string) -> int32
```

Hashes a human code for storage: out = HMAC-SHA256(pepper, "cryptex/code/v1" || normalized code), where the code is first normalized with cr_code_normalize (case, dashes, spaces and look-alike letters do not matter) and the 15-byte context string has no nul. This, not a plain SHA-256, is how recovery codes are stored: a code has only ~50 bits (10 characters), so SHA-256 of it is brute-forceable from a leaked database. The pepper is a 32-byte server secret kept outside the database (config or environment) and never stored next to the tags. Store the tag; look a submitted code up by its tag (an equality lookup is fine, because the tag is useless without the pepper); delete the row once the code is used. 10 or more characters are recommended (50 bits).

- **in:** out; pepper, the 32-byte server secret; code, the code as typed
- **returns:** CR_OK; CR_E_ARG (null out or pepper); CR_E_DECODE (cr_code_normalize refuses the code); CR_E_INIT / CR_E_INTERNAL from the HMAC. On error out is wiped
- **owns:** nothing; the caller owns out and pepper. The local buffer holding the normalized code and the HMAC state are wiped
- **thread:** main thread only

```mettle
var tag: CrHmacSha256; cr_code_hash(&tag, &pepper, typed);  // WHERE tag = ?
```

## seal

Sealed messages: XChaCha20-Poly1305 with the nonce carried inside the output, so callers never handle nonces.

### CR_SEAL_OVERHEAD

```mettle
export const CR_SEAL_OVERHEAD: int64 = 40;
```

Bytes a sealed message adds to the plaintext: 24-byte nonce + 16-byte tag.

- **returns:** 40
- **owns:** nothing
- **thread:** constant

```mettle
var cap: int64 = mlen + CR_SEAL_OVERHEAD;
```

### CR_SEAL_VERSION

```mettle
export const CR_SEAL_VERSION: int32 = 1;
```

The format version cr_seal writes into the first byte of the nonce, and the only one cr_open accepts.

- **returns:** 1
- **owns:** nothing
- **thread:** constant

```mettle
if (box[0] != (uint8)CR_SEAL_VERSION) { not_a_sealed_message(); }
```

### cr_seal

```mettle
export fn cr_seal(out: uint8*, out_cap: int64, out_len: int64*, msg: uint8*, mlen: int64, ad: uint8*, adlen: int64, key: CrKey32*) -> int32
```

Encrypts and authenticates msg under key with a fresh nonce (version byte + 23 random bytes), and authenticates (does not encrypt) ad. Output is nonce || ciphertext || tag.

- **in:** out writable for out_cap bytes, out_cap >= mlen + CR_SEAL_OVERHEAD (out must not overlap msg); out_len, where the sealed length goes; msg/mlen the plaintext; ad/adlen associated data, authenticated but not encrypted (0, 0 for none); key
- **returns:** CR_OK and *out_len = mlen + 40; CR_E_ARG (null out/key/out_len, negative length or capacity, null msg with mlen > 0, null ad with adlen > 0); CR_E_SPACE if out_cap is too small (nothing written, *out_len = 0); CR_E_INIT; CR_E_INTERNAL. If a failure happens after writing started, *out_len = 0 and the first out_cap bytes of out are wiped
- **owns:** nothing; the caller owns every buffer and must wipe key and plaintext when done
- **thread:** main thread only

```mettle
cr_seal(&box[0], 128, &blen, m.chars, (int64)m.length, 0, 0, &k);
```

### cr_open

```mettle
export fn cr_open(out: uint8*, out_cap: int64, out_len: int64*, sealed: uint8*, slen: int64, ad: uint8*, adlen: int64, key: CrKey32*) -> int32
```

Verifies and decrypts a message produced by cr_seal with the same key and ad. Fails closed: on every failure *out_len = 0 and out's first out_cap bytes are wiped.

- **in:** out writable for out_cap bytes, out_cap >= slen - CR_SEAL_OVERHEAD (out must not overlap sealed); out_len; sealed/slen the sealed message (nonce || ciphertext || tag); ad/adlen the same associated data given to cr_seal (0, 0 for none); key
- **returns:** CR_OK and *out_len = slen - 40; CR_E_VERIFY if authentication fails (wrong key or ad, tampered nonce, ciphertext or tag, slen < 40, a version other than CR_SEAL_VERSION); CR_E_SPACE if out_cap < slen - 40; CR_E_ARG (null out/key/sealed/ out_len, negative length or capacity, null ad with adlen > 0); CR_E_INIT
- **owns:** nothing; the caller owns every buffer and must wipe key and the plaintext when done
- **thread:** main thread only

```mettle
if (cr_open(&pt[0], 128, &plen, &box[0], blen, 0, 0, &k) != CR_OK) { reject(); }
```
