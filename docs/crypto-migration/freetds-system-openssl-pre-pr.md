# FreeTDS system OpenSSL pre-PR guide

Guide reviewed: 2026-09-14. Existing validation was collected on 2026-09-12.

## Status: experimental and release-blocked

Do not open an Azure Linux PR for this branch until legal review explicitly
approves the OpenSSL linkage. FreeTDS upstream still describes
`--with-openssl` as license-incompatible in `m4/check_openssl.m4`.

Technical success does not resolve that issue.

## Proposed PR, if the blocker is cleared

- Suggested title: `fix(freetds): use system OpenSSL for TLS`
- Branch: `lyrydber/azl4/pqc/freetds`
- Target: current upstream `4.0`
- Experimental base: `905c3db85fadff13914dc75763ad715666f973ba`
- Goal: route TDS TLS through system OpenSSL and default cipher selection to
  Azure Linux `PROFILE=SYSTEM`.

The branch replaces GnuTLS and libgcrypt build dependencies, selects
`--without-gnutls --with-openssl`, and preserves the existing administrator
`openssl ciphers` override.

## Evidence already collected

The x86_64 experiment produced `freetds-1.4.23-7.azl4.x86_64.rpm` and matching
library/development packages.

- The final committed component rendered and built successfully with `azldev`.
- `tsql -C` reports `OpenSSL: yes` and `GnuTLS: no`.
- `tsql`, `tdspool`, `libtdsodbc`, `libsybdb`, and `libct` need
  `libssl.so.3` and `libcrypto.so.3`.
- Those payloads do not need GnuTLS, Nettle, Hogweed, or libgcrypt.
- The shipped binary contains the `PROFILE=SYSTEM` default.
- OpenSSL on the test system expands `PROFILE=SYSTEM`.

No TDS-over-TLS session has been negotiated with this package. A plain TLS
server is not a valid substitute because TDS performs prelogin framing before
the TLS exchange.

## Mandatory blockers

1. Obtain a written legal determination for linking the shipped FreeTDS
   binaries to the Azure Linux OpenSSL package.
2. Record the reviewer, scope, applicable FreeTDS subpackages, and any required
   notices or packaging changes.
3. Do not open or merge the PR if the determination is absent, ambiguous, or
   limited to a different distribution context.

## Mandatory pre-PR gates after legal approval

### 1. Refresh and build

```sh
azldev comp list -p freetds -q -O json
azldev comp update -p freetds
azldev comp render -p freetds
azldev comp build -p freetds --preserve-buildenv on-failure
```

- Rebase or refresh against current upstream `4.0`.
- Build on x86_64 and native aarch64.
- Retain SRPMs, RPMs, logs, package lists, checksums, and the legal approval.
- Confirm a second render is clean.

### 2. Artifact, API, and linkage checks

- Inspect every executable and shared library in `freetds` and `freetds-libs`.
- Confirm TLS-bearing payloads need `libssl.so.3` and `libcrypto.so.3`.
- Confirm no payload needs GnuTLS, Nettle, Hogweed, or libgcrypt.
- Run `tsql -C` and record the complete compile-time settings.
- Compare exported symbols, SONAMEs, RPM dependencies, ODBC driver metadata,
  and development files with the published package.
- Build and run representative DB-Library, CT-Library, and ODBC consumers.

### 3. Real SQL Server compatibility matrix

Use disposable test databases and non-production credentials. Include:

- Azure SQL or the current cloud equivalent;
- current supported SQL Server releases;
- the oldest customer-relevant SQL Server release;
- all supported TDS protocol versions, including `auto`;
- SQL authentication and Kerberos/integrated authentication;
- `tsql`, unixODBC, DB-Library, and CT-Library;
- login-only encryption and full-session encryption modes supported by the
  selected server/client versions; and
- reconnect, timeout, failover, and interrupted-handshake behavior.

For each connection, capture:

- FreeTDS configuration and interface used;
- server and TDS version;
- encryption requirement;
- TLS version and cipher suite;
- key-establishment group;
- certificate signature algorithm and chain;
- authentication mechanism; and
- pass/fail plus server and client logs.

### 4. Certificate and configuration tests

- System CA trust succeeds with a valid hostname.
- Unknown CA, expired certificate, and hostname mismatch fail.
- Custom CA file and CRL behavior work.
- The default empty `openssl ciphers` setting resolves to `PROFILE=SYSTEM`.
- A valid administrator cipher override works.
- Invalid and policy-disallowed overrides fail safely.
- Legacy servers rejected by system policy produce actionable diagnostics.

### 5. System provider, FIPS, and PQC checks

- Prove which providers service FreeTDS operations.
- Demonstrate SCOSSL/SymCrypt routing for applicable algorithms.
- Repeat representative SQL authentication and Kerberos tests in FIPS mode.
- Confirm system policy rejects disallowed protocols, groups, signatures, and
  ciphers.
- Record hybrid/PQC negotiation when supported by both client and SQL Server;
  otherwise record the exact classical fallback.

## Acceptance criteria

Do not open the PR until:

- legal approval is recorded;
- x86_64 and aarch64 builds pass;
- the SQL Server/TDS/interface matrix passes at the agreed compatibility bar;
- negative certificate tests fail safely;
- provider/FIPS evidence is captured;
- ABI and consumer compatibility are acceptable; and
- any lost legacy compatibility has an approved release-note and support plan.

## PR description material

```text
## Legal review

Link to the written approval and summarize its scope. This section is
mandatory.

## Why

The published FreeTDS package uses GnuTLS plus direct Nettle/libgcrypt paths
and hard-coded priority strings. Moving TDS TLS to system OpenSSL allows
FreeTDS to use Azure Linux centralized crypto policy, FIPS behavior, and
future platform PQC support.

## What changed

- replace gnutls-devel and libgcrypt-devel with openssl-devel
- select --without-gnutls --with-openssl
- default OpenSSL cipher selection to PROFILE=SYSTEM
- preserve the administrator openssl ciphers override

## Risk

This changes both the linked TLS implementation and the default cipher policy.
Legacy SQL Server versions may stop connecting. Licensing, TDS version
compatibility, certificate validation, Kerberos, ODBC, DB-Library, and
CT-Library are all release gates.

## Validation

Include legal approval, architecture builds, exact linkage, ABI results, the
SQL Server/TDS/interface matrix, certificate failures, provider/FIPS evidence,
and negotiated algorithms.

## Rollback

Restore gnutls-devel, libgcrypt-devel, and --with-gnutls.
```
