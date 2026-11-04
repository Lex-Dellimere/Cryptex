# References

Cryptex follows the published standard for every primitive and takes its
techniques from proven implementations. Nothing here is a new algorithm.

## Standards

| Area | Standard |
|---|---|
| SHA-256, SHA-384, SHA-512, SHA-1 | FIPS 180-4 |
| BLAKE2b | RFC 7693 |
| HMAC | RFC 2104 (test cases: RFC 2202, RFC 4231) |
| HKDF | RFC 5869 |
| Argon2id | RFC 9106 |
| ChaCha20, Poly1305, ChaCha20-Poly1305 | RFC 8439 |
| XChaCha20-Poly1305 | draft-irtf-cfrg-xchacha-03 |
| AES | FIPS 197 |
| GCM | NIST SP 800-38D |
| X25519 | RFC 7748 |
| Ed25519 | RFC 8032 |
| P-256, P-384, ECDH, ECDSA | FIPS 186-5, SEC 1 v2, RFC 6979 (deterministic nonces) |
| RSA signatures (PKCS#1 v1.5, PSS) | RFC 8017 |
| HOTP, TOTP | RFC 4226, RFC 6238 |
| Hex, base64url, base32 | RFC 4648 |

## Where the techniques come from

| Part of Cryptex | Source | License | Why this one |
|---|---|---|---|
| AES (`src/cipher/aes.mettle`) | BearSSL `aes_ct64` | MIT | Bitsliced, constant time, no lookup tables, works on any 64-bit CPU. Table-based AES leaks the key through cache timing. |
| GHASH (`src/cipher/ghash.mettle`) | BearSSL `ghash_ctmul64` | MIT | Carry-less multiply built from masked integer multiplies: constant time without PCLMULQDQ. |
| P-256 (`src/ecc/p256.mettle`) | BearSSL `ec_p256_m31` | MIT | Curve-specific 30-bit limbs, constant-time windows and complete formulas; faster than BearSSL's generic code. |
| P-384 (`src/ecc/p384.mettle`) | BearSSL `ec_prime_i31` | MIT | Generic constant-time prime-curve code on i31 integers; 2-bit window with constant-time selection. |
| Big integers (`src/ecc/i31.mettle`) | BearSSL `i31` | MIT | Constant-time Montgomery arithmetic in 31-bit words, shared by ECDSA, P-384 and RSA. |
| ECDSA, RFC 6979 nonces | BearSSL `ecdsa_i31_*`, `hmac_drbg` | MIT | Deterministic nonces remove the need for a good random source when signing. Strict DER is checked by re-encoding, as BoringSSL does. |
| RSA verify | BearSSL `rsa_i31_pub`, `rsa_pkcs1_sig_unpad`, `rsa_pss_sig_unpad`, `mgf1` | MIT | PKCS#1 v1.5 is checked by encoding the expected block and comparing it whole (RFC 8017 8.2.2), which avoids parser bugs. |
| RSA sign | BearSSL `rsa_i31_priv`, `rsa_pss_sig_pad`, `rsa_pkcs1_sig_pad` | MIT | Constant-time CRT. Every signature is checked with the public key before release, which stops CRT fault attacks (Boneh-DeMillo-Lipton). |
| Curve25519, Ed25519 (`src/ecc/curve25519.mettle`) | TweetNaCl | public domain | Short, audited design with 16 limbs of 64 bits, masked swaps, no 128-bit math needed. Monocypher and ref10 were studied for a faster layout later. |
| Poly1305 | poly1305-donna (32-bit) | public domain / MIT | Five 26-bit limbs with 64-bit products; the standard constant-time layout. |
| ChaCha20, HChaCha20, BLAKE2b, SHA-2, SHA-1, Argon2id | the RFC and FIPS reference descriptions | — | Straightforward ARX and table-free designs; written from the specifications. |
| Constant-time compare and wipe | libsodium `sodium_memcmp`, `sodium_memzero` (ideas only) | ISC | OR-accumulate over every byte; volatile stores the optimizer cannot remove. |

Ported files say so in their header. BearSSL's license is in `NOTICE`.

## How it is checked

- **Official vectors:** FIPS and RFC examples, NIST CAVP (SHA, HMAC, GCM, ECDSA SigVer and KeyPair, PKV, ECC CDH, RSA SigVer15 and SigVerPSS).
- **Wycheproof** (C2SP), including invalid and edge cases, for X25519, Ed25519, ChaCha20-Poly1305, XChaCha20-Poly1305, AES-GCM, HMAC, HKDF, P-256 and P-384 ECDH and ECDSA, and RSA PKCS#1 v1.5 and PSS.
- **Reference tools:** libsodium and Python's `cryptography` (OpenSSL) generate or cross-check every vector file; the library itself does not need them.
- **Constant time:** the compiled release code of the secret paths was read for branches on secret data.
