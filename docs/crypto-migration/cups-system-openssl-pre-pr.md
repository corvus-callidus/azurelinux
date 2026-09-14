# CUPS system OpenSSL pre-PR guide

Guide reviewed: 2026-09-14. Existing validation was collected on 2026-09-12.

## Proposed PR

- Suggested title: `fix(cups): use system OpenSSL for TLS`
- Branch: `lyrydber/azl4/pqc/cups`
- Target: current upstream `4.0`
- Experimental base: `905c3db85fadff13914dc75763ad715666f973ba`
- Goal: route CUPS client and server TLS through system OpenSSL while keeping
  Azure Linux cipher and protocol policy authoritative.

The OpenSSL backend defaults to `PROFILE=SYSTEM`. CUPS protocol bounds are
intersected with OpenSSL's configured system bounds. The existing `NoSystem`
option remains the explicit escape hatch for application-defined policy.

## Evidence already collected

The x86_64 experiment produced
`cups-libs-1:2.4.16-10.azl4.x86_64.rpm` and the matching CUPS package set.

- The final committed component rendered and built successfully with `azldev`.
- `libcups.so.2` needs `libssl.so.3` and `libcrypto.so.3`.
- It does not need GnuTLS, Nettle, or Hogweed.
- `cups-devel` requires OpenSSL development metadata instead of GnuTLS.
- Rebuilt `ippeveprinter` and `ipptool` completed an IPPS
  Get-Printer-Attributes exchange.
- The observed session used TLS 1.3 with `TLS_AES_256_GCM_SHA384`.

This proves a CUPS client/server IPPS path uses system OpenSSL. It does not
prove cupsd production configuration, printer compatibility, FIPS provider
selection, or PQC negotiation.

## Mandatory pre-PR gates

### 1. Refresh and build

```sh
azldev comp list -p cups -q -O json
azldev comp update -p cups
azldev comp render -p cups
azldev comp build -p cups --preserve-buildenv on-failure
```

- Rebase or refresh against the current upstream `4.0`.
- Build the full package set on x86_64 and native aarch64.
- Perform the repository's applicable multilib/co-installability validation;
  the `cups-config` libdir handling is architecture-sensitive.
- Retain SRPMs, RPMs, logs, package lists, and source checksums.
- Confirm a second render is clean.

### 2. Package, ABI, and linkage checks

- Inspect all shipped CUPS ELFs, not only `libcups.so.2`.
- Confirm TLS-bearing payloads need `libssl.so.3` and `libcrypto.so.3`.
- Confirm no payload needs GnuTLS, Nettle, or Hogweed.
- Confirm `cups-devel`, `cups.pc`, `cups-config`, static metadata, and multilib
  paths reference OpenSSL correctly.
- Run ABI and dependency comparisons against the published package.
- Confirm existing CUPS consumers compile and link against `cups-devel`.

### 3. IPP and certificate test matrix

Capture TLS version, cipher suite, key-establishment group, certificate
signature algorithm, and peer chain.

| Area | Required coverage |
|---|---|
| IPP client | `ipptool`, destination discovery, Get-Printer-Attributes, print job |
| IPP server | `ippeveprinter` and cupsd listeners over IPPS |
| cupsd | `SSLListen`/TLS listener, scheduler startup, reload, and shutdown |
| Web interface | HTTPS access to enabled administrative and printer pages |
| Credentials | Auto-created credentials and administrator-provided certificate/key chains |
| Validation | Trusted chain succeeds; unknown CA, expiry, and hostname mismatch fail |
| Client auth | Optional and required client certificates where supported |
| Network | IPv4, IPv6, DNS-SD discovery, reconnect, timeout, and concurrent clients |
| Printer compatibility | Current IPP Everywhere devices plus oldest supported legacy TLS devices |

### 4. CUPS policy option matrix

Test each supported `SSLOptions` combination in both client and server paths:

- default options;
- `NoSystem`;
- `DenyCBC`;
- `AllowDH`;
- `AllowRC4`;
- minimum TLS 1.0, 1.1, 1.2, and 1.3 requests;
- maximum TLS 1.0, 1.1, 1.2, and 1.3 requests; and
- invalid or contradictory combinations.

For the default path, CUPS must not weaken the system cipher or protocol
policy. For `NoSystem`, document that CUPS intentionally restores its
application-defined policy. Confirm TLS 1.3 ciphersuites still obey OpenSSL
system configuration because `SSL_CTX_set_cipher_list` covers pre-TLS 1.3
ciphers only.

### 5. System provider, FIPS, and PQC checks

- Prove the provider used by CUPS; `DT_NEEDED` evidence is insufficient.
- Demonstrate SCOSSL/SymCrypt routing for applicable algorithms.
- Repeat client, `ippeveprinter`, and cupsd tests in normal and FIPS modes.
- Verify system policy rejects disallowed algorithms even when CUPS options
  request them.
- Record hybrid/PQC negotiation support or exact classical fallback.

## Acceptance criteria

Do not open the PR until:

- x86_64, aarch64, and applicable multilib validation pass;
- rebuilt clients interoperate with rebuilt cupsd and `ippeveprinter`;
- a real-printer compatibility sample passes;
- certificate failure cases fail safely;
- every `SSLOptions` path has a documented result;
- provider/FIPS evidence is captured; and
- no GnuTLS/Nettle/Hogweed linkage remains.

## PR description material

```text
## Why

CUPS currently uses GnuTLS. Moving CUPS TLS to system OpenSSL allows clients
and servers to inherit Azure Linux centralized crypto policy, FIPS behavior,
and future platform PQC support.

## What changed

- replace GnuTLS build and devel dependencies with OpenSSL
- select --with-tls=openssl
- default OpenSSL cipher selection to PROFILE=SYSTEM
- intersect CUPS protocol bounds with OpenSSL system bounds
- retain NoSystem as an explicit compatibility override
- update cups-config multilib metadata for OpenSSL

## Risk

Legacy printers may depend on protocols or ciphers rejected by system policy.
The NoSystem, RC4, DH, CBC, minimum-version, and maximum-version controls need
explicit compatibility testing.

## Validation

Include architecture and multilib builds, ABI/linkage results, cupsd and
ippeveprinter IPPS tests, real-printer coverage, option matrix results,
negative certificate tests, provider/FIPS evidence, and negotiated algorithms.

## Rollback

Restore pkgconfig(gnutls), gnutls-devel, and --with-tls=gnutls.
```
