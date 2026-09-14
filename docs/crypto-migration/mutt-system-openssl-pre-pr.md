# Mutt system OpenSSL pre-PR guide

Guide reviewed: 2026-09-14. Existing validation was collected on 2026-09-12.

## Proposed PR

- Suggested title: `fix(mutt): use system OpenSSL for TLS`
- Branch: `lyrydber/azl4/pqc/mutt`
- Target: current upstream `4.0`
- Experimental base: `905c3db85fadff13914dc75763ad715666f973ba`
- Goal: make system OpenSSL the default Mutt TLS backend while preserving
  Azure Linux cipher and protocol policy.

The GnuTLS bcond remains available for controlled comparison builds. The
OpenSSL backend defaults `ssl_ciphers` to `PROFILE=SYSTEM` and no longer resets
OpenSSL's configured minimum and maximum protocol versions.

## Evidence already collected

The x86_64 experiment produced `mutt-5:2.3.0-5.azl4.x86_64.rpm`.

- The final committed component rendered and built successfully with `azldev`.
- `mutt -v` reports `+USE_SSL_OPENSSL` and `-USE_SSL_GNUTLS`.
- The Mutt ELF needs `libssl.so.3` and `libcrypto.so.3`.
- It does not need GnuTLS, Nettle, or Hogweed.
- `ssl_ciphers` defaults to `PROFILE=SYSTEM`.
- TLS 1.0 and 1.1 are disabled by default; TLS 1.2 and 1.3 are enabled.
- A local IMAPS test completed TLS 1.3 negotiation, authentication, and INBOX
  selection with `TLS_AES_256_GCM_SHA384`.

This evidence proves the OpenSSL backend and system cipher profile are active.
It does not prove SCOSSL/SymCrypt selection, FIPS behavior, PQC negotiation, or
the SMTP and POP paths.

## Mandatory pre-PR gates

### 1. Refresh and build

```sh
azldev comp list -p mutt -q -O json
azldev comp update -p mutt
azldev comp render -p mutt
azldev comp build -p mutt --preserve-buildenv on-failure
```

- Rebase or refresh against the current upstream `4.0`.
- Build the default OpenSSL variant on x86_64 and native aarch64.
- Also build the retained GnuTLS bcond variant to prove reversibility and
  prevent bit-rot in the comparison path.
- Retain SRPMs, RPMs, logs, package lists, and source checksums.
- Confirm a second render is clean.

### 2. Artifact and configuration checks

- Confirm `mutt -v` reports OpenSSL enabled and GnuTLS disabled in the default
  package.
- Confirm the Mutt ELF needs only `libssl.so.3` and `libcrypto.so.3` for TLS.
- Confirm `ssl_ciphers="PROFILE=SYSTEM"` with a clean configuration.
- Confirm TLS 1.0/1.1 remain off and TLS 1.2/1.3 remain on by default.
- Confirm explicit user settings cannot weaken system protocol bounds.
- Confirm the optional GnuTLS build still uses the expected source and
  dependency set.

### 3. Mail protocol test matrix

Capture TLS version, cipher suite, key-establishment group, certificate
signature algorithm, and peer chain for every successful session.

| Protocol | Required coverage |
|---|---|
| IMAP | IMAPS and STARTTLS; login, mailbox select, fetch, append, reconnect |
| POP | POP3S and STLS; login, list, retrieve, delete-disabled test |
| SMTP | SMTPS and STARTTLS; authentication and message submission to a test sink |
| Authentication | Password, SASL mechanisms used by Azure Linux, and Kerberos where applicable |
| Client certificates | Optional and required client certificate servers |
| Certificate validation | Trusted chain succeeds; unknown CA, expiry, and hostname mismatch fail |
| Partial chains | Test `ssl_verify_partial_chains` enabled and disabled |
| Configuration | System defaults and valid OpenSSL `ssl_ciphers` overrides |
| Session behavior | Reconnect, timeout, interrupted handshake, and connection reuse |

Run the tests with isolated Mutt configuration and mailboxes. Do not use a
production account.

### 4. System provider, FIPS, and PQC checks

- Prove the provider used by Mutt operations; library linkage is not enough.
- Demonstrate the expected SCOSSL/SymCrypt path for applicable algorithms.
- Repeat representative IMAP, POP, and SMTP tests in normal and FIPS modes.
- Verify `PROFILE=SYSTEM` and OpenSSL configuration reject disallowed
  protocols, groups, signatures, and ciphers.
- Record supported and negotiated hybrid/PQC algorithms, or the exact
  classical fallback.

### 5. Compatibility and regression checks

- Compare the OpenSSL and GnuTLS variants against the same test servers.
- Test `certificate_file`, system certificates, client certificates, SNI,
  hostname validation, partial chains, and custom cipher configuration.
- Verify existing Mutt configuration does not contain GnuTLS-form
  `ssl_ciphers` values, or document the required migration.
- Run available upstream and package tests.
- Exercise non-TLS startup, local mailboxes, IMAP cache, and normal exit paths.

## Acceptance criteria

Do not open the PR until:

- default OpenSSL builds pass on x86_64 and aarch64;
- the retained GnuTLS comparison build still compiles;
- IMAP, POP, and SMTP implicit and STARTTLS coverage passes;
- negative certificate tests fail safely;
- provider/FIPS evidence is captured;
- configuration migration risk is documented; and
- no GnuTLS/Nettle/Hogweed linkage remains in the default package.

## PR description material

```text
## Why

The published Mutt package uses GnuTLS with application-defined priority
handling. Selecting system OpenSSL allows Mutt to use Azure Linux centralized
crypto policy, FIPS behavior, and future platform PQC support.

## What changed

- default the TLS bcond to OpenSSL
- use openssl-devel and --with-ssl
- retain only the selected backend source
- default OpenSSL ssl_ciphers to PROFILE=SYSTEM
- preserve OpenSSL's system protocol minimum and maximum
- retain the GnuTLS bcond for comparison builds

## Risk

The backends use different cipher configuration syntax and can differ in
certificate, partial-chain, and protocol behavior. Existing GnuTLS-form
ssl_ciphers values require explicit migration guidance.

## Validation

Include both architecture builds, the GnuTLS comparison build, exact linkage,
IMAP/POP/SMTP matrices, negative certificate cases, provider/FIPS evidence,
and negotiated algorithms.

## Rollback

Re-enable the GnuTLS bcond by default and restore the original backend source
selection.
```
