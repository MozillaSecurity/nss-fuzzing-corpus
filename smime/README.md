# S/MIME fuzzing corpus additions (bug 1961826)

Prepared 2026-10-09 for a PR to `MozillaSecurity/nss-fuzzing-corpus` (`smime/`).
Not committed anywhere; nothing here lives in the NSS tree.

## Layout

| Path | Contents |
|---|---|
| `smime/` | **The PR payload.** 1,748 files named by SHA-1 (the repo's convention), each of which adds coverage over the upstream `smime/` corpus at `5f0fc81` (2026-10-01, 4,461 files). Drop them into the repo's `smime/` directory as-is. |
| `seeds-named/` | The 164 hand-generated seeds with descriptive names, for reference. Every one of these was fed to the merge; the ones that survived are in `smime/` under their hash. |

## How the inputs were produced

Target: `nssfuzz-smime` from the rebased bug 1961826 stack (branch
`bug-1961826-extend-smime-fuzzer-rebased`, tip `ec139e22c1`), ASan+UBSan fuzz
build. The rebased fuzzer imports an RSA-2048 cert ("SMIME Fuzz Test",
`fuzz@test.invalid`) and a P-256 cert ("SMIME Fuzz EC", `fuzz-ec@test.invalid`)
into a temp DB and supplies a decrypt-key callback, so messages addressed to
those certs exercise actual decryption and signature verification rather than
just parsing.

1. **`cmsutil` seeds (48)** — NSS's own encoder against the two fuzzer certs:
   SignedData with SHA-1/256/384/512, with signing time + SMIMECapabilities,
   with EncryptionKeyPreference, detached; EnvelopedData to RSA, to ECDH, to
   both, by e-mail address; EncryptedData (`-C`); certs-only (`-O`); nested
   signed-in-enveloped and enveloped-in-signed; content sizes empty, small,
   5 KB, 70 KB, and exactly 4096/4097 bytes (the `cmscipher.c` block size).
2. **OpenSSL `cms` seeds (65)** — structures NSS cannot emit or that need
   other key types: RSA-OAEP (SHA-1/SHA-256), ECDH with every KDF hash
   (SHA-1/224/256/384/512) and cofactor mode, AES-128/192/256 key wrap,
   PasswordRecipientInfo, KEKRecipientInfo, AuthEnvelopedData (AES-GCM),
   indefinite-length BER (`-stream`), DigestedData, Data, EncryptedData
   (AES/3DES/GCM), SignedData with no attributes / no SMIMECapabilities / no
   certs / RSA-PSS / ECDSA / two signers / receipt request / extra certs,
   recipients on P-224/P-384/P-521/brainpoolP256r1/X25519, RSA-1024/4096,
   Camellia, MD5/SHA-224 digests, triple nesting, a signed receipt.
3. **NSS smime test-suite artifacts (45)** — the `alice*`/`fran-*` files the
   `tests/smime` suite generates: OAEP with labels/MGF variants, ECDH with
   several KDFs, AuthEnvelopedData including a corrupted-MAC and an empty
   variant, PKCS#7 envelopes.
4. **Four fuzzer seeds** from the original D297253 and **two hand-made
   invalid-point ECDH envelopes** (originator EC point zeroed / byte-flipped)
   that drive the PKCS#11 derive-retry path fixed in bug 2029752.
5. **Fuzz campaign** — 24 workers × 25 min from the earlier minimized corpus
   plus all of the above (`-max_len=65536 -len_control=100
   -use_value_profile=1`): 5.52 M executions, coverage 6641 → 7907, 23,770
   corpus files, **no crashes, leaks or UBSan reports**.
6. **Merge** — `nssfuzz-smime -merge=1` into a copy of the upstream `smime/`
   corpus; only inputs adding coverage over upstream were kept.

## Validation

- Upstream corpus (4,469 files incl. `regressions/`) through the rebased
  fuzzer: clean, coverage 4782.
- Final `smime/` additions through the rebased fuzzer: 1,775 runs, clean; the
  additions alone reach coverage 7826. libFuzzer merge statistics: 1,769
  inputs kept, 22,839 new features, **2,328 new coverage edges over upstream**
  (upstream alone: 4,782 edges).
- 51 of the 164 named seeds survived the merge as distinct files; the rest were
  subsumed by campaign-generated inputs that cover the same edges.

## Notes for the PR

- The two invalid-point ECDH inputs are regression seeds for bug 2029752;
  upstream has a `smime/regressions/` directory that may be the better home
  for those two (hashes `435832ef8d40a0821b5c66bb343da474752d717a` zeroed point,
  `67058f2812f55c7220dbf283bd9aeea578b196c0` flipped byte).
- The upstream corpus has files up to 4 MB. Everything here is ≤ 64 KB.
