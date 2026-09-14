# LFTP system OpenSSL pre-PR guide

Guide reviewed: 2026-09-14. Existing validation was collected on 2026-09-12.

## Proposed PR

- Suggested title: `fix(lftp): use system OpenSSL for TLS`
- Branch: `lyrydber/azl4/pqc/lftp`
- Target: current upstream `4.0`
- Experimental base: `905c3db85fadff13914dc75763ad715666f973ba`
- Goal: route LFTP TLS through Azure Linux system OpenSSL instead of the
  independent GnuTLS/Nettle path.

The change also separates two unrelated uses of `--with-openssl`: LFTP uses
the option to select its TLS backend, while bundled gnulib used the same option
for optional MD5 and SHA-1 acceleration. The new gnulib option is
`--with-openssl-hashes`.

## Evidence already collected

The x86_64 experiment produced `lftp-4.9.3-9.azl4.x86_64.rpm`.

- The final committed component rendered and built successfully with `azldev`.
- The RPM requires `libssl.so.3` and `libcrypto.so.3`.
- `liblftp-network.so` is the TLS-bearing payload and directly needs those two
  libraries.
- No shipped LFTP ELF needs GnuTLS, Nettle, or Hogweed.
- `lftp --version` and a basic command invocation succeeded in an Azure Linux
  4 mock root.
- A direct HTTPS download completed against a local TLS server. TLS 1.3 was
  demonstrated, but the key-establishment and signature algorithms were not
  captured.

This evidence proves system OpenSSL linkage. It does not prove SCOSSL or
SymCrypt selection, FIPS behavior, PQC negotiation, or complete FTPS
compatibility.

## Mandatory pre-PR gates

### 1. Refresh and build

Run on the approved Azure Linux development build host:

```sh
azldev comp list -p lftp -q -O json
azldev comp update -p lftp
azldev comp render -p lftp
azldev comp build -p lftp --preserve-buildenv on-failure
```

- Rebase or refresh the branch against the current upstream `4.0` first.
- Review the generated spec, lock, release, and patch copies.
- Repeat the complete build on x86_64 and native aarch64.
- Retain SRPM, binary RPM, build log, root log, installed-package list, and
  source checksums for both architectures.
- Confirm a second render produces no diff.

### 2. Artifact and linkage checks

- Inspect every shipped ELF, not only `/usr/bin/lftp`.
- Confirm `liblftp-network.so` needs `libssl.so.3` and `libcrypto.so.3`.
- Confirm no shipped ELF needs `libgnutls`, `libnettle`, or `libhogweed`.
- Confirm RPM requirements match the ELF evidence.
- Confirm the OpenSSL backend was compiled and the GnuTLS backend was not.
- Verify the gnulib MD5 and SHA-1 helpers remain functional without duplicate
  symbols or unintended libcrypto hash acceleration.

### 3. Protocol test matrix

Test with normal Azure Linux policy and record the negotiated TLS version,
cipher suite, key-establishment group, certificate signature algorithm, and
peer certificate chain.

| Area | Required coverage |
|---|---|
| HTTPS | GET, upload if supported, redirects, SNI, IPv4/IPv6, connection reuse |
| Certificate validation | Trusted chain succeeds; unknown CA, expired cert, and hostname mismatch fail |
| Client certificates | Required and optional client-authentication servers |
| Explicit FTPS | `AUTH TLS`, login, upload, download, listing, passive and active data channels |
| Implicit FTPS | Login, upload, download, listing, passive and active data channels |
| FTPS session behavior | Data-channel protection, TLS session reuse, reconnect, and resume |
| Configuration | Default settings plus supported OpenSSL-form `ssl:priority` overrides |
| Failure handling | TLS alert, abrupt close, timeout, retry limit, and unavailable provider |

Document configuration compatibility. Existing GnuTLS priority strings are
not valid OpenSSL cipher configuration and may require release notes.

### 4. System provider, FIPS, and PQC checks

- Prove which OpenSSL providers LFTP actually uses; dynamic linkage alone is
  insufficient.
- Demonstrate the expected SCOSSL/SymCrypt path for applicable algorithms.
- Repeat representative HTTPS and FTPS tests in normal and FIPS modes.
- Confirm system policy can reject disallowed protocol versions, groups,
  signatures, and ciphers even when LFTP configuration requests them.
- Record whether supported hybrid/PQC groups or signatures are available and
  negotiated. Record classical fallback explicitly when they are not.

### 5. Regression checks

- Compare command behavior and exit status with the published GnuTLS package.
- Exercise proxy, DNS, IPv6, scripting, mirror, and retry behavior touched by
  network setup.
- Run available upstream tests and package checks.
- Verify upgrades preserve configuration and produce a clear outcome for any
  GnuTLS-form `ssl:priority` value.

## Acceptance criteria

Do not open the PR until:

- x86_64 and aarch64 builds pass from the refreshed branch;
- HTTPS, explicit FTPS, and implicit FTPS pass;
- negative certificate tests fail safely;
- provider and FIPS evidence is captured;
- no GnuTLS/Nettle/Hogweed linkage remains;
- configuration compatibility and rollback are documented; and
- the `--with-openssl-hashes` patch has an upstream disposition.

## PR description material

```text
## Why

LFTP currently selects GnuTLS and bypasses the Azure Linux system OpenSSL
provider path. Moving TLS to system OpenSSL allows LFTP to inherit centralized
crypto policy, FIPS behavior, and future PQC support from the platform stack.

## What changed

- replace gnutls-devel with openssl-devel
- select LFTP's OpenSSL TLS backend
- rename gnulib's unrelated hash-acceleration option to
  --with-openssl-hashes
- keep gnulib MD5/SHA-1 acceleration disabled

## Risk

GnuTLS and OpenSSL configuration syntax and some FTPS/certificate behavior
differ. The most important compatibility areas are ssl:priority, explicit and
implicit FTPS, data-channel TLS, session reuse, and client certificates.

## Validation

Include architecture builds, exact ELF/RPM linkage, protocol matrix results,
negative certificate tests, provider/FIPS evidence, negotiated algorithms,
and any configuration migration notes.

## Rollback

Restore the GnuTLS dependency and the original
--with-gnutls --without-openssl build selection.
```
